# OpenBMC 开发环境一键初始化命令 —— 设计文档

> 日期：2026-05-31
> 状态：待审批
> 已确认未决事项：2026-05-31（全部已决，见"已确认的设计决策"和"已决事项"章节）

## 背景与目标

OpenBMC 构建依赖上百个子仓库，每个 machine 激活的子仓库集合和版本号不同。开发者首次搭建环境时需要手动 clone 主仓库、逐一拉取子仓库、配置 bitbake 指向本地源码——过程繁琐且容易遗漏。

**目标**：提供一个一键命令 `ob init <machine>`，自动完成从零到可本地开发的完整环境初始化。

**非目标**：
- 不解决 CI/CD 构建流水线问题
- 不解决跨 machine 的源码共享或切换（每个 machine 独立目录）
- 不解决 Windows 侧的直接运行（通过 WSL2 或远程 Linux 执行）
- 不替代 repo 工具或 bitbake 自身的依赖管理

## 已确认的设计决策

| # | 决策点 | 结论 |
|---|---|---|
| 1 | 命令形态 | CLI 工具 + skill 双层；底层 bash 脚本，上层 AI skill 可调用 |
| 2 | 运行环境 | Linux（WSL2 / 远程服务器）；Windows 侧未来可选加薄 PowerShell wrapper |
| 3 | 拉取方式 | 只 clone openbmc/openbmc 主仓库，通过 bitbake 依赖图发现子仓库 |
| 4 | 依赖发现 | 方案 B：首次引导主仓库后，用 bitbake 生成精确依赖；使用 OpenBMC 官方 `source setup <machine>` 初始化环境 |
| 5 | 配置文件存放 | 本仓库 `workspace/configs/<machine>.lock`（但 workspace/ 整体 gitignore，仅保留 .gitkeep） |
| 6 | 目录结构 | 见下方"目录结构"章节 |
| 7 | 本地源映射 | bitbake `externalsrc` 类，开发调试模式 |
| 8 | 命令入口 | `ob init <machine>` 一键初始化 |
| 9 | 同步机制 | 日常 `git pull` 同步上游；需更新基线时重新生成 lockfile |
| 10 | 构建目标 | `bitbake obmc-phosphor-image` |
| 11 | externalsrc 注入方式 | 生成独立 `externalsrc-<machine>.inc` 文件，在 local.conf 中 include |
| 12 | .gitignore 策略 | workspace/ 整体忽略，只留一个空的 .gitkeep 到 upstream |
| 13 | 构建目录 | 每个 machine 独立 build 目录：`workspace/openbmc/build/<machine>` |
| 14 | 私有仓库认证 | 不做特殊处理；SSH Key 认证对用户无感，git 全局配置即可 |
| 15 | 实现语言 | bash + Python 混合；bitbake 依赖图解析用 Python 辅助脚本，其余逻辑用 bash。Step 3 委托给 OpenBMC 官方 `setup` 脚本 |
| 16 | 多 machine 资源复用 | `DL_DIR` 和 `SSTATE_DIR` 指向 `workspace/openbmc/` 下的共享目录，多 machine 复用同一份下载缓存和构建缓存 |

## 目录结构

```
openbmc-aware-harness/
  workspace/
    .gitkeep                         # 占位文件，确保目录结构可追踪
    openbmc/                         # openbmc/openbmc 主仓库（git clone）
      build/<machine>/               # per-machine 独立构建目录
    configs/
      <machine>.lock                 # per-machine 子仓库依赖锁定文件（JSON）
    src/
      <machine>/                     # 该 machine 的子仓库源码根目录
        linux/                       # 例：linux 内核
        phosphor-dbus-interfaces/    # 例：D-Bus 接口定义
        ...                          # 其他 bitbake 依赖的子仓库
```

说明：
- `workspace/` 整体加入 `.gitignore`，仅保留 `.gitkeep` 文件
- 主仓库 clone 到 `workspace/openbmc/`，子仓库 clone 到 `workspace/src/<machine>/`
- 每个 machine 有独立构建目录 `workspace/openbmc/build/<machine>/`
- lockfile 存放在 `workspace/configs/<machine>.lock`，供本地脚本消费，不提交到 upstream

## `ob init <machine>` 执行流程

