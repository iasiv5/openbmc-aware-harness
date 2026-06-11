# start-qemu QEMU binary URL 配置化设计

> **状态**: 草稿 / 待审批 (2026-06-10)
> **动机**: `ob start-qemu` 在 custom 源下因 `OB_QEMU_BINARY_URL` 与 `meta-*/qemu-binary-url.sh` 均缺失而报错退出，报错不可执行；且 binary 名硬编码为 `qemu-system-arm`，无法支持 AST2700（aarch64）。需要把 QEMU binary 下载地址持久化到配置文件，按 SOC 架构区分，并在 custom 缺失时交互式补齐。

## 目标

1. QEMU binary 下载地址落盘到 `workspace/qemu-bin/` 下的配置文件，按 `source_label` × 架构（`QB_SYSTEM_NAME`）索引，跨运行复用。
2. 架构（`qemu-system-arm` / `qemu-system-aarch64`）由 `bitbake -e` 的 `QB_SYSTEM_NAME` 自动解析，用户与脚本都不手动选架构。
3. community 源对已知架构（arm）自动填 jenkins 默认；custom 源在配置缺失时弹菜单让用户输入并写回。
4. 修复 `SOURCE_LABEL` 生产侧失效，使 community/custom 判定可靠。
5. 非交互场景（CLI / 非 tty / CI）不阻塞，改用环境变量覆盖或打印可执行报错。

## 非目标

1. 不引入 per-machine 的 community/custom 属性；`source_label` 仍是 workspace 全局（绑定 OpenBMC 源）。
2. 不改 QEMU 启动命令行、端口转发、PID 管理、等待逻辑。
3. 不为 community 源预置 aarch64 默认地址（社区无此 binary）。
4. 不处理 QEMU binary 的版本更新/校验策略（沿用现有 `.manifest`）。

## 现状与问题（观察）

