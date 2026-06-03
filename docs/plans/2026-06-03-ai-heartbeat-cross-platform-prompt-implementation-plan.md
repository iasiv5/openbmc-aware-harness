# AI Heartbeat 跨平台 Prompt 重构 实施计划

## 目标

把 `.github/prompts/ai-heartbeat.prompt.md` 中的完整业务逻辑提取为平台无关的 SOP 文件，然后用两个薄壳入口（Copilot / Claude Code）各自引用 SOP。两个入口均内置 OS 探测逻辑以选择正确的 Python 解释器路径。OBSERVATIONS.md 写入策略（replace 优先 + terminal heredoc 回退）纳入 SOP。

## 架构快照

当前状态：一个文件 `.github/prompts/ai-heartbeat.prompt.md` 同时承担 Copilot 入口 + 完整执行合同 + Windows-only Python 路径。

目标状态（三层分离）：

```
periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md    ← 完整执行合同（无平台/OS 绑定）
.github/prompts/ai-heartbeat.prompt.md                  ← Copilot 薄壳（frontmatter + OS 探测 + 引用 SOP）
.claude/commands/ai-heartbeat.md                        ← Claude Code 薄壳（无 frontmatter + OS 探测 + 引用 SOP）
```

SOP 中 `<PYTHON>` 占位符由薄壳在运行时替换为实际值。Python 脚本（`heartbeat_preflight.py`、`heartbeat_status_cli.py`）已经是跨平台的，无需修改。

## 输入工件

- 设计文档：`docs/specs/2026-06-03-ai-heartbeat-cross-platform-prompt-design.md`
- 当前入口：`.github/prompts/ai-heartbeat.prompt.md`
- 知识参考：`periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`、`periodic_jobs/ai_heartbeat/docs/PRD.md`

## 文件结构与职责

- Create: `periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md` — 完整执行合同（observer/reflector 合同、决策表、状态回写、写入工具注释），使用 `<PYTHON>` 占位符
- Modify: `.github/prompts/ai-heartbeat.prompt.md` — 从 120 行完整内容改为约 30 行薄壳（frontmatter + 启动读取 + OS 探测 + 引用 SOP）
- Create: `.claude/commands/ai-heartbeat.md` — Claude Code 薄壳（标题 + 启动读取 + OS 探测 + 引用 SOP），无 YAML frontmatter

## 任务清单

### Task 1: 创建 AI_HEARTBEAT_SOP.md 执行合同

- 目标：从当前 `.github/prompts/ai-heartbeat.prompt.md` 提取全部业务逻辑，写入平台无关的 SOP 文件
- 涉及文件：Create `periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`
- 验证范围：SOP 文件存在、无 Windows/Linux 特定内容、所有 `<PYTHON>` 占位符正确

- [ ] Step 1: 确认当前 prompt 全文内容
- Run: `cat .github/prompts/ai-heartbeat.prompt.md`
- Expected: 完整的 120 行 prompt 文件内容，包含全部 observer/reflector 合同
- [ ] Step 2: 创建 SOP 文件
- Change: 创建 `periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`，内容包括：
  1. 标题 `# AI Heartbeat 执行合同 (SOP)`
  2. 启动约束段落（读 AGENTS.md、KNOWLEDGE_BASE.md、PRD.md）
  3. 输入解释和 override 规则
  4. 决策输入（使用 `<PYTHON>` 占位符替代 `.\.venv\Scripts\python.exe`）
  5. 默认决策表
  6. observer 合同（4 条规则，所有 Python 调用使用 `<PYTHON>`）
  7. reflector 合同（4 条规则，所有 Python 调用使用 `<PYTHON>`）
  8. 写入工具注释段：OBSERVATIONS.md 写入时优先使用 replace_string_in_file；若写入后 `tail` 验证发现内容未实际写入，立即回退到 `run_in_terminal` + `cat >> file << 'EOF'` heredoc，写入后再 `tail` 验证最后 10 行
  9. 输出要求
- 注意：SOP 中不出现 `powershell`、`bash`、`Windows`、`Linux`、`Copilot`、`Claude Code` 字样；代码块标记一律用 `text`
- [ ] Step 3: 验证 SOP 文件无平台绑定
- Run: `grep -in 'powershell\|bash\|windows\|linux\|copilot\|claude.code' periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`
- Expected: 无匹配（退出码 1）
- [ ] Step 4: 验证 SOP 中所有 Python 调用使用占位符
- Run: `grep -c '<PYTHON>' periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`
- Expected: 不少于 5 处（preflight 1 + observer status 3 + reflector status 2 = 6）
- [ ] Step 5: 可选 checkpoint commit
- Run: `git add periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md && git commit -m "feat(heartbeat): add platform-agnostic SOP"`

### Task 2: 改造 ai-heartbeat.prompt.md 为 Copilot 薄壳

- 目标：将当前 120 行完整内容替换为约 30 行薄壳（frontmatter + OS 探测 + 引用 SOP）
- 涉及文件：Modify `.github/prompts/ai-heartbeat.prompt.md`
- 验证范围：文件被缩短到 40 行以内、包含 OS 探测逻辑、引用 SOP 文件

