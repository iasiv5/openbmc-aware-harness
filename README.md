# ob-harness

```raw
      ██████╗ ██████╗ ███████╗ ███╗   ██╗ ██████╗ ███╗   ███╗ ██████╗
     ██╔═══██╗██╔══██╗██╔════╝ ████╗  ██║ ██╔══██╗████╗ ████║██╔════╝ 
     ██║   ██║██████╔╝█████╗   ██╔██╗ ██║ ██████╔╝██╔████╔██║██║      
     ██║   ██║██╔═══╝ ██╔══╝   ██║╚██╗██║ ██╔══██╗██║╚██╔╝██║██║      
     ╚██████╔╝██║     ███████╗ ██║ ╚████║ ██████╔╝██║ ╚═╝ ██║╚██████╗ 
      ╚═════╝ ╚═╝     ╚══════╝ ╚═╝  ╚═══╝ ╚═════╝ ╚═╝     ╚═╝ ╚═════╝ 
     ██╗  ██╗  █████╗  █████╗   ███╗   ██╗ ███████╗ ███████╗ ███████╗ 
     ██║  ██║ ██╔══██╗ ██╔══██╗ ████╗  ██║ ██╔════╝ ██╔════╝ ██╔════╝ 
     ███████║ ███████║ ██████╔╝ ██╔██╗ ██║ █████╗   ███████╗ ███████╗ 
     ██╔══██║ ██╔══██║ ██╔══██╗ ██║╚██╗██║ ██╔══╝   ╚════██║ ╚════██║ 
     ██║  ██║ ██║  ██║ ██║  ██║ ██║ ╚████║ ███████╗ ███████║ ███████║ 
     ╚═╝  ╚═╝ ╚═╝  ╚═╝ ╚═╝  ╚═╝ ╚═╝  ╚═══╝ ╚══════╝ ╚══════╝ ╚══════╝ 
    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
    ┃      OpenBMC Development Environment · ob-harness · 𝓲𝓪𝓼𝓲      ┃
    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

面向 OpenBMC 固件开发的一键开发环境初始化工具。`ob init` 自动完成主仓库克隆、machine 选择、bitbake 环境初始化、依赖解析、bare mirror 缓存填充、lockfile 生成和构建缓存配置，开箱即用。

## 快速开始

### 前提条件

- Linux 环境
- `git`、`python3`
- 30 GB+ 可用磁盘空间
- 网络能访问 [GitHub](https://github.com/openbmc/openbmc.git)（或自定义 OpenBMC Git 服务器）

### 克隆仓库

```bash
git clone https://github.com/iasiv5/ob-harness.git
cd ob-harness
```

### 初始化 OpenBMC 开发环境

```bash
# 交互式选择 machine 和 OpenBMC 主仓源（社区/自定义）—— 推荐
./ob init

# 直接指定 machine
./ob init romulus

# 预览操作但不执行
./ob init romulus --dry-run

# 使用自定义 OpenBMC 仓库 URL
./ob init romulus --url https://git.example.com/openbmc.git

# 只指定自定义 URL，machine 在后续交互时选择
./ob init --url https://git.example.com/openbmc.git
```

### 构建固件

```bash
cd workspace/openbmc
source setup <machine>
bitbake obmc-phosphor-image
```

### 查看环境状态

```bash
./ob status           # 查看 OpenBMC 源绑定状态
```

## ob init 做什么

`ob init` 会依次执行以下步骤：

- **前置检查**：验证 OS、工具链、网络和磁盘空间
- **准备主仓库**：下载 OpenBMC 主仓库（社区版或自定义 URL），解析目标 machine
- **初始化 bitbake**：执行 `source setup <machine>` 创建构建目录，写入 CONNECTIVITY_CHECK_URIS 和 GITLAB_IP 等引导配置
- **生成依赖图**：运行 `bitbake -g` 并逐 recipe 解析 SRC_URI/SRCREV
- **填充 bare mirror 缓存**：将全部 git 子仓库以 bare clone 形式缓存到 `DL_DIR/git2/`，供 BitBake `PREMIRRORS` 加速 fetch
- **生成 lockfile**：记录所有子仓库的 commit hash，写入 `workspace/configs/<machine>.lock`
- **生成构建缓存配置**：生成 `externalsrc-<machine>.inc`（含 DL_DIR/SSTATE_DIR 和 externalsrc 占位），include 到 `local.conf`
- **状态报告**：汇总 mirror 填充结果和失败项，落盘到 `workspace/configs/<machine>.report.txt`

**关键行为**：

- **增量且幂等**：已有 bare mirror 不会重新克隆，重跑只会补齐缺失项
- **可中断**：Ctrl+C 后重跑会从中断处继续，不会从头开始
- **可跳过依赖解析**：已有 `deps.json` 时可用 `--skip-deps` 跳过耗时的 bitbake -g 阶段

## 命令参考

```raw
ob <command> [options] [arguments]