1. **`SOURCE_LABEL` 生产侧已废**：`SOURCE_LABEL` 在 [ob](ob#L18) 初始化为空，另一处赋值 [ob](ob#L942) 也是空串，`write_source_lock` 把空值写入 lock（[ob](ob#L843)）。无任何代码推导成 `community`/`custom`。`read_source_label` 读到空 → fallback `community`（[ob](ob#L417-L427)）。当前 [workspace/configs/openbmc-source.lock](workspace/configs/openbmc-source.lock#L4) 里的 `custom` 是旧版 ob 残留。
2. **binary 名硬编码 arm**：`derive_qemu_paths` 把目标固定为 `qemu-system-arm`（[ob](ob#L430-L436)），AST2700 需要的 `qemu-system-aarch64` 无法落地。
3. **custom URL 发现机制脆弱**：`ensure_qemu_binary` 的 custom 分支只查 `OB_QEMU_BINARY_URL` 和顶层 glob `meta-*/qemu-binary-url.sh`（[ob](ob#L490-L510)）。后者 `meta-*` 只匹配单层，扫不到深层嵌套 layer（b865g8 的 machine layer 在 `meta-inventec/meta-platform-customer/meta-b865g8-bytedance/`）。缺失时直接 `exit 1`，报错不告诉用户放哪个文件、写什么。
4. **下载吞错**：custom 分支 `curl -fSL ... 2>/dev/null`（[ob](ob#L518)）吞掉 404/DNS/证书等真实失败原因；`mktemp /tmp/qemu-binary-XXXXXX`（[ob](ob#L515)）硬编码 `/tmp`，不尊重 `$TMPDIR`。
5. **调用顺序错位**：`cmd_start_qemu` 先 `ensure_qemu_binary`（[ob](ob#L2491)）再 `resolve_qb_vars`（[ob](ob#L2597)）。新方案需先知道 `QB_SYSTEM_NAME` 才能决定下载哪个 binary。

## 关键事实（已验证）

| machine | SOC | `QB_MACHINE` | `QB_SYSTEM_NAME` |
|---|---|---|---|
| `b865g8-bytedance` | AST2700 | `-machine ast2700a1-evb` | `qemu-system-aarch64` |
| `romulus` | AST2600 | `-machine romulus-bmc` | `qemu-system-arm` |

`QB_SYSTEM_NAME` 来自 OpenBMC 标准的 qemuboot.conf / `bitbake -e`，权威标识该用哪个 QEMU binary。b865g8 是 AST2700 见 [workspace/openbmc/meta-inventec/meta-platform/meta-b865g8/conf/machine/b865g8.conf](workspace/openbmc/meta-inventec/meta-platform/meta-b865g8/conf/machine/b865g8.conf#L1)。

## 设计决策

### 1. 架构自动解析（扩展 `resolve_qb_vars`）

在 `resolve_qb_vars` 内，沿用现有 `bitbake -e` 一次展开，额外解析 `QB_SYSTEM_NAME` 到新全局变量 `QB_SYSTEM_NAME`（取值 `qemu-system-arm` / `qemu-system-aarch64`）。与 [ADR 0002](docs/adr/0002-qb-variables-via-bitbake-e.md)「QB 变量经 bitbake -e 解析，不提供 fallback」一致：解析不到则报错退出，不猜默认。

### 2. 配置文件

单文件，落在用户指定的 `workspace/qemu-bin/` 下：

```raw
# workspace/qemu-bin/qemu-binary-urls.conf — auto-managed by 'ob start-qemu'
# key 格式: <source_label>.<QB_SYSTEM_NAME>
community.qemu-system-arm=https://jenkins.openbmc.org/job/latest-qemu-x86/lastSuccessfulBuild/artifact/qemu/build/qemu-system-arm
custom.qemu-system-arm=<custom arm 地址，首次运行时输入>
custom.qemu-system-aarch64=<custom aarch64 地址，首次运行时输入>
```

- key 为 `<source_label>.<QB_SYSTEM_NAME>`，同时承载 source 与架构两个维度，单文件即可表达全部组合。
- 新增两个辅助函数：
  - `read_qemu_url_config <source> <arch>`：读取对应 key 的 URL，无则返回空。
  - `write_qemu_url_config <source> <arch> <url>`：upsert 对应 key（存在则替换，不存在则追加），写入前确保目录存在。

### 3. binary 缓存落点

binary 落点从硬编码 `qemu-bin/<source>/qemu-system-arm` 改为 `qemu-bin/<source>/<QB_SYSTEM_NAME>`，使 arm 与 aarch64 在同一 source 下共存。`derive_qemu_paths` 改为在 `QB_SYSTEM_NAME` 已知后派生 `QEMU_BIN_FILE`。

### 4. `SOURCE_LABEL` 推导（修复 init 侧）

新增 `derive_source_label`，在 `write_source_lock` 内部（写入前）调用，覆盖两个写入点（[ob](ob#L943) 的 `verify_source` 和 [ob](ob#L1932) 的 `clone_openbmc`）：

- 规则：`normalize_repo_url "$OPENBMC_REPO_URL"` 结果**精确等于** `github.com/openbmc/openbmc` → `community`，否则 `custom`。该值是 `DEFAULT_OPENBMC_REPO_URL`（`https://github.com/openbmc/openbmc.git`，[ob](ob#L16)）经 `normalize_repo_url` 归一化（去 scheme、去 `.git`、小写）后的完整 `host/path`，**不是** `github.com/openbmc`。
- **不兼容** `git.openbmc.org` 等社区镜像 host：`ob init` 选择社区主仓时下载地址固定，精确匹配即可，无需扩大匹配面引入歧义。
- 结果写入 `openbmc-source.lock` 的 `source_label`，供 `read_source_label` 消费。

### 5. start-qemu 解析与下载流程

调整 `cmd_start_qemu` 调用顺序：**先 `resolve_qb_vars`（得 `QB_SYSTEM_NAME`）→ `derive_qemu_paths` → `ensure_qemu_binary`**。`ensure_qemu_binary` 重构为按下列优先级确定 URL：

1. `OB_QEMU_BINARY_URL`（环境变量，最高优先级，用于当前解析出的架构；非交互/CI 覆盖通道）。设置时写回对应 `<source>.<arch>` key。
2. 配置文件 `read_qemu_url_config <source> <arch>` 命中 → 用。
3. 未命中：
   - `community` + `qemu-system-arm` → 用内置 jenkins 默认，**写回配置文件**（让默认值在配置文件中可见，与 custom 行为一致；文件头注释标明 auto-managed，说明无需手工维护）。
   - `custom`（任意架构）→ 若 tty，弹菜单提示输入 URL，校验 scheme（`http(s)://`）后写回；若非 tty，报错并提示设 `OB_QEMU_BINARY_URL` 或编辑配置文件。
   - `community` + `qemu-system-aarch64`（社区无此 binary）→ 走与 custom 相同的交互输入路径，但**先打印一行告知**「社区源无 aarch64 QEMU binary，请提供自定义地址，或直接回车/Ctrl-C 退出」，用户可选择输入或退出。

URL 确定后，沿用现有下载、文件类型识别（tarball/单 binary）、`chmod +x`、`.manifest` 写入逻辑。

### 6. 非交互兼容

- 所有交互输入前先判 `[[ -t 0 ]]`；非 tty 时不 `read`，改打印可执行报错（指明配置文件路径、key 名、`OB_QEMU_BINARY_URL` 用法）。
- 交互输入时允许空输入（直接回车）视为放弃，干净退出（非错误码或明确提示），对应「告知后用户可选择退出」。
- 保留 `OB_QEMU_BINARY_URL` 向后兼容。

### 7. 附带修复

- custom 下载去掉 `2>/dev/null`（或仅保留 stderr），暴露 curl 真实失败原因。
- `mktemp` 改用 `mktemp "${TMPDIR:-/tmp}/qemu-binary-XXXXXX"`，尊重 `$TMPDIR`。

## 影响范围

| 函数 / 位置 | 改动 |
|---|---|
| 全局变量区 | 新增 `QB_SYSTEM_NAME`、`QEMU_URL_CONFIG_FILE` |
| `derive_source_label`（新增） | 由 origin 归一化推导 community/custom |
| `write_source_lock`（[ob](ob#L821)） | 写入前调用 `derive_source_label` |
| `read_qemu_url_config` / `write_qemu_url_config`（新增） | 读写 `qemu-binary-urls.conf` |
| `resolve_qb_vars`（[ob](ob#L575)） | 增解析 `QB_SYSTEM_NAME` |
| `derive_qemu_paths`（[ob](ob#L429)） | binary 名改用 `QB_SYSTEM_NAME` |
| `ensure_qemu_binary`（[ob](ob#L438)） | 重构 URL 解析优先级 + 交互输入 + 非 tty 处理 + 附带修复 |
| `cmd_start_qemu`（[ob](ob#L2432)） | 调整调用顺序：`resolve_qb_vars` 提前到 `ensure_qemu_binary` 之前 |

## 边界与已知后果

1. **当前 workspace 全局 custom**：OpenBMC 源是企业源（`172.17.8.200/...`），推导为 `custom`。修复后 **b865g8 和 romulus 都走 custom 弹菜单输入**，romulus 也不例外。community 自动填分支只有用 `github.com/openbmc` 源 init 的 harness 副本才触发。
2. **community + aarch64**：社区无 aarch64 binary，告知后走交互输入，用户可选择退出，不预置默认。
3. **跨架构 env 覆盖**：`OB_QEMU_BINARY_URL` 是单值，仅作用于当前解析出的架构；同 workspace 跑两种架构时应改用配置文件而非 env。

## 验证方式

1. **架构解析**：`ob start-qemu b865g8-bytedance` 在 verbose 下打印 `QB_SYSTEM_NAME=qemu-system-aarch64`；`romulus` 打印 `qemu-system-arm`。
2. **custom 交互写回**：custom 源下首次对 b865g8 运行，弹菜单输入 URL 后，`qemu-binary-urls.conf` 出现 `custom.qemu-system-aarch64=<输入值>`；二次运行不再弹菜单。
3. **community arm 写回**：community 源下对 arm machine 运行，配置缺失时自动填 jenkins 默认并写回；`qemu-binary-urls.conf` 出现 `community.qemu-system-arm=<jenkins URL>`。
4. **community aarch64 告知退出**：community 源下对 aarch64 machine 运行，打印告知行后，空输入可干净退出。
5. **source_label 推导**：以 `github.com/openbmc/openbmc.git` 重新 init，`openbmc-source.lock` 的 `source_label=community`；企业源则为 `custom`。
6. **非交互**：`echo | ob start-qemu b865g8-bytedance`（非 tty）在配置缺失时报可执行错误，不阻塞。
7. **附带修复**：故意配错误 URL，custom 下载失败时 stderr 可见 curl 原因。
