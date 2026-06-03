# AI Heartbeat 跨平台 Prompt 重构 设计文档

## 背景与目标

`ai-heartbeat` 是仓库的定时认知维护机制，通过 observer（L1 观测）和 reflector（L2 反思）两个任务对抗上下文腐烂。当前执行入口 `.github/prompts/ai-heartbeat.prompt.md` 存在两个问题：

1. **平台绑定**：Python 调用全部硬编码为 PowerShell 语法 + Windows 路径（`.\.venv\Scripts\python.exe`），在 Linux 上无法执行
2. **单平台入口**：只有 Copilot 的 `.github/prompts/` 入口，没有 Claude Code 的 `.claude/commands/` 入口

与此同时，Copilot 和 Claude Code 均可能运行在 Windows 或 Linux 上，真正的变量只有 1 个：**Python 解释器路径**（Windows: `.venv\Scripts\python.exe`，Linux: `python`）。

**目标**：将 `ai-heartbeat.prompt.md` 中的完整业务逻辑提取为一份平台无关的 SOP 文件，然后用两个薄壳入口（Copilot / Claude Code）各自引用 SOP，且两个入口均内置 OS 探测逻辑以选择正确的 Python 路径。

**成功标准**：
1. 同一份业务逻辑只需维护一处（SOP）
2. Windows + Linux 均可正确调用 Python 脚本
3. Copilot slash command 和 Claude Code slash command 均可触发完整 ai-heartbeat 流程
4. OBSERVATIONS.md 的写入策略（replace 优先 + terminal heredoc 回退）被明确记录在 SOP 中

## 范围

1. 新建 `periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md` — 无平台绑定的完整执行合同
2. 改造 `.github/prompts/ai-heartbeat.prompt.md` — 变为 Copilot 薄壳（frontmatter + OS 探测 + 引用 SOP）
3. 新建 `.claude/commands/ai-heartbeat.md` — Claude Code 薄壳（无 frontmatter + OS 探测 + 引用 SOP）
4. SOP 中纳入 OBSERVATIONS.md 写入工具注释（replace 优先 + terminal heredoc 回退 + 尾部验证）

## 非范围

- 不改 Python 脚本（`heartbeat_preflight.py`、`heartbeat_status_cli.py`、`heartbeat_state.py`）— 它们已经是跨平台的
- 不改 `.github/hooks/pre-session.ps1` — 这是 Windows-only 的提醒层 hook，职责不同
- 不改 `.github/hooks/ai-heartbeat.session-start.json` — 这是 Copilot session-start 配置，已有 Linux text-only 降级
- 不改 `KNOWLEDGE_BASE.md`、`PRD.md` — 它们是知识参考层，不是执行合同
- 不新增测试框架 — SOP 是 Markdown 文本，验证方式为"在两个平台分别手动触发一次 observer"

## 方案比较

### 方案 A：SOP + 双薄壳

- **核心思路**：提取完整业务逻辑到 `AI_HEARTBEAT_SOP.md`（无平台/OS 绑定）。两个入口各自只含 frontmatter + OS 探测 + 一行引用
- **优点**：单点维护业务逻辑；薄壳改动频率极低；职责分层清晰（SOP = 执行合同，入口 = 平台适配）
- **缺点**：多一个文件；入口启动时需额外读一个文件

### 方案 B：双入口各自全量

- **核心思路**：两个入口各自独立包含全部 120 行业务逻辑，仅在 Python 路径处做 OS 探测
- **优点**：自包含，不依赖第三个文件
- **缺点**：业务逻辑改一处要同步改两处；长期必然漂移

### 方案 C：SOP 内联 KNOWLEDGE_BASE.md

- **核心思路**：不新建文件，把执行合同写入 KNOWLEDGE_BASE.md 的新章节
- **优点**：零新文件
- **缺点**：KNOWLEDGE_BASE.md 职责是知识参考而非执行合同；混入后职责模糊；每次 observer 启动时 KNOWLEDGE_BASE.md 已被加载，额外的执行合同细节增加 token 消耗

## 推荐方案

**方案 A：SOP + 双薄壳**

- 维护成本最低：业务逻辑只维护一处
- 职责最清晰：SOP = "做什么"，薄壳 = "怎么调 Python"
- 符合仓库已有分层模式（rules/ = 约束层，KNOWLEDGE_BASE.md = 参考层，SOP = 执行层）
- Trade-off：多一个文件。但 SOP 是纯 Markdown，无 frontmatter、无平台绑定，改动频率远低于入口文件

## 关键边界与组件职责

### AI_HEARTBEAT_SOP.md（新建）

