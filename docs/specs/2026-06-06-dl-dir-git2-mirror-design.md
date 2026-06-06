# DL_DIR/git2/ Bare Mirror 缓存机制 设计文档

## 背景与目标

`ob init` 用 `externalsrc` 机制绕过 BitBake 的 git fetcher，将 git 源码直接克隆到 `workspace/src/<machine>/`，再通过 `EXTERNALSRC_pn-<recipe>` 指向本地路径。这导致 `workspace/downloads/`（`DL_DIR`）对 git 源码始终为空——只有 tarball 等非 git 资源才会在 `bitbake` 编译时进入该目录。

问题：

1. **概念不一致**：BitBake 用户预期 `DL_DIR` 是源码缓存的核心位置，但它对 git 源码为空，实际源码在 `src/<machine>/`，造成困惑。
2. **跨 machine 无去重**：`ob init romulus` 和 `ob init witherspoon` 各自克隆独立的 working tree，大量共享 repo（bmcweb、phosphor-\* 等）重复下载。
3. **缺少标准回退路径**：如果去掉 externalsrc 做干净构建，BitBake 必须从头下载全部 git 源码，因为 `DL_DIR/git2/` 里没有任何缓存。

**目标**：在 `ob init` 克隆 git 子仓库时，同步填充 `DL_DIR/git2/` 中的 bare mirror，使 harness 的下载行为与 BitBake 原生缓存机制对齐。

**成功标准**：

- `ob init romulus` 执行后，`downloads/git2/` 中包含与 deps.json 对应的 bare mirror
- `ob init witherspoon`（第二个 machine）能复用已有 bare mirror，不再从远程重新下载共享 repo
- `downloads/` 目录不再对 git 源码为空，概念一致
- 重跑 `ob init` 幂等安全，已有 mirror 和 working tree 不被破坏

## 范围

- 改造 `ob` 脚本的 `clone_sub_repos()` 函数，加入 bare mirror 创建步骤
- 移除 `OB_GIT_REFERENCE_DIR` 变量及其相关代码
- 新增"有效 DL_DIR"计算逻辑（读 local.conf → fallback 到 harness 默认值）
- 处理 DL_DIR 不可写时的 fallback 路径

## 非范围

- 不改变 `externalsrc` 机制本身（`EXTERNALSRC_pn-*` 的生成逻辑不变）
- 不改变非 git 源码（tarball 等）的下载流程
- 不改变 `deps.json` 的生成逻辑
- 不实现 mirror 的清理/垃圾回收机制
- 不实现跨 harness 实例的 mirror 共享（仅限同一 harness 内的跨 machine 复用）

## 方案比较

### 方案 A：Bare mirror 先行（已选定）

先创建 bare mirror 到 `DL_DIR/git2/`（全量，所有分支），再从 bare mirror 本地克隆 working tree 到 `src/<machine>/`。

**优点**：

- 与 BitBake 原生 git fetcher 的行为模式一致（先 bare mirror，后 working tree）
- 跨 machine 复用自然实现——第二个 machine 发现已有 bare mirror 后直接用它
- bare mirror 是"源"，working tree 是"派生"，主从关系清晰

**缺点**：

- 对 `clone_sub_repos()` 改动较大（核心流程重构）
- 首次 init 时每个 repo 多一次本地 clone 操作（从 bare mirror 到 working tree），但因为是纯本地操作，耗时可以忽略

### 方案 B：Working tree 先行 + 追加 bare mirror

保持现有远程 clone 流程不变，克隆完成后额外执行 `git clone --bare` 从 working tree 创建 bare mirror。

**优点**：改动最小，在现有流程后追加一步。

**缺点**：

- bare mirror 可能缺少分支（working tree 可能是 `--single-branch`）
- 主从关系倒置（working tree 是"源"，bare mirror 是"派生"），与 BitBake 原生行为相反
- 未来 machine 复用 bare mirror 时可能因缺分支而需要重新从远程拉取

## 推荐方案

**方案 A：Bare mirror 先行。**

选择理由：

1. 全量 bare mirror 为跨 machine 复用提供可靠基础（所有分支/标签都在）
2. 与 BitBake 原生行为对齐，减少用户的认知负担
3. 从 bare mirror 克隆 working tree 是纯本地操作，不增加网络开销

主要 trade-off：`clone_sub_repos()` 的改动量比方案 B 大，但这是一次性投入。

## 关键边界与组件职责

### 1. 有效 DL_DIR 计算函数 `resolve_effective_dl_dir()`

- 输入：`local.conf` 路径、harness 默认值（`workspace/downloads/`）
- 逻辑：读取 `local.conf` 中的 `DL_DIR` 赋值（跳过注释行）；未找到则使用 harness 默认值
- 输出：绝对路径字符串
- 可写检查：对输出路径尝试创建临时文件，失败则 fallback 到 `workspace/downloads/`

### 2. Mirror 路径计算 `derive_bitbake_git_mirror_path()`（已有，保留）

