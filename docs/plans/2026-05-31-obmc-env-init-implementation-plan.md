# OpenBMC 开发环境一键初始化命令 —— 实施计划

> 设计文档：`docs/specs/2026-05-31-obmc-env-init-design.md`
> 日期：2026-05-31

## 目标

实现 `ob init <machine>` 一键初始化命令，完成从零到可本地开发环境的完整搭建。

## 架构快照

```
openbmc-aware-harness/
  .gitignore                                # 新建：忽略 workspace/ 内容
  workspace/.gitkeep                        # 新建：占位
  tools/
    ob                                      # 新建：主入口 bash 脚本
    parse_bitbake_deps.py                   # 新建：bitbake 依赖图解析（Python）
  rules/skills/workflow_obmc_env_init.md    # 新建：AI skill 触发层
```

已有文件不改动。所有新增文件职责独立，无互相耦合。

## 文件结构与职责

| 文件 | 职责 | 与设计文档对应 |
|---|---|---|
| `.gitignore` | 忽略 `workspace/` 下除 `.gitkeep` 外的所有内容 | 决策 #12 |
| `workspace/.gitkeep` | 确保 workspace/ 目录可追踪 | 决策 #12 |
| `tools/ob` | CLI 主入口：参数解析、前置检查、调用各步骤函数 | Step 1-8 |
| `tools/parse_bitbake_deps.py` | 解析 bitbake 依赖图输出，提取 SRC_URI + SRCREV | Step 4 |
| `rules/skills/workflow_obmc_env_init.md` | AI skill：触发词、执行方式、排障指南 | AI Skill 层 |

## 任务清单

### Task 1: 创建 .gitignore

**动作**：在仓库根目录创建 `.gitignore`，忽略 `workspace/` 下除 `.gitkeep` 的所有内容。

**涉及文件**：`.gitignore`（新建）

**文件内容**：
```
# OpenBMC workspace: source trees, builds, lockfiles
workspace/*
!workspace/.gitkeep
```

**验证**：
```powershell
# 确认 .gitignore 存在且内容正确
Get-Content .gitignore
# 预期：包含 workspace/* 和 !workspace/.gitkeep 两行
```

---

### Task 2: 创建 workspace/.gitkeep

**动作**：在 `workspace/` 目录下创建空的 `.gitkeep` 文件。

**涉及文件**：`workspace/.gitkeep`（新建）

**验证**：
```powershell
Test-Path workspace/.gitkeep
# 预期：True
```

---

### Task 3: 创建 tools/parse_bitbake_deps.py

**动作**：编写 Python 脚本，解析 `bitbake -g` 生成的 pn-buildlist，逐 recipe 调用 `bitbake -e <recipe>` 提取 SRC_URI 和 SRCREV。

**涉及文件**：`tools/parse_bitbake_deps.py`（新建）

**脚本职责**：
1. 接收 `pn-buildlist` 文件路径（bitbake -g 生成的 recipe 列表）和 `machine` 名称
2. 对 pn-buildlist 中的每个 recipe，调用 `bitbake -e <recipe>` 通过 subprocess 获取该 recipe 的环境变量
3. 通过 `query_recipe_env(recipe_name)` 函数运行 subprocess 并解析输出，提取 `SRC_URI` 和 `SRCREV`
4. 过滤掉非 git 源的 SRC_URI（只保留 `git://` 开头的）
5. 从 git URL 中提取仓库名（如 `git://github.com/openbmc/linux` → `linux`）
6. 输出 JSON 数组到 stdout，每个元素包含 `name`、`src_uri`、`srcrev`、`recipe`、`clone_url`
7. 在 stderr 上显示进度："Querying recipe N/total: <name>..."

**接口定义**：
```bash
python3 tools/parse_bitbake_deps.py \
  --pn-buildlist <path>/pn-buildlist \
  --machine <machine>
```

**输出格式**（JSON 数组）：
```json
[
  {
    "name": "linux",
    "src_uri": "git://github.com/openbmc/linux;protocol=https;branch=dev-6.6",
    "srcrev": "def456...",
    "recipe": "linux-obmc",
    "clone_url": "https://github.com/openbmc/linux"
  }
]
```

**验证**：
```powershell
# 确认脚本存在且语法正确
python3 -c "import py_compile; py_compile.compile('tools/parse_bitbake_deps.py', doraise=True)"
# 预期：无报错

# 确认 --help 可用
python3 tools/parse_bitbake_deps.py --help
# 预期：输出用法说明
```

---

### Task 4: 创建 tools/ob 主入口脚本

**动作**：编写 bash 脚本，实现 `ob init <machine>` 完整流程。

**涉及文件**：`tools/ob`（新建）

**脚本结构**（函数拆分）：

