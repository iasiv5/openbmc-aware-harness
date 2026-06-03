# AI Heartbeat 执行合同 (SOP)

本文档是 AI Heartbeat 的完整执行合同。由各平台入口文件（薄壳）引用执行。

入口文件负责：
1. 确定当前 OS 对应的 `<PYTHON>` 值
2. 引导 agent 读取本文件并完整执行

本文档中所有 `<PYTHON>` 占位符由入口文件在运行时替换为实际 Python 解释器路径。

---

## 启动约束

1. 先读取以下文件，再决定执行路径：
   - `AGENTS.md`
   - `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
   - `periodic_jobs/ai_heartbeat/docs/PRD.md`
2. 不要调用或恢复任何本地 runner、legacy opencode 或 hidden hook execution。
3. observer / reflector 的状态记账必须自动完成，不要要求用户手工补记。

## 输入解释

- 默认模式：没有 override 时，先读取 command-spec 再决定执行计划。
- 支持三种 override：`force observer`、`force reflector`、`force both`。
- 如果用户输入同时包含多个 override，以最后一个明确 override 为准。

## 决策输入

使用命令读取当前状态，不要自行复制 due-task 判定逻辑：

```text
<PYTHON> periodic_jobs/ai_heartbeat/src/v0/heartbeat_preflight.py --command-spec
```

预期会得到 JSON，至少包含：

- `due_tasks`
- `recommended_action`
- `target_date`

## 默认决策表

- `recommended_action == "observer"`：只执行 observer。
- `recommended_action == "reflector"`：只执行 reflector。
- `recommended_action == "observer_and_reflector"`：先执行 observer；只有 observer 成功或 skipped 时才执行 reflector。
- `recommended_action == "none"`：说明当前无需执行，并退出；不要改状态。

override 优先级高于默认决策：

- `force observer`：无视 due 状态，只执行 observer。
- `force reflector`：无视 due 状态，只执行 reflector。
- `force both`：无视 due 状态，按 observer 后 reflector 的顺序执行。

## observer 合同

1. 先读取 `contexts/memory/OBSERVATIONS.md`。
2. 检查是否已存在当前 `target_date` 对应的 `Date: YYYY-MM-DD` 条目。
3. 如果当天条目已存在：
   - 不要重复写入。
   - 立即运行：

```text
<PYTHON> periodic_jobs/ai_heartbeat/src/v0/heartbeat_status_cli.py observer --status skipped --target-date <target_date>
```

   - 然后把 observer 视为 `skipped`，允许 `force both` 或默认双任务路径继续进入 reflector。
4. 如果当天条目不存在：
   - 按 `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md` 与 `periodic_jobs/ai_heartbeat/docs/PRD.md` 定义的 observer 语义，扫描、过滤、归纳并写入 `contexts/memory/OBSERVATIONS.md`。
   - observer 只更新观测层，不要在这一步修改 `rules/`。
   - 成功后立即运行：

```text
<PYTHON> periodic_jobs/ai_heartbeat/src/v0/heartbeat_status_cli.py observer --status success --target-date <target_date>
```

5. 如果 observer 失败：
   - 立即运行：

```text
<PYTHON> periodic_jobs/ai_heartbeat/src/v0/heartbeat_status_cli.py observer --status failed --target-date <target_date> --error "<brief error>"
```

   - 如果当前计划原本包含 reflector，直接停止，不要继续 reflector。

## reflector 合同

1. 读取 `contexts/memory/OBSERVATIONS.md` 与相关规则面。
2. 按 `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md` 与 `periodic_jobs/ai_heartbeat/docs/PRD.md` 定义的 reflector 语义执行晋升、整理和 GC。
3. 成功后立即运行：

```text
<PYTHON> periodic_jobs/ai_heartbeat/src/v0/heartbeat_status_cli.py reflector --status success --target-date <target_date>
```

4. 如果 reflector 失败：

```text
<PYTHON> periodic_jobs/ai_heartbeat/src/v0/heartbeat_status_cli.py reflector --status failed --target-date <target_date> --error "<brief error>"
```

## 写入工具注释（OBSERVATIONS.md）

向 `contexts/memory/OBSERVATIONS.md` 写入 observer 结果时，必须使用以下策略：

1. **优先**：使用 `replace_string_in_file` 工具写入。
2. **验证**：写入后立即用 `tail` 检查文件末尾 10 行，确认内容已实际写入文件。
3. **回退**：如果 `tail` 验证发现内容未写入，立即改用 `run_in_terminal` + `cat >> file << 'EOF'` heredoc 方式追加写入。
4. **二次验证**：heredoc 写入后再次 `tail` 验证末尾 10 行，确认写入成功。

**原因**：某些编辑环境对 `.md` 文件的 `replace_string_in_file` 操作存在静默失败的情况，必须通过尾部验证来检测并回退。

## 输出要求

- 先简洁说明本次决策：observer only、reflector only、both 或 none。
- 如果命中了 override，明确写出命中的 override。
- 如果 observer 因当天已存在条目而 skipped，明确说明 skipped，而不是说 success。
