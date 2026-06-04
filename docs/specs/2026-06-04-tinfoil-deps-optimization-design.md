# parse_bitbake_deps.py Tinfoil 优化设计文档

> **修订说明（2026-06-04 实施后更新）：** 原设计假设 Tinfoil 可以直接获取 ~569 个 build target recipes，但实测 `tinfoil.all_recipes()` 返回 ~4492 个（包含所有 layer recipes、native/nativesdk 变体）。为解决此问题，调整为保留 `bitbake -g` 生成 `pn-buildlist`，Tinfoil 仅查询该文件中的 ~569 个 recipes。总耗时从 ~17 min 降至 ~3.5 min（5x 提速）。

## 背景与目标

**为什么要做：** `ob init` 的 Step 5（生成依赖图）是整个流程的最大瓶颈，当前耗时 11-33 分钟。根因是 `parse_bitbake_deps.py` 使用 N+1 模型：先跑 `bitbake -g` 生成 `pn-buildlist`（~3 min），再对其中 569 个 recipe 逐个调用 `bitbake -e`（~14 min），其中 387 次（69%）查到的没有 git 源白跑。

**要解决什么：** 将 Step 5 中 `bitbake -e × N` 的 N+1 子进程查询替换为 Tinfoil 单进程查询，将总耗时从 ~17 min 降至 ~3.5 min，同时保持输出 `deps.json` 的格式和内容完全兼容。

**成功标准：**
- Step 5 耗时 < 5 min（实测 ~3.5 min：bitbake -g ~3min + Tinfoil ~37s）
- `deps.json` 输出格式和字段与现有完全一致
- 保留 `bitbake -g` 生成 `pn-buildlist`（Tinfoil 无法直接获取 build target 依赖列表）
- `ob` 脚本中 `ob init` 流程无需结构性改动

## 范围

- 重写 `tools/parse_bitbake_deps.py` 的核心查询逻辑：用 bitbake Tinfoil API 替代 `bitbake -e × N`
- 保留 `ob` 脚本中 `generate_dep_graph()` 的 `bitbake -g` 调用（用于生成 `pn-buildlist`）
- Tinfoil 模式下从 `pn-buildlist` 读取 recipe 列表，单进程批量查询
- 将 `--skip-deps` 从 `--help` 中移除（降级为隐性应急选项）

## 非范围

- 不修改 `deps.json` 的输出字段或格式
- 不修改下游 `clone_sub_repos()`、`generate_lockfile()`、`inject_externalsrc()` 的逻辑
- 不实现缓存机制（按 commit hash 缓存 deps.json）
- 不修改 `parse_bitbake_deps.py` 的命令行接口（`--pn-buildlist` 和 `--machine` 参数保留以兼容，但 `--pn-buildlist` 变为可选）
- 不做预筛优化（方案 A），因为 Tinfoil 已足够快，预筛的复杂度不值得引入
- 不移除 `bitbake -g` 步骤（Tinfoil `all_recipes()` 返回 ~4492 个 recipes 而非 ~569 个 build target deps，直接使用会慢 8 倍）

## 方案比较

### 方案 A：预筛 + bitbake -e 仅查 git recipe

- **核心思路：** 先 grep 所有 .bb 文件包含 `git://` 的 recipe 名（~0.5s），然后只对这些 recipe 调用 `bitbake -e`
- **优点：** 改动最小（~30 行），不引入新依赖
- **缺点：** `bitbake -g` 仍需 ~3 min；.bb 文件名到 recipe name 的映射不精确；仍需 569→210 次 `bitbake -e`，总耗时 ~8 min，改善有限

### 方案 B：Tinfoil 单进程批量查询（采用）

- **核心思路：** 使用 bitbake 内置的 `bb.tinfoil.Tinfoil` Python API，在单个 Python 进程内完成 SRC_URI/SRCREV 查询。保留 `bitbake -g` 生成 `pn-buildlist`，Tinfoil 仅查询其中 ~569 个 recipes。
- **优点：** 消除 N 次进程启动开销；一次 Tinfoil 初始化（~7s）后每个 recipe 查询仅 ~0.06s；总耗时从 ~17 min 降至 ~3.5 min（5x 提升）；走 bitbake 内部解析，数据准确性等同 `bitbake -e`
- **缺点：** 依赖 bitbake 内部 API（Tinfoil），跨 bitbake 大版本可能有接口变化；需要在正确的 bitbake 环境（`source setup` 后）中运行；Tinfoil.prepare() 会向 stdout 写入日志，需要临时重定向 stdout→stderr