```bash
#!/usr/bin/env bash
set -euo pipefail

# === 全局变量 ===
HARNESS_ROOT=""      # 本仓库根目录（自动检测）
WORKSPACE_DIR=""     # workspace/ 路径
MACHINE=""           # 目标 machine 名称
OPENBMC_DIR=""       # workspace/openbmc/
BUILD_DIR=""         # workspace/openbmc/build/<machine>/
SRC_DIR=""           # workspace/src/<machine>/
CONFIGS_DIR=""       # workspace/configs/
VERBOSE=0
DRY_RUN=0
SKIP_FETCH=0

# === 函数 ===
usage()              # 打印帮助
parse_args()         # 解析命令行参数
detect_harness_root() # 检测本仓库根目录
prerequisites_check() # Step 1: 前置检查
clone_openbmc()      # Step 2: Clone 主仓库
init_bitbake_env()   # Step 3: 初始化 bitbake 环境
generate_dep_graph() # Step 4: 生成依赖图
clone_sub_repos()    # Step 5: Clone 子仓库
generate_lockfile()  # Step 6: 生成 lockfile
inject_externalsrc() # Step 7: 注入 externalsrc 配置
print_report()       # Step 8: 输出状态报告

# === 主流程 ===
main() {
    parse_args "$@"
    detect_harness_root
    prerequisites_check
    clone_openbmc
    init_bitbake_env
    generate_dep_graph
    clone_sub_repos
    generate_lockfile
    inject_externalsrc
    print_report
}

main "$@"
```

**各步骤实现要点**：

**Step 1 - prerequisites_check()**：
- `[ "$(uname -s)" = "Linux" ]` 检查 OS
- `command -v git`、`command -v python3` 检查工具
- bitbake 在 source oe-init-build-env 后才可用，此步骤只检查 git 和 python3
- 网络检查：`curl -s -o /dev/null -w "%{http_code}" https://github.com` 返回 200 或 301

**Step 2 - clone_openbmc()**：
- 目标目录已存在：输出提示 "主仓库已存在，跳过 clone。如需更新请手动 git pull。"
- 目标目录不存在：`git clone https://github.com/openbmc/openbmc.git "$OPENBMC_DIR"`
- 支持 `--dry-run`：只输出将要执行的命令

**Step 3 - init_bitbake_env()**：
- `cd "$OPENBMC_DIR"`
- 使用 OpenBMC 官方 `source setup "$MACHINE" "$BUILD_DIR"` 初始化构建环境
- `source setup` 自动处理 TEMPLATECONF、bblayers.conf、local.conf（含 MACHINE 设置和 phosphor 兼容性检查）
- 需要 `set +u` 保护，因为 setup 内部引用未定义变量
- 无需手动设置 MACHINE（setup 已处理）
- 无需手动备份 local.conf（setup 首次创建全新文件）

**Step 4 - generate_dep_graph()**：
- `cd "$OPENBMC_DIR"`，重新 `source setup "$MACHINE" "$BUILD_DIR"` 进入构建环境
- `bitbake -g obmc-phosphor-image`（生成 pn-buildlist）
- 不再执行全局 `bitbake -e` 输出到文件（`bitbake -e` 不带 recipe 名只输出全局变量，不含 per-recipe SRC_URI/SRCREV）
- 调用 Python 脚本，由脚本内部逐 recipe 调用 `bitbake -e <recipe>`：
  ```bash
  python3 "$HARNESS_ROOT/tools/parse_bitbake_deps.py" \
    --pn-buildlist "$BUILD_DIR/pn-buildlist" \
    --machine "$MACHINE" \
    > "$BUILD_DIR/deps.json"
  ```
- 读取 deps.json，存入变量供后续步骤使用

**Step 5 - clone_sub_repos()**：
- 读取 deps.json 中每个条目
- 从 src_uri 提取 clone URL（去掉 `;protocol=...;branch=...` 参数，转成 https URL）
- 本地路径：`$SRC_DIR/<name>`
- 目录已存在：`git -C "$local_path" fetch --all`
- 目录不存在：`git clone <url> "$local_path"`
- `git -C "$local_path" checkout "$srcrev"`
- 记录成功/失败到状态数组

**Step 6 - generate_lockfile()**：
- 组装 JSON（含 machine、generated_at、openbmc_commit、target_image、sub_repos 数组）
- openbmc_commit = `git -C "$OPENBMC_DIR" rev-parse HEAD`
- 写入 `$CONFIGS_DIR/<machine>.lock`

**Step 7 - inject_externalsrc()**：
- 生成 `$BUILD_DIR/conf/externalsrc-<machine>.inc`
- 文件头标注自动生成时间和 "Do not edit" 提示
- 配置多 machine 共享的下载和构建缓存：
  - `DL_DIR = "${TOPDIR}/../../../downloads"` → 指向 `workspace/downloads/`
  - `SSTATE_DIR = "${TOPDIR}/../../../sstate-cache"` → 指向 `workspace/sstate-cache/`
