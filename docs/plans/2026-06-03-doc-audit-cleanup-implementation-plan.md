# 仓库文档一致性清理 实施计划

## 目标

按照已批准的设计文档，修正 4 个文件的文档描述，使其与仓库实际状态一致。

## 架构快照

本次不引入新文件、新目录或新机制。只对现有文档做事实性修正：
- AGENTS.md、WORKSPACE.md、KNOWLEDGE_BASE.md 做最小精确修正
- README.md 做结构重写，从"仓库说明"变为"人类使用手册"

## 输入工件

- 设计文档：`docs/specs/2026-06-03-doc-audit-cleanup-design.md`

## 文件结构与职责

| 文件 | 动作 | 职责 |
|------|------|------|
| `AGENTS.md` | Modify | 修正 Memory System 小节的 Heartbeat 描述 |
| `rules/WORKSPACE.md` | Modify | 补齐 4 个缺失路由条目 + 修正 `tools/` 描述 |
| `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md` | Modify | 替换 3 个幽灵路径 + 删除 Blog 章节 + 重写扫描规则 |
| `README.md` | Modify | 重写为人类使用手册，保留致谢原文 |

## 任务清单

### Task 1: 更新 AGENTS.md Heartbeat 描述

- 目标：将"稍后将…改造成"的过时措辞改为反映 slash command 已落地的当前状态
- Files: Modify `AGENTS.md` → Memory System 小节 → "手动积累"行
- 验证范围：AGENTS.md 中不再包含"稍后将"或"改造成"这类未完成语气

- [ ] Step 1: 确认当前文本
- Run: `grep -n "稍后将" AGENTS.md`
- Expected: 命中 Memory System 小节的一行，包含"稍后将 periodic_jobs/ai_heartbeat/…"

- [ ] Step 2: 替换为当前状态描述
- Change: 将该行替换为：`- **手动积累**：通过 `/ai-heartbeat` slash command（[实现](.github/prompts/ai-heartbeat.prompt.md)）手动触发 observer（L1，当天观测）和 reflector（L2，每周反思）。`

- [ ] Step 3: 验证替换结果
- Run: `grep -n "稍后将" AGENTS.md`
- Expected: 无匹配（过时措辞已删除）
- Run: `grep -n "/ai-heartbeat" AGENTS.md`
- Expected: 命中新写入的行，包含 slash command 引用

### Task 2: 补齐 WORKSPACE.md 路由表

- 目标：在"项目与代码"或"系统与规则"分区下新增 4 个缺失目录的路由条目
- Files: Modify `rules/WORKSPACE.md`
- 验证范围：WORKSPACE.md 包含 `periodic_jobs/`、`.claude/skills/`、`.github/`、`docs/` 的路由条目

- [ ] Step 1: 确认当前缺失项
- Run: `grep -c "periodic_jobs\|\.claude/skills\|\.github\|docs/" rules/WORKSPACE.md`
- Expected: 1（仅 docs/ 出现在命名规则区注释中）或 0

- [ ] Step 2: 在"系统与规则"分区末尾新增路由条目
- Change: 在 `rules/skills/` 条目之后追加：

```markdown
- AI Heartbeat 心跳子系统（PRD、源码、测试、配置）：`periodic_jobs/ai_heartbeat/`
- 设计文档：`docs/specs/`
- 实施计划：`docs/plans/`
- GitHub Copilot 入口与 hooks：`.github/`
- Claude Code 自定义 skill：`.claude/skills/`
```

- [ ] Step 3: 验证新增条目
- Run: `grep "periodic_jobs\|\.claude/skills\|\.github\|docs/specs\|docs/plans" rules/WORKSPACE.md`
- Expected: 每个关键词至少命中一次

### Task 3: 修正 WORKSPACE.md 的 tools/ 描述

