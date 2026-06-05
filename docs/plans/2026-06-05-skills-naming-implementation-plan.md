# Skills 文件命名规范重命名 · 实施计划

## 目标

将 `rules/skills/` 下 5 个文件按 `<category>_<NN>-<topic>.md` 格式重命名，并同步更新所有内部引用。

## 架构快照

命名格式从 `<category>_<topic>.md` 切换到 `<category>_<NN>-<topic>.md`，序号按重要性/阅读优先级分配。目录结构和文件内容（引用路径除外）不变。

## 输入工件

- 设计文档：`docs/specs/2026-06-05-skills-naming-convention-design.md`

## 文件结构与职责

- Rename: `rules/skills/bestpractice_skill_writing.md` → `bestpractice_01-skill_writing.md`
- Rename: `rules/skills/bestpractice_ai_programming_mindset.md` → `bestpractice_02-ai_programming_mindset.md`
- Rename: `rules/skills/bestpractice_ai_debugging_diagnosis.md` → `bestpractice_03-ai_debugging_diagnosis.md`
- Rename: `rules/skills/bestpractice_temporal_info_verification.md` → `bestpractice_04-temporal_info_verification.md`
- Rename: `rules/skills/workflow_obmc_env_init.md` → `workflow_01-obmc_env_init.md`
- Modify: `rules/05_SKILLS_INDEX.md` — 更新命名规范说明、重排条目顺序、更新所有引用路径
- Modify: `rules/skills/bestpractice_01-skill_writing.md`（重命名后）— 更新"与现有 skill 的关系"段落中的两个引用路径
- Modify: `AGENTS.md` — 更新 skill 写作指南引用路径
- Modify: `rules/skills/bestpractice_03-ai_debugging_diagnosis.md`（重命名后）— 更新交叉引用路径
- Modify: `rules/skills/bestpractice_02-ai_programming_mindset.md`（重命名后）— 更新交叉引用路径
- Modify: `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md` — 更新 skill 写作指南引用路径

---

## 任务清单

### Task 1: 重命名前状态检查

- 目标：确认 5 个源文件均存在且未被提前改名
- 涉及文件：`rules/skills/` 下 5 个文件
- 验证范围：`ls` 输出包含全部 5 个旧文件名

- [ ] Step 1: 列出当前文件
- Run: `ls rules/skills/`
- Expected: 输出包含 `bestpractice_ai_debugging_diagnosis.md`、`bestpractice_ai_programming_mindset.md`、`bestpractice_skill_writing.md`、`bestpractice_temporal_info_verification.md`、`workflow_obmc_env_init.md`

### Task 2: 执行 git mv 重命名

- 目标：将 5 个文件重命名为新命名格式
- 涉及文件：`rules/skills/` 下 5 个文件
- 验证范围：`ls` 输出包含全部 5 个新文件名

- [ ] Step 1: 执行 5 条 git mv 命令
- Run:
  ```bash
  git mv rules/skills/bestpractice_skill_writing.md rules/skills/bestpractice_01-skill_writing.md && \
  git mv rules/skills/bestpractice_ai_programming_mindset.md rules/skills/bestpractice_02-ai_programming_mindset.md && \
  git mv rules/skills/bestpractice_ai_debugging_diagnosis.md rules/skills/bestpractice_03-ai_debugging_diagnosis.md && \
  git mv rules/skills/bestpractice_temporal_info_verification.md rules/skills/bestpractice_04-temporal_info_verification.md && \
  git mv rules/skills/workflow_obmc_env_init.md rules/skills/workflow_01-obmc_env_init.md
  ```
- Expected: 5 条命令均无报错

- [ ] Step 2: 确认新文件名
- Run: `ls rules/skills/`
- Expected: 输出为 `bestpractice_01-skill_writing.md`、`bestpractice_02-ai_programming_mindset.md`、`bestpractice_03-ai_debugging_diagnosis.md`、`bestpractice_04-temporal_info_verification.md`、`workflow_01-obmc_env_init.md`，且按此顺序排列（类型聚簇 + 序号递增）