- **文件路径**：`periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`
- **职责**：完整的执行合同——决策表、observer 合同、reflector 合同、状态回写、写入工具注释
- **Python 路径**：使用 `<PYTHON>` 占位符（不含反引号），由薄壳在引用前说明替换规则
- **写入策略注释**：明确记录 OBSERVATIONS.md 写入时 replace 优先、terminal heredoc 回退、写入后 tail 验证
- **无 frontmatter**：不是平台入口，不需要 YAML 头
- **无 OS 探测**：OS 探测由薄壳负责，SOP 只定义"做什么"
- **无平台术语**：不出现 "Copilot"、"Claude Code"、"Windows"、"Linux"、"PowerShell" 等字样

### .github/prompts/ai-heartbeat.prompt.md（改造）

- **职责**：Copilot slash command 入口（薄壳）
- **内容**：
  1. YAML frontmatter（`agent: agent`，`description: ...`）
  2. 启动读取列表（AGENTS.md、KNOWLEDGE_BASE.md、PRD.md）
  3. OS 探测逻辑（检测运行环境，确定 `<PYTHON>` 的实际值）
  4. 一行引用指令："将以下文件中的 `<PYTHON>` 替换为上面确定的值，然后完整执行：`periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`"
- **OS 探测实现**：
  ```
  检测当前操作系统：
  - 如果是 Windows：<PYTHON> = `.\.venv\Scripts\python.exe`
  - 如果是 Linux：<PYTHON> = `python`
  检测方法：尝试 `python --version`，如果成功则用 `python`；否则尝试 `.\.venv\Scripts\python.exe --version`
  ```

### .claude/commands/ai-heartbeat.md（新建）

- **职责**：Claude Code slash command 入口（薄壳）
- **内容**：
  1. 描述行（Claude Code 用第一行 Markdown 标题作为命令描述）
  2. 启动读取列表
  3. OS 探测逻辑（同 Copilot 薄壳）
  4. 一行引用指令（同 Copilot 薄壳，但引用路径可省略 `../../` 前缀）
- **无 YAML frontmatter**：Claude Code commands 不使用 YAML 头

## 数据流 / 控制流

```
用户输入 /ai-heartbeat [override]
         │
         ▼
    ┌─────────────────┐
    │   薄壳入口文件    │  (.github/prompts/ 或 .claude/commands/)
    │  1. 读 AGENTS.md │
    │  2. 读 KB + PRD  │
    │  3. OS 探测       │──→ <PYTHON> = .\.venv\Scripts\python.exe | python
    │  4. 引用 SOP     │
    └────────┬────────┘
             │ 读 AI_HEARTBEAT_SOP.md
             ▼
    ┌─────────────────────────┐
    │       SOP 执行合同        │
    │ 1. 调 heartbeat_preflight│──→ <PYTHON> heartbeat_preflight.py --command-spec
    │    获取 due_tasks JSON   │
    │ 2. 决策表判定             │
    │ 3. 执行 observer         │──→ 写 OBSERVATIONS.md (replace/heredoc)
    │    回写状态               │──→ <PYTHON> heartbeat_status_cli.py observer --status ...
    │ 4. 执行 reflector        │
    │    回写状态               │──→ <PYTHON> heartbeat_status_cli.py reflector --status ...
    └─────────────────────────┘
```

## 错误处理与回退

| 失败模式 | 处理策略 |
|---------|---------|
| OS 探测无法找到 Python | 薄壳中止，提示用户确保 Python 可用（Windows: 安装并激活 venv；Linux: 确保 python 在 PATH） |
| OBSERVATIONS.md 写入 replace_string_in_file 失败 | 回退到 `run_in_terminal` + `cat >> file << 'EOF'` heredoc，写入后 `tail` 验证最后 10 行 |
| heartbeat_preflight.py 执行失败 | SOP 中止，报告错误，不进入 observer/reflector |
| observer 执行失败 | 回写 `--status failed`，不继续 reflector |
| reflector 执行失败 | 回写 `--status failed`，observer 结果保留（不回滚） |

## 测试策略

核心验证行为：
1. **Windows + Copilot**：触发 `/ai-heartbeat`，OS 探测选择 `.venv\Scripts\python.exe`，observer 正常执行
2. **Linux + Copilot**：触发 `/ai-heartbeat`，OS 探测选择 `python`，observer 正常执行
3. **Windows + Claude Code**：触发 `/ai-heartbeat`，同 1
4. **Linux + Claude Code**：触发 `/ai-heartbeat`，同 2
5. **写入回退**：在任一平台上，模拟 replace_string_in_file 失败后 heredoc 回退成功

测试层级：手动冒烟测试（4 个组合 × 1 次 observer）。无自动化测试框架需要搭建。

## 未决事项

无。所有关键设计决策已在澄清阶段确认：
- 入口策略：双薄壳 + SOP（已确认）
- 写入策略：replace 优先 + heredoc 回退（已确认）
- 文件路径：SOP 在 `periodic_jobs/ai_heartbeat/docs/`，Copilot 入口在 `.github/prompts/`，Claude Code 入口在 `.claude/commands/`（已确认）