- 目标：将 `tools/` 路由描述从引用不存在的功能改为实际内容
- Files: Modify `rules/WORKSPACE.md` → "项目与代码"区 → `tools/` 行
- 验证范围：tools/ 描述不再包含"邮件、语义搜索、分享报告"

- [ ] Step 1: 确认当前文本
- Run: `grep -n "邮件" rules/WORKSPACE.md`
- Expected: 命中 `tools/` 路由行

- [ ] Step 2: 替换描述
- Change: 将 `工具脚本（邮件、语义搜索、分享报告等）：tools/` 替换为 `工具脚本（依赖解析等）：tools/`

- [ ] Step 3: 验证替换结果
- Run: `grep -n "邮件\|语义搜索\|分享报告" rules/WORKSPACE.md`
- Expected: 无匹配
- Run: `grep -n "tools/" rules/WORKSPACE.md`
- Expected: 命中修正后的行，包含"依赖解析"

### Task 4: 替换 KNOWLEDGE_BASE.md 幽灵路径

- 目标：删除 3 个不存在的路径引用，删除 Blog 内容识别章节，替换为仓库实际扫描路径表
- Files: Modify `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- 验证范围：文档中不再包含 `contexts/blog/`、`contexts/daily_records/`、`contexts/life_record/`

- [ ] Step 1: 确认当前幽灵路径
- Run: `grep -n "contexts/blog\|contexts/daily_records\|contexts/life_record" periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- Expected: 命中 2.2 Blog 内容识别章节和 2.3 路径白名单章节中的引用

- [ ] Step 2: 删除 Blog 内容识别章节（2.2 节整体）
- Change: 删除 `### 2.2 Blog 内容识别` 小节的全部内容

- [ ] Step 3: 重写路径白名单与黑名单（2.3 节）
- Change: 将 `### 2.3 路径白名单与黑名单` 的内容替换为：

```markdown
### 2.3 扫描路径表

Observer 扫描以下路径，检测有意义的变更：

| 扫描路径 | 扫描语义 |
|----------|---------|
| `docs/specs/` | 新设计文档 = 新功能在规划 |
| `docs/plans/` | 新实施计划 = 功能在构建 |
| `rules/` | 核心规则变动 = 系统进化 |
| `rules/skills/` | 新增 skill = 新能力 |
| `rules/axioms/` | 新增公理 = 认知更新 |
| `ob` | 主工具脚本变更 |
| `tools/` | 工具变更 |
| `.claude/skills/` | 自定义 skill 变更 |

**忽略**：`workspace/`（整体 gitignore，内容由 `ob init` 管理）、`.venv/`（Python 虚拟环境）、`__pycache__/`。
```

- [ ] Step 4: 验证幽灵路径已清除
- Run: `grep -c "contexts/blog\|contexts/daily_records\|contexts/life_record" periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- Expected: 0

- [ ] Step 5: 验证新扫描路径已写入
- Run: `grep "docs/specs\|rules/skills\|rules/axioms\|\.claude/skills" periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- Expected: 每个关键词至少命中一次

### Task 5: 重写 README.md 为人类使用手册

- 目标：将 README.md 从 AI 导向的仓库说明重写为人类可操作的使用手册
- Files: Modify `README.md`（全文重写）
- 验证范围：(1) 包含分步操作指南 (2) `ob` 命令与脚本实际参数一致 (3) 目录树与仓库一致 (4) 致谢章节保留原文

- [ ] Step 1: 记录当前致谢原文
- Run: `sed -n '/^## 致谢/,/^$/p' README.md`
- Expected: 输出致谢章节的完整内容，后续重写时原样保留

- [ ] Step 2: 重写 README.md
- Change: 用以下内容替换 README.md 全文。致谢章节使用 Step 1 记录的原文。