- 对每个 sub_repo 生成 EXTERNALSRC_pn-<recipe> 和 EXTERNALSRC_BUILD_pn-<recipe>
- 在 local.conf 中检查是否已有 `include externalsrc-<machine>.inc`；没有则追加
- 备份旧 .inc 文件为 .bak.<timestamp>（如存在）

**Step 8 - print_report()**：
- 输出成功数、失败数、失败列表
- 输出 lockfile 路径、externalsrc 配置路径
- 输出下一步命令

**验证**：
```powershell
# 确认脚本存在且语法正确
bash -n tools/ob
# 预期：无报错

# 确认 --help 可用
bash tools/ob --help
# 预期：输出用法说明，包含 init 子命令和各选项
```

---

### Task 5: 创建 AI Skill 文件

**动作**：在 `rules/skills/` 下创建 `workflow_obmc_env_init.md`，定义 AI 触发词和执行方式。

**涉及文件**：`rules/skills/workflow_obmc_env_init.md`（新建）

**文件内容要点**：
- 触发词："初始化 openbmc 环境"、"setup openbmc"、"ob init"
- 前置条件：Linux 环境、git、python3 可用
- 执行命令：`bash tools/ob init <machine>`
- 常见问题排障：
  - bitbake -g 报错：检查 machine 名称是否有效
  - 子仓库 clone 失败：检查网络和权限
  - externalsrc 不生效：确认 local.conf 中 include 行存在

**验证**：
```powershell
Test-Path rules/skills/workflow_obmc_env_init.md
# 预期：True
```

---

### Task 6: 更新 rules/skills/INDEX.md

**动作**：在 INDEX.md 的分类索引中添加新 skill 入口。

**涉及文件**：`rules/skills/INDEX.md`（修改）

**添加内容**：在 Workflow 分类下添加：
```markdown
- [OpenBMC 开发环境初始化](./workflow_obmc_env_init.md) — 一键初始化 OpenBMC 开发环境，clone 子仓库，注入 externalsrc 配置。
```

**验证**：
```powershell
Select-String -Path rules/skills/INDEX.md -Pattern "workflow_obmc_env_init"
# 预期：匹配到一行
```

---

### Task 7: 更新 rules/WORKSPACE.md 路由

**动作**：在 WORKSPACE.md 中添加 workspace/ 目录的路由说明。

**涉及文件**：`rules/WORKSPACE.md`（修改）

**添加内容**：在"项目与代码"分类下添加：
```markdown
- OpenBMC 环境：`workspace/`（主仓库、子仓库源码、lockfile；整体 gitignore）
- 初始化工具：`tools/ob`（`ob init <machine>`）
```

**验证**：
```powershell
Select-String -Path rules/WORKSPACE.md -Pattern "tools/ob"
# 预期：匹配到一行
```

## 执行纪律

1. **环境**：所有脚本在 Linux（WSL2 / 远程）中执行和验证；Windows 侧只做文件创建和静态检查
2. **顺序**：Task 1-2 可并行；Task 3 独立；Task 4 依赖 Task 3 的接口定义；Task 5-7 可并行
3. **checkpoint commit**：
   - Task 1-2 完成后提交一次（基础结构）
   - Task 3-4 完成后提交一次（核心工具）
   - Task 5-7 完成后提交一次（skill 层和文档）
4. **每完成一个 Task，跑对应的验证命令确认通过后再继续下一个**

## 最终验证

完整流程需要在 Linux 环境执行，本次实现阶段先完成以下静态验证：

```powershell
# 1. 文件完整性
Test-Path .gitignore, workspace/.gitkeep, tools/ob, tools/parse_bitbake_deps.py, rules/skills/workflow_obmc_env_init.md
# 预期：全部 True

# 2. Python 语法
python3 -c "import py_compile; py_compile.compile('tools/parse_bitbake_deps.py', doraise=True)"
# 预期：无报错

# 3. Bash 语法
bash -n tools/ob
# 预期：无报错

# 4. 帮助信息
bash tools/ob --help
# 预期：输出用法说明

# 5. INDEX.md 已注册
Select-String -Path rules/skills/INDEX.md -Pattern "workflow_obmc_env_init"
# 预期：匹配到

# 6. WORKSPACE.md 已更新
Select-String -Path rules/WORKSPACE.md -Pattern "tools/ob"
# 预期：匹配到
```

集成验证（需 Linux + 网络）：
```bash
# 在 WSL2 或远程 Linux 中执行
bash tools/ob init romulus --dry-run
# 预期：输出完整执行计划，无实际操作

# 正式执行（需要有网络）
bash tools/ob init romulus
# 预期：完成 8 个步骤，输出状态报告
```