```
ob init <machine>
  |
  +-- Step 1: 前置检查
  |   +-- 检查当前 OS（必须 Linux）
  |   +-- 检查必需工具：git, bitbake, python3
  |   +-- 检查网络连通性（github.com）
  |
  +-- Step 2: Clone 主仓库
  |   +-- git clone https://github.com/openbmc/openbmc.git workspace/openbmc/
  |   +-- （已存在则跳过，提示用户手动 git pull 更新）
  |
  +-- Step 3: 初始化 bitbake 环境
  |   +-- cd workspace/openbmc/
  |   +-- source setup <machine> build/<machine>     # 使用 OpenBMC 官方 setup 脚本
  |   +-- setup 自动处理 TEMPLATECONF、bblayers.conf、local.conf（含 MACHINE 设置）
  |   +-- 注：需要 set +u 保护，因为 setup 内部引用未定义变量
  |
  +-- Step 4: 生成依赖图
  |   +-- bitbake -g obmc-phosphor-image  # 生成 pn-buildlist
  |   +-- Python 辅助脚本逐 recipe 调用 bitbake -e <recipe>
  |   +-- 从每个 recipe 的 bitbake -e 输出中提取 SRC_URI 和 SRCREV
  |   +-- 注：bitbake -e 不带 recipe 名只输出全局变量，不含 per-recipe SRC_URI/SRCREV
  |
  +-- Step 5: Clone 子仓库
  |   +-- 按 SRC_URI 逐个 clone 到 workspace/src/<machine>/
  |   +-- 已存在的子仓库执行 git fetch（不强制 reset）
  |   +-- checkout 到 SRCREV 指定的 commit
  |
  +-- Step 6: 生成 lockfile
  |   +-- 输出 workspace/configs/<machine>.lock（JSON 格式，见下方定义）
  |
  +-- Step 7: 注入 externalsrc 配置
  |   +-- 生成独立文件 workspace/openbmc/build/<machine>/conf/externalsrc-<machine>.inc
  |   +-- 配置共享下载和构建缓存（多 machine 复用）：
  |   |   DL_DIR = "${TOPDIR}/../../../downloads"
  |   |   SSTATE_DIR = "${TOPDIR}/../../../sstate-cache"
  |   +-- 对每个子仓库生成：
  |   |   EXTERNALSRC_pn-<recipe> = "<绝对路径>/workspace/src/<machine>/<repo>"
  |   |   EXTERNALSRC_BUILD_pn-<recipe> = "<绝对路径>/workspace/src/<machine>/<repo>"
  |   |   inherit externalsrc（通过 pn 覆盖）
  |   +-- 在 local.conf 末尾追加：include externalsrc-<machine>.inc
  |   +-- 如果 include 已存在则跳过
  |   +-- 备份原有 local.conf 为 local.conf.bak.<timestamp>
  |
  +-- Step 8: 输出状态报告
      +-- 成功 clone 的子仓库数量和列表
      +-- 失败的子仓库（含错误原因）
      +-- lockfile 路径
      +-- externalsrc 配置文件路径
      +-- 下一步建议：cd workspace/openbmc && source setup <machine> build/<machine> && bitbake obmc-phosphor-image
```

## Lockfile 格式

```json
{
  "machine": "romulus",
  "generated_at": "2026-05-31T10:00:00+08:00",
  "openbmc_commit": "abc123...",
  "target_image": "obmc-phosphor-image",
  "sub_repos": [
    {
      "name": "linux",
      "src_uri": "git://github.com/openbmc/linux;protocol=https;branch=dev-6.6",
      "srcrev": "def456...",
      "local_path": "workspace/src/romulus/linux",
      "recipe": "linux-obmc_6.6.bb"
    },
    {
      "name": "phosphor-dbus-interfaces",
      "src_uri": "git://github.com/openbmc/phosphor-dbus-interfaces;protocol=https",
      "srcrev": "789abc...",
      "local_path": "workspace/src/romulus/phosphor-dbus-interfaces",
      "recipe": "phosphor-dbus-interfaces_git.bb"
    }
  ]
}
```

## 外部 src 配置文件格式

文件路径：`workspace/openbmc/build/<machine>/conf/externalsrc-<machine>.inc`