- [ ] Step 1: 确认当前 prompt 行数
- Run: `wc -l .github/prompts/ai-heartbeat.prompt.md`
- Expected: 120 行左右
- [ ] Step 2: 用薄壳内容覆盖当前文件
- Change: 将 `.github/prompts/ai-heartbeat.prompt.md` 全文替换为：
  1. YAML frontmatter（`agent: agent`，`description: 智能执行 AI Heartbeat 的 observer / reflector`）
  2. 标题 `# AI Heartbeat (Copilot Entry)`
  3. 启动读取段落：读 AGENTS.md、KNOWLEDGE_BASE.md、PRD.md
  4. OS 探测段落：
     ```
     ## OS 探测
     检测当前操作系统来确定 Python 解释器路径（`<PYTHON>`）：
     - 先尝试 `python --version`：如果成功，则 `<PYTHON> = python`
     - 如果 python 不可用，尝试 `.\.venv\Scripts\python.exe --version`：如果成功，则 `<PYTHON> = .\.venv\Scripts\python.exe`
     - 如果两者都不可用，停止并提示用户确保 Python 可用
     ```
  5. 引用指令：
     ```
     ## 执行合同
     读取 `periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md`。
     将其中所有 `<PYTHON>` 替换为上面确定的值，然后按完整合同执行。
     不要跳过或省略合同中的任何步骤。
     ```
- [ ] Step 3: 验证薄壳行数
- Run: `wc -l .github/prompts/ai-heartbeat.prompt.md`
- Expected: 40 行以内
- [ ] Step 4: 验证薄壳不再包含完整业务逻辑
- Run: `grep -c 'observer 合同\|reflector 合同\|heartbeat_status_cli' .github/prompts/ai-heartbeat.prompt.md`
- Expected: 0（这些业务细节已移到 SOP）
- [ ] Step 5: 可选 checkpoint commit
- Run: `git add .github/prompts/ai-heartbeat.prompt.md && git commit -m "refactor(heartbeat): convert prompt to thin shell referencing SOP"`

### Task 3: 创建 Claude Code 入口薄壳

- 目标：新建 `.claude/commands/ai-heartbeat.md` 作为 Claude Code 的 `/ai-heartbeat` 入口
- 涉及文件：Create `.claude/commands/ai-heartbeat.md`
- 验证范围：文件存在、格式正确、包含 OS 探测和 SOP 引用

- [ ] Step 1: 确认 .claude/commands/ 目录不存在
- Run: `ls -la .claude/commands/ 2>/dev/null; echo "RC=$?"`
- Expected: 目录不存在（RC \!= 0）
- [ ] Step 2: 创建目录和文件
- Change: 创建 `.claude/commands/ai-heartbeat.md`，内容包括：
  1. 标题 `# AI Heartbeat (Claude Code Entry)`
  2. 启动读取段落（同 Copilot 薄壳）
  3. OS 探测段落（同 Copilot 薄壳）
  4. 引用指令（同 Copilot 薄壳，路径保持相对项目根目录）
- 注意：Claude Code commands 不使用 YAML frontmatter；第一行即标题
- [ ] Step 3: 验证文件存在且内容正确
- Run: `cat .claude/commands/ai-heartbeat.md`
- Expected: 约 30 行，包含标题、启动读取、OS 探测、SOP 引用
- [ ] Step 4: 验证两个薄壳的 OS 探测和引用指令一致（复制粘贴）
- Run: `diff <(sed -n '/## OS 探测/,/不要跳过/p' .github/prompts/ai-heartbeat.prompt.md) <(sed -n '/## OS 探测/,/不要跳过/p' .claude/commands/ai-heartbeat.md)`
- Expected: 无差异
- [ ] Step 5: 可选 checkpoint commit
- Run: `git add .claude/commands/ai-heartbeat.md && git commit -m "feat(heartbeat): add Claude Code entry shell"`

### Task 4: 回归验证

- 目标：确认重构后 Copilot 入口仍能正确触发 observer 流程
- 涉及文件：无新改动
- 验证范围：Linux 环境下通过 Copilot 触发 `/ai-heartbeat`，OS 探测选择 `python`，preflight 能执行

- [ ] Step 1: 验证 Python 可用
- Run: `python --version`
- Expected: 成功输出 Python 版本号
- [ ] Step 2: 验证 preflight 脚本可执行
- Run: `python periodic_jobs/ai_heartbeat/src/v0/heartbeat_preflight.py --command-spec`
- Expected: 输出包含 `recommended_action` 和 `target_date` 的 JSON
- [ ] Step 3: 验证 SOP 中引用的所有文件路径有效
- Run: `test -f AGENTS.md && test -f periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md && test -f periodic_jobs/ai_heartbeat/docs/PRD.md && test -f contexts/memory/OBSERVATIONS.md && echo "ALL_EXIST"`
- Expected: `ALL_EXIST`

## 执行纪律

- 每个任务完成后立即验证
- 若验证失败，停下来定位原因再继续
- 不跳过 Step 1（当前状态检查）

## 最终验证

Linux 环境下执行：

```bash
# 1. SOP 无平台绑定
grep -in 'powershell\|bash\|windows\|linux\|copilot\|claude.code' periodic_jobs/ai_heartbeat/docs/AI_HEARTBEAT_SOP.md
# Expected: 无匹配

# 2. Copilot 薄壳不包含完整业务逻辑
grep -c 'observer 合同\|reflector 合同\|heartbeat_status_cli' .github/prompts/ai-heartbeat.prompt.md
# Expected: 0

# 3. Claude Code 入口存在且约 30 行
wc -l .claude/commands/ai-heartbeat.md
# Expected: 20-40 行

# 4. preflight 可正常执行
python periodic_jobs/ai_heartbeat/src/v0/heartbeat_preflight.py --command-spec
# Expected: JSON 输出含 recommended_action + target_date

# 5. 两个薄壳 OS 探测段一致
diff <(sed -n '/## OS 探测/,/不要跳过/p' .github/prompts/ai-heartbeat.prompt.md) <(sed -n '/## OS 探测/,/不要跳过/p' .claude/commands/ai-heartbeat.md)
# Expected: 无差异
```
