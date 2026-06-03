# 仓库文档一致性清理 设计文档

## 背景与目标

仓库经过几轮功能开发后，多处文档与实际状态脱节：路由表缺目录、描述引用不存在的功能、幽灵路径误导 AI agent、过时措辞掩盖已落地的机制。同时 README.md 偏向"仓库说明"而非"人类使用手册"，新手拿到仓库后不知道第一步做什么。

**目标**：
1. 让 5 个内部文档的描述与仓库实际状态完全一致
2. 把 README.md 重写为人类可操作的使用手册——拿到仓库后按步骤走就能用起来

**成功标准**：一个新用户读完 README.md 后，无需其他文档就能完成克隆→初始化→构建的全流程。

## 范围

7 个文件的修正/重写：

1. `README.md` — 重写为人类使用手册（最大改动）
2. `rules/WORKSPACE.md` — 补齐缺失路由，修正 `tools/` 描述
3. `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md` — 替换幽灵路径为实际扫描路径
4. `AGENTS.md` — 更新 Heartbeat / Memory System 段落

## 非范围

- 不改 `rules/SOUL.md`、`rules/USER.md`、`rules/COMMUNICATION.md`（内容与实际一致）
- 不改 `rules/axioms/` 和 `rules/skills/`（结构与 INDEX 一致）
- 不改 `.github/` 下的 hooks 和 prompts（功能正常）
- 不改 `ob` 脚本和 `tools/parse_bitbake_deps.py`（代码文件不属于本次文档清理）
- 不新增功能、不重构目录结构

## 方案比较

### 方案 A：逐文件原地修正 + README 重写（推荐）

- 核心思路：内部文档做最小事实修正；README 作为唯一大改项，重写为人类手册
- 优点：改动精准，README 获得面向人类的信息架构，内部文档保持稳定
- 缺点：README 改动量较大

### 方案 B：全部最小修正，不改 README 结构

- 核心思路：所有文件都只做事实修正，README 只更新目录树
- 优点：改动量最小
- 缺点：README 仍然是 AI 导向的说明，新人类用户看到后不知道怎么用

## 推荐方案

**方案 A**。内部文档精确修正，README 重写为人类使用手册。两件事的受众不同，策略也该不同。

## 关键边界与组件职责

### 改动 1：`README.md` 重写为人类使用手册

**目标读者**：刚拿到仓库的 OpenBMC 固件开发者。

**新结构**：

```
# openbmc-aware-harness

一句话定位。

## 快速开始

### 前提条件
- Linux 环境
- git, python3
- 25GB+ 可用磁盘空间
- 网络能访问 GitHub

### 第一步：克隆仓库
git clone https://... && cd openbmc-aware-harness

### 第二步：初始化 OpenBMC 开发环境
./ob init                    # 交互式选择 machine 和源
./ob init romulus            # 直接指定 machine
./ob init romulus --dry-run  # 预览不会真正执行

说明 ob init 的 8 步流程在做什么（用户不需要自己操作，但要知道在等什么）。
说明可选参数：--obmc-url, -v
说明环境变量：OB_OPENBMC_URL
说明它是增量和可中断的——Ctrl+C 后重跑会接着来。

### 第三步：构建固件
cd workspace/openbmc
source setup <machine>
bitbake obmc-phosphor-image

### 第四步：开发和调试
源码在 workspace/src/<machine>/<repo>/ 下，改完重编即可。
externalsrc 已注入，bitbake 会直接用本地源码。

### 查看状态
./ob status          # 查看 OpenBMC 源绑定状态

## 仓库结构
（更新后的目录树，补充 ob、periodic_jobs/、.github/、.claude/）

## AI 协作能力
简要介绍 rules/、skills、heartbeat 系统——说明这些是给 AI agent 用的，
人类用户不需要手动读，但如果想自定义 AI 行为可以参考。

## 致谢
（保留原文不动）
```

**`ob` 脚本功能清单**（供设计参考，不一定全部写入 README）：

| 命令 | 功能 |
|------|------|
| `ob init` | 8 步全自动：前置检查→克隆主仓→解析 machine→初始化 bitbake→生成依赖图→克隆子仓库→生成 lockfile→注入 externalsrc |
| `ob init <machine>` | 跳过交互选择，直接指定 machine |
| `ob init --obmc-url <url>` | 使用自定义 OpenBMC 仓库（企业内网 fork 等） |
| `ob init --dry-run` | 预览所有操作但不执行 |
| `ob status` | 显示当前 OpenBMC 源绑定状态（lock file vs 实际 origin） |
| 环境变量 `OB_OPENBMC_URL` | 非交互模式下指定仓库 URL |

**关键行为要写进 README**：
- 增量且幂等：重跑安全，已有仓库不会重新克隆
- 可中断：Ctrl+C 后重跑会从中断处继续
- 子仓库克隆失败有自动重试（shallow→无分支约束）
- 生成 lockfile 和 externalsrc 配置，无需手动编辑

### 改动 2：`rules/WORKSPACE.md` 路由表补齐

当前缺失 4 个目录的路由，新增：

| 路由条目 | 说明 |
|----------|------|
| `periodic_jobs/` | AI Heartbeat 心跳子系统（PRD、源码、测试、配置） |
| `.claude/skills/` | Claude Code 自定义 skill（brainstorming、writing-plans） |
| `.github/` | Copilot 入口、session hooks、slash command prompts |
| `docs/` | 设计文档（`docs/specs/`）和实施计划（`docs/plans/`） |

### 改动 3：`rules/WORKSPACE.md` 的 `tools/` 描述修正

当前：`工具脚本（邮件、语义搜索、分享报告等）：tools/`

改为：`工具脚本（依赖解析等）：tools/`

### 改动 4：`KNOWLEDGE_BASE.md` 幽灵路径替换

当前引用 3 个不存在的路径：
- `contexts/blog/content/`
- `contexts/daily_records/`
- `contexts/life_record/`

替换为仓库内实际有意义的 observer 扫描目标：

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

同时删除 "Blog 内容识别" 这一节（本仓库没有 blog 内容），将"路径白名单与黑名单"改为上述实际扫描路径表。

### 改动 5：`AGENTS.md` Heartbeat 段落更新

将"手动积累"这一行从：
> （稍后将 periodic_jobs/ai_heartbeat/ 中的 observer（L1，当天观测）与 reflector（L2，每周反思）两种任务，改造成手动触发的记忆积累机制 - slash command）

改为反映当前实际状态：slash command 已落地为 `.github/prompts/ai-heartbeat.prompt.md`，通过 `/ai-heartbeat` 手动触发。去掉"稍后将"的前缀语气。

## 数据流 / 控制流

无新增数据流。本次只修改文档内容。

## 错误处理与回退

无风险改动。所有修改都是文本内容修正，不影响任何运行时行为。如改动后发现问题，`git revert` 即可。

## 测试策略

- **验证 1**：WORKSPACE.md 每个路由条目指向的目录在仓库中真实存在
- **验证 2**：README.md 中的每条命令都是可执行的（路径、参数与 `ob` 脚本一致）
- **验证 3**：README.md 目录树与仓库实际结构一致
- **验证 4**：KNOWLEDGE_BASE.md 中引用的扫描路径全部在仓库中存在
- **验证 5**：AGENTS.md 对 Heartbeat 系统的描述与 `.github/prompts/ai-heartbeat.prompt.md` 的实际功能一致
- **验证 6**：README.md 的 ## 致谢 章节保留原文

## 未决事项

无。