### 方案 C（原设计，已放弃）：Tinfoil all_recipes() 完全替代 bitbake -g

- **核心思路：** 用 `tinfoil.all_recipes()` 完全替代 `bitbake -g`，不依赖 `pn-buildlist`
- **放弃原因：** `tinfoil.all_recipes()` 返回 ~4492 个 recipes（所有 layer recipes，含 native/nativesdk/cross 变体），过滤后仍有 ~2751 个 target recipes，查询耗时 ~4.5 min。而 `pn-buildlist`（来自 `bitbake -g`）只包含 build target 的直接和递归依赖 ~569 recipes，查询仅需 ~37s。代价差 8 倍，不值得为省 3 min 的 `bitbake -g` 而牺牲 4 min 的查询效率。

### 方案 D：缓存 + 增量

- **核心思路：** 按 openbmc 主仓库 commit hash 缓存 `deps.json`，同 commit 直接复用
- **优点：** 重跑 0 耗时
- **缺点：** 用户担心信息过期；首次 init 不受益；缓存失效管理增加复杂度

## 采用方案

**方案 B（Tinfoil + pn-buildlist）**

实测 benchmark 数据（<MACHINE>, 2026-06-04）：

| 方案 | 耗时 | 加速比 |
|---|---|---|
| 现状（bitbake -g ~3min + bitbake -e × 569 ~14min） | ~17 min | 1x |
| 方案 A（预筛 + bitbake -e × 210） | ~8 min | 2x |
| 方案 C（Tinfoil all_recipes ~2751 recipes） | ~4.5 min | 3.8x |
| **方案 B（bitbake -g ~3min + Tinfoil × 569 ~37s）** | **~3.5 min** | **5x** |

*注：原设计文档中方案 B 标注的 ~44s/23x 是基于 Tinfoil 可以直接获取 ~569 recipes 的假设。实测发现 `all_recipes()` 返回 ~4492 个，采用 pn-buildlist 限定范围后实际 ~37s。*

采用原因：
1. **5x 加速**是显著改善，从 ~17 min 降到 ~3.5 min
2. `bitbake -e × 569` 的 ~14 min 瓶颈被完全消除（~37s 替代），这是最大的相对改善（23x）
3. **数据更准确**：Tinfoil 走 bitbake 内部解析路径，和 `bitbake -e` 结果一致（实测 182 repos 完全匹配）
4. **架构务实**：保留 `bitbake -g` 避免了 `all_recipes()` 返回过多 recipes 的问题
5. **保留回退**：`--pn-buildlist` 仍可用于 legacy `bitbake -e` 模式

主要 trade-off：
- 对 bitbake Tinfoil API 的依赖。但 OpenBMC 当前使用的 bitbake 2.x 版本中 Tinfoil API 稳定（`prepare()` + `parse_recipe()` + `getVar()` 是 bitbake 的核心公共接口，`devtool` 等官方工具重度依赖它）
- `Tinfoil.prepare()` 会向 stdout 写入缓存加载日志，需要临时将 `sys.stdout` 重定向到 `sys.stderr`，确保输出到 stdout 的 JSON 不被污染

## 关键边界与组件职责

### `tools/parse_bitbake_deps.py`

职责变更：
- **旧：** 接收 `--pn-buildlist` 文件路径，逐行读取 recipe name，对每个 recipe 启动 `bitbake -e` 子进程解析 SRC_URI/SRCREV
- **新：** 
  - **默认 Tinfoil 模式**（`--build-dir`）：使用 Tinfoil API 在进程内完成查询。如果 `build_dir/pn-buildlist` 存在，只查询其中 ~569 个 recipes；否则降级为 `all_recipes()` + native 过滤（~2750 个，较慢）
  - **保留 --pn-buildlist 参数**用于指定 pn-buildlist 路径时仍走旧 `bitbake -e` 路径（legacy 回退）

接口兼容：
- `--pn-buildlist` 参数降级为可选（保留向后兼容，传入时仍使用旧逻辑）
- `--machine` 参数保留
- 新增 `--build-dir` 参数：指定 build 目录，Tinfoil 从中读取 `conf/bblayers.conf` 并自动查找 `pn-buildlist`
- stdout 输出 JSON 格式和字段不变：`name, src_uri, srcrev, recipe, clone_url, branch`