### Task 3: 更新 05_SKILLS_INDEX.md

- 目标：更新索引文件中的命名规范说明、条目顺序和所有文件引用路径
- 涉及文件：`rules/05_SKILLS_INDEX.md`
- 验证范围：文件中无旧文件名引用，条目按序号排列

- [ ] Step 1: 确认当前内容包含旧引用
- Run: `grep -n 'bestpractice_\|workflow_o' rules/05_SKILLS_INDEX.md`
- Expected: 命中若干行，均为旧文件名

- [ ] Step 2: 更新文件内容

改动 1 — Workflow 分类索引（第 16 行）：将 `skills/workflow_obmc_env_init.md` 改为 `skills/workflow_01-obmc_env_init.md`

改动 2 — BestPractice 分类索引（第 22-25 行）：按新序号重排条目顺序，并更新路径：
- `[Skill 写作指南（Meta-Skill）](skills/bestpractice_01-skill_writing.md)` — 排第一
- `[AI 编程核心方法论](skills/bestpractice_02-ai_programming_mindset.md)` — 排第二
- `[AI 辅助调试诊断](skills/bestpractice_03-ai_debugging_diagnosis.md)` — 排第三
- `[时间敏感信息验证](skills/bestpractice_04-temporal_info_verification.md)` — 排第四

改动 3 — "如何添加你自己的 Skill"段落（第 31 行）：将 `[`bestpractice_skill_writing.md`]` 改为 `[`bestpractice_01-skill_writing.md`]`，链接路径同步更新为 `skills/bestpractice_01-skill_writing.md`

改动 4 — 命名规范说明（第 33 行）：将 `文件命名建议采用 \`<category>_<name>.md\`` 改为 `文件命名建议采用 \`<category>_<NN>-<name>.md\``，示例更新为 `workflow_01-my_process.md`、`bestpractice_01-my_insight.md`

- [ ] Step 3: 确认无旧引用残留
- Run: `grep -n 'bestpractice_ai_\|bestpractice_skill\|bestpractice_temporal\|workflow_obmc' rules/05_SKILLS_INDEX.md`
- Expected: 无输出（零匹配）

### Task 4: 更新 bestpractice_01-skill_writing.md 中的引用

- 目标：更新"与现有 skill 的关系"段落中的两个引用路径
- 涉及文件：`rules/skills/bestpractice_01-skill_writing.md`
- 验证范围：文件中无旧文件名引用

- [ ] Step 1: 确认当前内容包含旧引用
- Run: `grep -n 'workflow_obmc_env_init\|bestpractice_temporal_info_verification' rules/skills/bestpractice_01-skill_writing.md`
- Expected: 命中第 92 行，包含两个旧文件名

- [ ] Step 2: 更新引用路径
- Change: 将第 92 行的 `rules/skills/workflow_obmc_env_init.md` 改为 `rules/skills/workflow_01-obmc_env_init.md`，将 `rules/skills/bestpractice_temporal_info_verification.md` 改为 `rules/skills/bestpractice_04-temporal_info_verification.md`

- [ ] Step 3: 确认无旧引用残留
- Run: `grep -n 'workflow_obmc_env_init\|bestpractice_temporal_info_verification' rules/skills/bestpractice_01-skill_writing.md`
- Expected: 无输出（零匹配）

### Task 5: 更新 AGENTS.md 中的引用

- 目标：将主入口文件中的 skill 写作指南路径更新为新文件名
- 涉及文件：`AGENTS.md`
- 验证范围：文件中无旧文件名引用

- [ ] Step 1: 确认当前内容包含旧引用
- Run: `grep -n 'bestpractice_skill_writing' AGENTS.md`
- Expected: 命中第 27 行

- [ ] Step 2: 更新引用路径
- Change: 将第 27 行的 `rules/skills/bestpractice_skill_writing.md` 改为 `rules/skills/bestpractice_01-skill_writing.md`

- [ ] Step 3: 确认无旧引用残留
- Run: `grep -n 'bestpractice_skill_writing' AGENTS.md`
- Expected: 无输出（零匹配）

