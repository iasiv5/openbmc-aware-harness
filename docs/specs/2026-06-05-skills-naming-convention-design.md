# Skills 文件命名规范设计文档

## 背景与目标

`rules/skills/` 目录下的 skill 文件当前使用 `<category>_<topic>.md` 命名，类型前缀（`bestpractice_` / `workflow_`）提供了按类型聚簇的 `ls` 排序，但同类型内的文件排序取决于 topic 字母序，没有实际含义。

本次目标：
1. 在保持类型聚簇的前提下，让同类型文件有一个明确的阅读/优先级排序
2. 趁文件数量少（5 个）时固化命名规范，避免后续增长时再做迁移

成功标准：
- `ls rules/skills/` 输出先按类型聚簇，类型内按人工定义的优先级排序
- 命名规范写入 `05_SKILLS_INDEX.md`，后续新增文件有明确的命名指引
- 所有内部引用（跨文件链接、索引条目）与新文件名一致

## 范围

- 重命名 `rules/skills/` 下全部 5 个文件
- 更新 `rules/05_SKILLS_INDEX.md` 中的命名规范说明和所有文件引用
- 更新 `rules/skills/bestpractice_skill_writing.md`（重命名后为 `bestpractice_01-skill_writing.md`）中的跨文件引用

## 非范围

- 不改变文件内容（引用路径除外）
- 不改变目录结构（不引入子目录）
- 不改变 `rules/03_WORKSPACE.md`（它只记录目录级路由）
- 不增加新的 skill 文件

## 方案比较

### 方案 A：现状微调

- 核心思路：保持 `<category>_<topic>.md`，文件内 header 加日期字段
- 优点：零迁移成本
- 缺点：同类型内排序无意义，文件名无法反映优先级

### 方案 B：日期前缀

- 核心思路：`<YYYY>-<category>_<topic>.md`
- 优点：按时间排序
- 缺点：破坏类型聚簇，对稳定参考文档意义不大

### 方案 C：类型前缀 + 类型内序号（推荐）

- 核心思路：`<category>_<NN>-<topic>.md`
- 优点：保留类型聚簇 + 类型内有明确排序
- 缺点：序号需人工分配（skill 增长慢，维护成本低）

## 推荐方案

方案 C。

理由：skill 文件增长速度慢（不是每天新增），序号维护成本极低；序号反映优先级判断，比字母序有实际意义。

主要 trade-off：序号是人工约定，插入中间文件需要调号。但 skill 低频增长，且追加到末尾（`05-`、`06-`…）是最常见场景，几乎不会遇到中间插入。

## 命名格式

```
<category>_<NN>-<topic>.md
```

| 部分 | 规则 | 示例 |
|------|------|------|
| `<category>` | 小写，现有 `bestpractice` / `workflow`，未来可扩展 | `bestpractice` |
| `<NN>` | 两位数字，同类型内按重要性/阅读优先级排列 | `01` |
| `-` | 序号与 topic 之间用短横线分隔 | — |
| `<topic>` | snake_case，描述性名称 | `skill_writing` |

完整示例：`bestpractice_01-skill_writing.md`

## 文件重命名映射

| 旧文件名 | 新文件名 | 序号理由 |
|---|---|---|
| `bestpractice_skill_writing.md` | `bestpractice_01-skill_writing.md` | Meta-skill，创建其他 skill 前先读 |
| `bestpractice_ai_programming_mindset.md` | `bestpractice_02-ai_programming_mindset.md` | 基础方法论，使用频率高 |
| `bestpractice_ai_debugging_diagnosis.md` | `bestpractice_03-ai_debugging_diagnosis.md` | 特定场景实践 |
| `bestpractice_temporal_info_verification.md` | `bestpractice_04-temporal_info_verification.md` | 特定场景实践 |
| `workflow_obmc_env_init.md` | `workflow_01-obmc_env_init.md` | 当前唯一的 workflow |

## 受影响文件与改动

### 1. 文件重命名（5 个文件）

`git mv` 执行上述映射表中的重命名。

### 2. `rules/05_SKILLS_INDEX.md`

改动点：
- 命名规范说明从 `<category>_<name>.md` 更新为 `<category>_<NN>-<name>.md`
- 分类索引中所有文件引用路径更新为新文件名

### 3. `rules/skills/bestpractice_01-skill_writing.md`（重命名后）

改动点：
- 末尾"与现有 skill 的关系"段落中两个引用路径更新：
  - `workflow_obmc_env_init.md` → `workflow_01-obmc_env_init.md`
  - `bestpractice_temporal_info_verification.md` → `bestpractice_04-temporal_info_verification.md`

### 4. `AGENTS.md`

改动点：
- 第 27 行 skill 写作指南引用路径：`rules/skills/bestpractice_skill_writing.md` → `rules/skills/bestpractice_01-skill_writing.md`

### 5. `rules/skills/bestpractice_03-ai_debugging_diagnosis.md`（重命名后）

改动点：
- 第 108 行交叉引用：`bestpractice_ai_programming_mindset.md` → `bestpractice_02-ai_programming_mindset.md`

### 6. `rules/skills/bestpractice_02-ai_programming_mindset.md`（重命名后）

改动点：
- 第 117 行交叉引用：`bestpractice_temporal_info_verification.md` → `bestpractice_04-temporal_info_verification.md`

### 7. `periodic_jobs/ai_heartbeat/docs/KNOWLEDGE_BASE.md`

改动点：
- 第 74 行 skill 写作指南引用路径：`rules/skills/bestpractice_skill_writing.md` → `rules/skills/bestpractice_01-skill_writing.md`

## 错误处理与回退

- 如果 `git mv` 前发现文件已被手动改名或不存在，中止并报告
- 重命名后通过 `git diff` 确认无遗漏引用（grep 旧文件名应返回零结果，覆盖范围：`rules/`、`AGENTS.md`、`periodic_jobs/`）

## 测试策略

- `ls rules/skills/` 验证排序：先 `bestpractice_*` 聚簇，再 `workflow_*` 聚簇，类型内按序号排列
- `grep -r "bestpractice_ai_" rules/ AGENTS.md periodic_jobs/` 等搜索旧文件名，确认无遗漏引用
- 人工审阅 `05_SKILLS_INDEX.md` 的链接是否指向正确文件

## 未决事项

无。