Tinfoil stdout 污染修复：
- `Tinfoil.prepare()` 会向 stdout 写入 "Loading cache...", "Parsing recipes..." 等日志
- 在 Tinfoil 上下文期间临时 `sys.stdout = sys.stderr`，完成后恢复 `sys.stdout`
- 确保 `json.dump(results, sys.stdout)` 输出纯净 JSON

### `ob` 脚本 `generate_dep_graph()`

职责变更：
- **保留 `bitbake -g obmc-phosphor-image`**（生成 pn-buildlist，~3 min）
- 将 `parse_bitbake_deps.py --pn-buildlist` 替换为 `parse_bitbake_deps.py --build-dir`（Tinfoil 查询，~37s）

### `ob` 脚本 `main()`

变更：
- `generate_dep_graph()` 中保留 `bitbake -g` 调用，改用 `--build-dir` 参数替代 `--pn-buildlist`
- `--skip-deps` 保留在代码中但从 `--help` 移除（降级为隐性应急选项）

## 数据流 / 控制流
### 旧流程（bitbake -g + bitbake -e × 569）

```raw
ob main()
  └─ generate_dep_graph()
       ├─ cd $OPENBMC_DIR && source setup $MACHINE $BUILD_DIR
       ├─ bitbake -g obmc-phosphor-image    (~3 min)
       │    └─ 生成 pn-buildlist (569 recipes)
       └─ parse_bitbake_deps.py --pn-buildlist --machine
            └─ for recipe in pn-buildlist:    (× 569, ~14 min)
                 ├─ subprocess: bitbake -e <recipe>  (~1-3s each)
                 └─ parse SRC_URI, SRCREV from stdout
            └─ output: deps.json
```
### 新流程（bitbake -g + Tinfoil × 569）
```raw
ob main()
  └─ generate_dep_graph()
       ├─ cd $OPENBMC_DIR && source setup $MACHINE $BUILD_DIR
       ├─ bitbake -g obmc-phosphor-image    (~3 min, 保留)
       │    └─ 生成 pn-buildlist (569 recipes)
       └─ parse_bitbake_deps.py --build-dir $BUILD_DIR --machine $MACHINE
            ├─ build_dir/pn-buildlist 存在 → 读取 569 recipe names
            ├─ sys.stdout = sys.stderr  (防 Tinfoil 日志污染)
            ├─ Tinfoil().prepare()              (~7s, 一次)
            ├─ for recipe in pn-buildlist:      (× 569, ~30s)
            │    ├─ d = tinfoil.parse_recipe(recipe)  (~0.06s each)
            │    └─ extract SRC_URI, SRCREV from d.getVar()
            ├─ sys.stdout 恢复
            └─ output: deps.json → stdout
```
**关键变化：** `bitbake -g` 步骤保留（用于生成精确的 ~569 recipe 列表）；`bitbake -e × 569` 子进程查询被 Tinfoil 单进程查询替代（~14 min → ~37s）。整体从 ~17 min 降到 ~3.5 min。

**Tinfoil 技术要点：**
1. `pn-buildlist` 为 Tinfoil 提供精确的 recipe 范围，避免查询 ~4492 个无关 recipes
2. `sys.stdout` 临时重定向到 `sys.stderr` 防止 `Tinfoil.prepare()` 的日志污染 JSON 输出
3. Tinfoil 上下文结束前恢复 stdout，确保 `json.dump()` 输出纯净
```bash
# 在已有 build 环境中对比新旧输出
cd workspace/openbmc/build/<MACHINE>
# 确保已 source setup
python3 ../../../tools/parse_bitbake_deps.py --build-dir . --machine <MACHINE> > /tmp/deps_new.json
# 对比 repo 数量和名字集合
python3 -c "
import json, sys
old = sorted(json.load(open('deps.json')), key=lambda x: x['name'])
new = sorted(json.load(open('/tmp/deps_new.json')), key=lambda x: x['name'])
print(f'Old: {len(old)} repos, New: {len(new)} repos')
missing = set(d['name'] for d in old) - set(d['name'] for d in new)
extra = set(d['name'] for d in new) - set(d['name'] for d in old)
if missing: print(f'Missing in new: {sorted(missing)}')
if extra: print(f'Extra in new: {sorted(extra)}')
if not missing and not extra: print('Name sets match!')
"
```

## 未决事项

无。