```bitbake
# Auto-generated by ob init <machine> at 2026-05-31T10:00:00+08:00
# Do not edit manually. Re-run 'ob init <machine>' to regenerate.

# Shared across machines to avoid re-downloading/re-building
DL_DIR = "${TOPDIR}/../../../downloads"
SSTATE_DIR = "${TOPDIR}/../../../sstate-cache"

EXTERNALSRC_pn-linux-obmc = "/abs/path/to/workspace/src/<machine>/linux"
EXTERNALSRC_BUILD_pn-linux-obmc = "/abs/path/to/workspace/src/<machine>/linux"

EXTERNALSRC_pn-phosphor-dbus-interfaces = "/abs/path/to/workspace/src/<machine>/phosphor-dbus-interfaces"
EXTERNALSRC_BUILD_pn-phosphor-dbus-interfaces = "/abs/path/to/workspace/src/<machine>/phosphor-dbus-interfaces"

# ... (one block per sub-repo)
```

local.conf 中追加：
```
include externalsrc-<machine>.inc
```

## 错误处理

| 错误场景 | 处理方式 |
|---|---|
| 主仓库 clone 失败 | 终止，输出网络错误信息和建议 |
| bitbake -g 失败 | 终止，提示检查 machine 名称是否有效、bitbake 环境是否完整 |
| 子仓库 clone 失败 | 记录失败，继续处理其他子仓库，最终在报告中汇总 |
| 子仓库 checkout 到 SRCREV 失败 | 先尝试 git fetch --all，再重试；仍失败则标记并跳过 |
| externalsrc include 已存在 | 检测 include 行已存在则跳过，不重复追加 |
| externalsrc .inc 文件已存在 | 备份旧文件为 .bak.<timestamp>，重新生成 |
| workspace/ 目录权限问题 | 终止，提示修复权限 |

## 测试策略

1. **单元验证**：lockfile 格式校验（JSON schema）；externalsrc .inc 文件格式校验
2. **集成验证**：选一个公开 machine（如 `romulus` 或 `witherspoon`），完整跑通 `ob init <machine>`，确认 bitbake 构建使用本地源码
3. **幂等性验证**：连续跑两次 `ob init <machine>`，确认第二次不破坏已有状态，include 不重复追加
4. **回退验证**：lockfile 存在的情况下，确认新增/删除子仓库的差异检测正确

## 命令入口设计（CLI）

脚本位置：`tools/ob`（bash），可通过 `chmod +x` 后 symlink 到 PATH 或直接 `bash tools/ob init <machine>` 调用。

辅助脚本：`tools/parse_bitbake_deps.py`（Python），负责解析 bitbake 依赖图输出。

```
用法：ob <command> [options] [arguments]

命令：
  init <machine>    一键初始化开发环境
  status [<machine>]  显示当前环境状态（可选子命令，后续实现）

选项：
  --skip-fetch      跳过子仓库 clone/fetch（假设已存在）
  --dry-run         只输出将要执行的操作，不实际执行
  --verbose         详细输出
  -h, --help        显示帮助
```

## AI Skill 层（上层）

后续编写一个 skill 文件 `rules/skills/workflow_obmc_env_init.md`，内容：
- 触发词："初始化 openbmc 环境"、"setup openbmc"、"ob init"
- 执行步骤：调用 `bash tools/ob init <machine>`
- 前置条件检查
- 常见问题排查指南

本设计文档只定义 skill 的存在和职责，不展开实现细节。

## 已决事项（原"未决事项"，全部已确认）

1. **target-image 默认值**：`obmc-phosphor-image` ✓
2. **externalsrc 配置注入方式**：生成独立 `externalsrc-<machine>.inc`，在 local.conf 中 include ✓
3. **workspace/ 目录的 .gitignore 规则**：整体忽略，仅保留空的 .gitkeep 文件 ✓
4. **多 machine 共存时的构建目录**：每个 machine 独立 `build/<machine>` 目录；使用 OpenBMC 官方 `source setup <machine>` 初始化（由社区维护，正确处理 TEMPLATECONF、bblayers.conf、local.conf） ✓
5. **私有子仓库认证**：不做特殊处理；SSH Key 认证对用户透明，git 全局配置即可 ✓
6. **Python 实现还是纯 bash**：bash + Python 混合；bitbake 依赖图解析用 Python 辅助脚本，其余用 bash ✓

## 范围边界

**本设计覆盖**：
- `ob init <machine>` 一键初始化命令
- per-machine lockfile 生成
- externalsrc 独立配置文件生成 + local.conf include 注入
- 目录结构约定
- AI skill 层职责定义

**本设计不覆盖**（后续迭代）：
- `ob status` / `ob update` / `ob reset` 等扩展子命令
- Windows PowerShell wrapper
- CI/CD 集成
- 多 machine 源码共享优化（共享 git objects）
- 私有仓库的特殊认证处理