```markdown
# openbmc-aware-harness

面向 OpenBMC 固件开发的一键开发环境初始化工具。`ob init` 自动完成主仓库克隆、machine 选择、bitbake 环境初始化、子仓库拉取、lockfile 生成和 externalsrc 注入，开箱即用。

## 快速开始

### 前提条件

- Linux 环境
- `git`、`python3`
- 25 GB+ 可用磁盘空间
- 网络能访问 GitHub（或企业内网 Git 服务器）

### 第一步：克隆仓库

```bash
git clone https://github.com/<your-org>/openbmc-aware-harness.git
cd openbmc-aware-harness
```

### 第二步：初始化 OpenBMC 开发环境

```bash
# 交互式选择 machine 和源
./ob init

# 直接指定 machine
./ob init romulus

# 预览操作但不执行
./ob init romulus --dry-run

# 使用企业内网 fork
./ob init romulus --obmc-url https://git.example.com/openbmc.git

# 只指定仓库源，machine 交互选择
./ob init --obmc-url https://git.example.com/openbmc.git
```

`ob init` 会自动执行以下 8 步：

1. **前置检查**：验证 OS、工具链、网络和磁盘空间
2. **克隆主仓库**：下载 OpenBMC 主仓库（社区版或自定义 URL）
3. **解析 machine**：交互选择或直接确认目标 machine
4. **初始化 bitbake**：执行 `source setup <machine>` 创建构建目录
5. **生成依赖图**：运行 `bitbake -g` 并逐 recipe 解析 SRC_URI/SRCREV
6. **拉取子仓库**：克隆约 100+ 个 git 子仓库到 `workspace/src/<machine>/`
7. **生成 lockfile**：记录所有子仓库的 commit hash，写入 `workspace/configs/<machine>.lock`
8. **注入 externalsrc**：生成 `externalsrc-<machine>.inc` 并 include 到 `local.conf`

**关键行为**：
- **增量且幂等**：已有仓库不会重新克隆，重跑只会 fetch 更新和补齐缺失项
- **可中断**：Ctrl+C 后重跑会从中断处继续，不会从头开始
- **自动重试**：子仓库克隆失败时自动尝试 shallow clone → 无分支约束 clone

### 第三步：构建固件

```bash
cd workspace/openbmc
source setup <machine>
bitbake obmc-phosphor-image
```

### 第四步：开发和调试

源码位于 `workspace/src/<machine>/<repo>/`，externalsrc 已注入，改完源码后直接重编：

```bash
cd workspace/openbmc
source setup <machine>
bitbake <recipe>
```

### 查看环境状态

```bash
./ob status           # 查看 OpenBMC 源绑定状态
./ob status romulus   # 查看指定 machine 的状态
```

## 命令参考

```
ob <command> [options] [arguments]

Commands:
  init   [<machine>]    一键初始化 OpenBMC 开发环境
  status [<machine>]    查看当前 OpenBMC 源绑定状态

Options:
  --dry-run             预览操作但不执行
  --obmc-url <url>      使用自定义 OpenBMC 仓库 URL
  -v, --verbose         详细输出
  -h, --help            显示帮助

环境变量:
  OB_OPENBMC_URL        非交互模式下指定 OpenBMC 仓库 URL