- 输入：有效 DL_DIR 绝对路径、repo 的 `SRC_URI`
- 逻辑：沿用现有 BitBake 命名规则（`github.com.openbmc.bmcweb.git` 格式）
- 输出：`<DL_DIR>/git2/<bitbake_name>` 绝对路径

### 3. `clone_sub_repos()` 改造

核心流程变更——对每个 repo：

```
Mirror 阶段:
  mirror_path = derive_bitbake_git_mirror_path(effective_dl_dir, src_uri)
  mirror_path 存在且是 bare repo → git fetch --all
  mirror_path 不存在             → git clone --bare <remote> <mirror_path>

Working tree 阶段:
  working_path 存在且 .git 存在  → git fetch --all
  working_path 不存在            → git clone <mirror_path> <working_path>

Checkout 阶段:
  checkout srcrev（现有逻辑不变）
```

### 4. 移除的组件

| 组件 | 处置 |
|------|------|
| `resolve_git_reference_root()` | 移除 |
| `ensure_bootstrap_local_conf()` 中 `OB_GIT_REFERENCE_DIR` 相关逻辑 | 移除 |
| `read_local_conf_var()` | 保留（其他地方仍使用） |
| `OB_GIT_REFERENCE_DIR` 在 local.conf 中的写入 | 不再写入 |

## 数据流 / 控制流

```
ob init <machine>
  │
  ├─ Step 1-4: （不变）
  │
  ├─ Step 5: clone_sub_repos()
  │   │
  │   ├─ 调用 resolve_effective_dl_dir()
  │   │   ├─ 读 local.conf 中的 DL_DIR
  │   │   ├─ 未找到 → 用 harness 默认值 workspace/downloads/
  │   │   └─ 检查可写 → 不可写 → fallback 到 workspace/downloads/
  │   │
  │   ├─ 对 deps.json 中每个 repo:
  │   │   │
  │   │   ├─ 计算 mirror_path = derive_bitbake_git_mirror_path(effective_dl_dir, src_uri)
  │   │   │
  │   │   ├─ Mirror 存在？
  │   │   │   ├─ Yes → git -C <mirror_path> fetch --all
  │   │   │   └─ No  → git clone --bare <clone_url> <mirror_path>
  │   │   │             失败 → 记录失败，跳到下一个 repo
  │   │   │
  │   │   ├─ Working tree 存在？
  │   │   │   ├─ Yes → git -C <working_path> fetch --all
  │   │   │   └─ No  → git clone <mirror_path> <working_path>
  │   │   │             失败 → 回退到直接从远程 clone（兼容旧行为）
  │   │   │
  │   │   ├─ Checkout srcrev（现有逻辑不变）
  │   │   │
  │   │   └─ 下一个 repo
  │   │
  │   └─ 输出统计信息
  │
  ├─ Step 6-7: （不变）
  │
  └─ Step 8: print_report()
      └─ 新增：报告 mirror 数量和路径
```

## 错误处理与回退

| 失败模式 | 处理策略 |
|----------|----------|
| 有效 DL_DIR 不可写 | Fallback 到 `workspace/downloads/`，并在 report 中提示 |
| Bare mirror clone 失败（网络问题） | 跳过 working tree clone，记录 `STATUS_FAILED`，继续下一个 repo |
| 从 mirror 克隆 working tree 失败（mirror 损坏） | 回退到直接从远程 clone working tree（等同当前行为），不创建 mirror |
| Mirror 存在但 fetch 失败 | Warn 并继续，使用 mirror 的现有状态 |
| Working tree 存在但没有对应 mirror（旧版本遗留） | 从远程创建全量 bare mirror：`git clone --bare <clone_url> <mirror_path>`（不从 working tree 创建，因为 working tree 可能是单分支的） |
| `derive_bitbake_git_mirror_path()` 无法解析 SRC_URI | 跳过 mirror 创建，直接从远程 clone working tree |

## 测试策略

### 核心验证行为

1. **首次 init**：`ob init romulus` 后，`downloads/git2/` 包含与 deps.json 对应的 bare mirror，且 `src/romulus/` 包含 working tree
2. **跨 machine 复用**：先 `ob init romulus`，再 `ob init witherspoon`；共享 repo 的 bare mirror 不重复下载
3. **幂等重跑**：`ob init romulus` 执行两次，第二次应 fetch 已有 mirror 和 working tree，不重新 clone
4. **DL_DIR 不可写**：当有效 DL_DIR 不可写时，mirror 写入 `workspace/downloads/` 且不报错
5. **旧数据兼容**：已有 working tree 但没有 mirror 时，补建 mirror
6. **bitbake 编译不受影响**：`bitbake obmc-phosphor-image` 行为不变（externalsrc 仍然生效）

### 测试方式

- 手动测试：在干净 workspace 上执行完整 `ob init` 流程，验证 `downloads/git2/` 产出
- 增量测试：重跑 `ob init`，验证 fetch 行为
- 跨 machine 测试：连续 init 两个 machine，对比网络流量和 clone 日志

## 未决事项

无。所有设计决策已在 `/grill-with-docs` 讨论中确认。