### Task 6: 更新 bestpractice_03-ai_debugging_diagnosis.md 中的交叉引用

- 目标：更新 skill 文件间的交叉引用路径
- 涉及文件：`rules/skills/bestpractice_03-ai_debugging_diagnosis.md`（重命名后）
- 验证范围：文件中无旧文件名引用

- [ ] Step 1: 确认当前内容包含旧引用
- Run: `grep -n 'bestpractice_ai_programming_mindset' rules/skills/bestpractice_03-ai_debugging_diagnosis.md`
- Expected: 命中第 108 行

- [ ] Step 2: 更新引用路径
- Change: 将第 108 行的 `bestpractice_ai_programming_mindset.md` 改为 `bestpractice_02-ai_programming_mindset.md`

- [ ] Step 3: 确认无旧引用残留
- Run: `grep -n 'bestpractice_ai_programming_mindset' rules/skills/bestpractice_03-ai_debugging_diagnosis.md`
- Expected: 无输出（零匹配）

### Task 7: 更新 bestpractice_02-ai_programming_mindset.md 中的交叉引用

- 目标：更新 skill 文件间的交叉引用路径
- 涉及文件：`rules/skills/bestpractice_02-ai_programming_mindset.md`（重命名后）
- 验证范围：文件中无旧文件名引用

- [ ] Step 1: 确认当前内容包含旧引用
- Run: `grep -n 'bestpractice_temporal_info_verification' rules/skills/bestpractice_02-ai_programming_mindset.md`
- Expected: 命中第 117 行

- [ ] Step 2: 更新引用路径
- Change: 将第 117 行的 `bestpractice_temporal_info_verification.md` 改为 `bestpractice_04-temporal_info_verification.md`

- [ ] Step 3: 确认无旧引用残留
- Run: `grep -n 'bestpractice_temporal_info_verification' rules/skills/bestpractice_02-ai_programming_mindset.md`
- Expected: 无输出（零匹配）

### Task 8: 更新 KNOWLEDGE_BASE.md 中的引用

- 目标：将 heartbeat 知识库中的 skill 写作指南路径更新为新文件名
- 涉及文件：`periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- 验证范围：文件中无旧文件名引用

- [ ] Step 1: 确认当前内容包含旧引用
- Run: `grep -n 'bestpractice_skill_writing' periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- Expected: 命中第 74 行

- [ ] Step 2: 更新引用路径
- Change: 将第 74 行的 `rules/skills/bestpractice_skill_writing.md` 改为 `rules/skills/bestpractice_01-skill_writing.md`

- [ ] Step 3: 确认无旧引用残留
- Run: `grep -n 'bestpractice_skill_writing' periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`
- Expected: 无输出（零匹配）

### Task 9: Checkpoint commit

- 目标：提交所有重命名和引用更新
- 涉及文件：本次改动的全部文件
- 验证范围：commit 成功

- [ ] Step 1: 查看变更摘要
- Run: `git status --short`
- Expected: 5 个 rename + 6 个 modified

- [ ] Step 2: 提交
- Run: `git add -A && git commit -m "refactor(skills): 统一命名规范为 <category>_<NN>-<topic>.md"`
- Expected: commit 成功

---

## 执行纪律

- 开始实现前，先复查整份计划；如果发现缺项、矛盾、命名不一致或验证命令无效，先修计划
- 按任务顺序执行，不要无声跳步、合并步或改变任务目标
- 每完成一个任务，都运行该任务定义的验证
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下来说明，不要猜
- 全部任务完成后，运行最终验证并输出修改摘要

## 最终验证

- Run: `ls rules/skills/`
- Expected: 输出按类型聚簇、类型内按序号排列：`bestpractice_01-*`、`bestpractice_02-*`、`bestpractice_03-*`、`bestpractice_04-*`、`workflow_01-*`

- Run: `grep -rn 'bestpractice_ai_\|bestpractice_skill_writing\|bestpractice_temporal_info\|workflow_obmc_env' rules/ AGENTS.md periodic_jobs/`
- Expected: 无输出（所有活跃文件中的旧文件名引用已清零）