```

## 仓库结构

```
openbmc-aware-harness/
├── ob                         # OpenBMC 开发环境初始化脚本
├── tools/                     # 工具脚本（依赖解析等）
├── workspace/                 # 工作空间（由 ob init 生成，gitignore）
│   ├── openbmc/               # OpenBMC 主仓库
│   ├── src/<machine>/         # 子仓库源码
│   ├── configs/               # lockfile 和报告
│   ├── downloads/             # 下载缓存
│   └── sstate-cache/          # 构建状态缓存
├── rules/                     # AI 协作规则（agent 自动加载）
│   ├── SOUL.md                # AI 身份与行为准则
│   ├── USER.md                # 服务对象画像
│   ├── WORKSPACE.md           # 目录路由表
│   ├── COMMUNICATION.md       # 沟通规范
│   ├── axioms/                # 决策公理（43 条）
│   └── skills/                # 可复用能力（工作流、最佳实践）
├── docs/
│   ├── specs/                 # 设计文档
│   └── plans/                 # 实施计划
├── contexts/memory/           # AI 记忆观测日志
├── .claude/skills/            # Claude Code 自定义 skill
├── CLAUDE.md                   # Claude Code 入口（指向 AGENTS.md）
├── AGENTS.md                   # AI agent 主入口，定义 session 启动读取链
├── .github/
│   ├── copilot-instructions.md # GitHub Copilot 入口（指向 AGENTS.md）
│   ├── hooks/                  # Session 启动 hook（AI Heartbeat 提醒）
│   └── prompts/                # Copilot slash command prompt（/ai-heartbeat）
├── periodic_jobs/ai_heartbeat/# AI Heartbeat 心跳子系统
└── adhoc_jobs/                # 一次性任务和实验
```

## AI 协作能力

本仓库内置了一套 AI agent 协作框架：

- **rules/**：定义 AI 的身份、用户画像、目录路由和沟通风格，每轮 session 自动加载
- **rules/skills/**：可复用的工作流和最佳实践，按需引入
- **rules/axioms/**：从团队经历中提炼的决策原则，用于启发深度思考
- **AI Heartbeat**：通过 `/ai-heartbeat` slash command 手动触发的记忆积累系统，记录观测并定期反思

这些内容是给 AI agent 用的。人类开发者通常不需要手动阅读，但如果想自定义 AI 行为，可以从 `AGENTS.md` 开始。

## 致谢

（此处保留原文）
```

- [ ] Step 3: 将致谢原文回填
- Change: 把 Step 1 记录的致谢原文，替换占位符"（此处保留原文）"

- [ ] Step 4: 验证 `ob` 命令一致性
- Run: `grep -o "ob init\|ob status\|--dry-run\|--obmc-url\|-v\|--verbose\|-h\|--help\|OB_OPENBMC_URL" README.md | sort -u`
- Expected: 每个关键词都出现，且与 `./ob --help` 输出一致

- [ ] Step 5: 验证目录树准确性
- Run: `ls -d ob tools/ docs/ rules/ contexts/ .claude/skills/ .github/ periodic_jobs/ adhoc_jobs/`
- Expected: 每个目录/文件都存在

- [ ] Step 6: 验证致谢章节完整
- Run: `grep -A5 "^## 致谢" README.md`
- Expected: 输出包含 "grapeot" 和 "context-infrastructure"（与原文一致）

## 执行纪律

- 开始实现前，先复查整份计划；如果发现缺项、矛盾或验证命令无效，先修计划
- 按任务顺序执行（Task 1 → 2 → 3 → 4 → 5），不要跳步或合并
- 每完成一个任务，都运行该任务定义的验证
- 遇到计划与仓库现实不符，立即停下说明，不要猜
- 当前在 `main` 分支且工作区干净，直接在 `main` 上实施

## 最终验证

全部任务完成后，依次运行：

```bash
# 1. AGENTS.md：过时措辞已清除
grep -c "稍后将" AGENTS.md
# Expected: 0

# 2. WORKSPACE.md：所有路由条目指向真实目录
for dir in periodic_jobs .claude/skills .github docs/specs docs/plans tools; do
  test -e "$dir" && echo "OK: $dir" || echo "MISSING: $dir"
done

# 3. KNOWLEDGE_BASE.md：幽灵路径已清除
grep -c "contexts/blog\|contexts/daily_records\|contexts/life_record" periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md
# Expected: 0

# 4. README.md：包含分步指南且致谢保留
grep -c "第一步\|第二步\|第三步\|第四步" README.md
# Expected: ≥ 4
grep "grapeot" README.md
# Expected: 命中致谢章节
```

## 审阅 Checkpoint

计划正文结束。请先确认这份实施计划；如果没问题，下一步可以按计划由普通编码 agent 或人工继续执行。
