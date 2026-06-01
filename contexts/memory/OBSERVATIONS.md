# Memory Observations

这是三层记忆系统的动态记忆日志。observer 会把当天观测追加到这里；reflector 会回看这里的近期内容，清理低价值项，并据此产出规则晋升与报告。默认触发方式是 `.github/hooks/pre-session.ps1` 调用 `periodic_jobs/ai_heartbeat/src/v0/heartbeat_local_runner.py`。

## 格式说明

每个日期条目格式如下：

```raw
Date: YYYY-MM-DD

🔴 High: [方法论/约束] 描述
🟡 Medium: [项目状态/决策] 描述
🟢 Low: [任务流水] 描述
```

### 优先级定义

- **🔴 High**：跨项目通用的经验教训、硬性约束、影响系统架构的重大决策。永久保留，候选晋升为 axiom 或 skill。
- **🟡 Medium**：活跃项目的关键进展、技术决策背景、未来几周仍需参考的信息。
- **🟢 Low**：日常任务流水、瞬时 debug 记录、临时上下文。定期垃圾回收。

## 如何加载记忆

不要全文加载这个文件（可能很大）。按需检索：

```bash
# 搜索特定主题
grep -n "关键词" contexts/memory/OBSERVATIONS.md

# 搜索最近 N 天
grep -A 20 "Date: $(date -v-7d +%Y-%m-%d)" contexts/memory/OBSERVATIONS.md
```

或使用 `grep_search`（正则搜关键词）或 `semantic_search`（语义搜意图）做跨日期检索。

---

<!-- 以下是记录区域，由 AI Heartbeat 本地执行器追加与整理 -->

Date: 2026-06-01

🟡 Medium: [ob init 工具状态] `ob init <machine>` 的设计、计划和主脚本已经落地到 `docs/specs/2026-05-31-obmc-env-init-design.md`、`docs/plans/2026-05-31-obmc-env-init-implementation-plan.md`、`tools/ob` 与 `tools/parse_bitbake_deps.py`；当前实现覆盖主仓库准备、OpenBMC `source setup`、`bitbake -g`、逐 recipe `bitbake -e` 解析、子仓库 clone/fetch、lockfile 与 externalsrc 注入。
🟡 Medium: [AI Heartbeat 执行边界] `periodic_jobs/ai_heartbeat/src/v0/heartbeat_preflight.py`、`heartbeat_state.py` 和 `heartbeat_status_cli.py` 已形成 due-task 判定与自动记账链；hook 只提醒，observer/reflector 必须由当前 chat 显式运行 `/ai-heartbeat` 并在完成后写回 success/skipped/failed。