Commands:
  init   [<machine>]    一键初始化 OpenBMC 开发环境
  status                查看当前 OpenBMC 源绑定状态

Options:
  -d, --dry-run         预览操作但不执行
  -s, --skip-deps       跳过依赖解析，复用已有 deps.json
  -u, --url <url>       使用自定义 OpenBMC 仓库 URL
  -v, --verbose         详细输出
  -h, --help            显示帮助

环境变量:
  OB_OPENBMC_URL        非交互模式下指定 OpenBMC 仓库 URL
```

## 仓库结构

```raw
ob-harness/
├── ob                          # OpenBMC 开发环境初始化脚本
├── tools/                      # 工具脚本（依赖解析等）
├── CLAUDE.md                   # Claude Code 入口（指向 AGENTS.md）
├── AGENTS.md                   # AI agent 主入口，定义 session 启动读取链
├── rules/                      # AI 协作规则（agent 自动加载）
│   ├── 01_SOUL.md              # AI 身份与行为准则
│   ├── 02_USER.md              # 服务对象画像
│   ├── 03_WORKSPACE.md         # 目录路由表
│   ├── 04_COMMUNICATION.md     # 沟通规范
│   ├── 05_SKILLS_INDEX.md      # Skills 索引
│   ├── 06_AXIOMS_INDEX.md      # Axioms 索引
│   ├── axioms/                 # 决策公理（43 条）
│   └── skills/                 # 可复用能力（工作流、最佳实践）
├── docs/
│   ├── specs/                  # 设计文档
│   └── plans/                  # 实施计划
├── contexts/memory/            # AI 记忆观测日志
├── .claude/
│   ├── skills/                 # Claude Code/Copilot 自定义 skill
│   └── commands/               # Claude Code 自定义命令（/ai-heartbeat 入口）
├── .github/
│   ├── copilot-instructions.md # GitHub Copilot 入口（指向 AGENTS.md）
│   ├── hooks/                  # Session 启动 hook（AI Heartbeat 提醒）
│   └── prompts/                # Copilot slash command prompt（/ai-heartbeat）
├── periodic_jobs/ai_heartbeat/ # AI Heartbeat 心跳子系统
└── workspace/                  # 工作空间（gitignore，以下子目录由 'ob init' 自动创建）
    ├── openbmc/                #  （After ob init）OpenBMC 主仓库
    ├── configs/                #  （After ob init）lockfile 和报告
    ├── downloads/              #  （After ob init）下载缓存
    │   └── git2/               #  （After ob init）bare mirror 缓存，BitBake PREMIRRORS
    └── sstate-cache/           #  （After ob init）构建状态缓存
```

## AI 协作

本仓库内置 AI agent 上下文框架。用 Claude Code 或 GitHub Copilot 打开本仓库时，agent 会自动加载项目结构、沟通规范和开发约定，无需手动喂上下文。

- **仓库级上下文**：agent 自动理解项目目录、沟通风格和 OpenBMC 开发规范
- **内置 Skills**：环境初始化、调试诊断等可复用工作流，按需引入
- **记忆积累**：通过 `/ai-heartbeat` 让 AI 持续学习项目变化和团队决策
- **决策公理**：从团队经历中提炼的决策原则，辅助深度分析

入口配置在 `AGENTS.md` 和 `.github/copilot-instructions.md`，感兴趣可以翻看源码。

## 致谢

本项目受 [grapeot (Yan Wang / 鸭哥)](https://github.com/grapeot) 的 [context-infrastructure](https://github.com/grapeot/context-infrastructure) 项目启发并基于其架构思路构建。感谢鸭哥在 AI 上下文工程领域的开创性探索。
