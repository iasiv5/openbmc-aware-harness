# `ob status` 总览面板重构设计

> **状态**: 已批准 (2026-06-08)
> **动机**: 当前 `ob status` 只读 `openbmc-source.lock` 并比对 origin，功能单一，用户无法一眼了解整个 workspace 的状态。workspace 下已沉淀大量结构化数据（per-machine lock、init-done、build 产物）但未被展示。

## 设计决策

### 核心定位

`ob status` 是**信息总览面板**，不是校验器。用户场景："回到工作岗位，快速看一眼当前什么情况"。

### 输出结构

整个面板分三个 Section，按顺序输出：

```
────────────────────────────────────────────────────────────
  OpenBMC Main Repository
────────────────────────────────────────────────────────────
  Status       : present
  Source       : git@github.com:openbmc/openbmc.git (community)
  Local path   : /home/iasi/ob-harness/workspace/openbmc
  Branch       : master
  Commit       : 2d39837 phosphor-logging: srcrev bump...
  Upstream     : ✅ up-to-date
  First init   : 2026-06-06 17:13 UTC

────────────────────────────────────────────────────────────
  Machines
────────────────────────────────────────────────────────────
  Machine            Init      Repos   Build
  gb200nvl-obmc      ✅ done    100    🔨 succeeded
  romulus             ✅ done    110    — never

  ── gb200nvl-obmc ─────────────────────────────────
    Init time    : 2026-06-08 03:36 UTC
    OB commit    : 2d39837 phosphor-logging: srcrev bump...
    Repos        : 100
    Build        : 🔨 succeeded
    Image        : .../obmc-phosphor-image-gb200nvl-obmc.static.mtd

  ── romulus ───────────────────────────────────────
    Init time    : 2026-06-08 03:39 UTC
    OB commit    : 2d39837 phosphor-logging: srcrev bump...
    Repos        : 110
    Build        : — never built

  💡 Run 'ob build' to build a machine.
```

### Section 1：主仓信息（key-value 格式）

| 字段 | 数据来源 | 备注 |
|---|---|---|
| Status | `workspace/openbmc/.git/` 是否存在 | `present` / `missing` |
| Source | `git remote get-url origin` + `source.lock` 的 `source_label` | 括号内显示 label |
| Local path | 固定路径 `workspace/openbmc` | |
| Branch | `git rev-parse --abbrev-ref HEAD` | |
| Commit | `git log --oneline -1` | 短 hash + subject |
| Upstream | `git fetch origin` → 本地 HEAD vs `origin/HEAD` | 网络不通时显示 `⚠️ unreachable (skipped)` |
| First init | `openbmc-source.lock` 的 `created_at` | |

**Upstream 比对规则：**
- 需要 `git fetch origin`，只针对主仓一个仓库，耗时可控
- 比对本地 HEAD 与 `origin/HEAD` 的 commit 数：`up-to-date` / `behind N` / `ahead N`
- 网络超时或不可达：不阻塞，显示 `⚠️ unreachable (skipped)` 后继续输出其余字段

### Section 2：Machine 总览 + 逐个展开

**总览表（每个 machine 一行）：**

| 列 | 数据来源 | 值 |
|---|---|---|
| Machine | `configs/*.init-done` 文件名 | machine 名称 |
| Init | `.init-done` 是否存在 | `✅ done` / `⏳ partial` / `—` |
| Repos | `<machine>.lock` 的 `sub_repos` 数量 | 数字 |
| Build | `build/<machine>/tmp/deploy/images/<machine>/` 下有无 image | `🔨 succeeded` / `❌ failed` / `— never` |

**逐 machine 展开（key-value 缩进块）：**

| 字段 | 数据来源 |
|---|---|
| Init time | `.init-done` 文件第一行（UTC 时间戳） |
| Repos | `<machine>.lock` 的 `sub_repos` 数量 |
| Build | 检测 `tmp/deploy/images/` 下有无 `.static.mtd` image 文件 |
| Image | image 文件完整路径（仅 build 成功时显示） |

> **注**：不显示 per-machine OB commit。因为所有 machine 的 `openbmc_commit` 通常相同，且主仓 Section 1 的 `Commit` 字段已覆盖此信息。

**Build 状态判定逻辑：**
1. 检查 `build/<machine>/tmp/deploy/images/<machine>/obmc-phosphor-image-<machine>.static.mtd` 是否存在
2. 存在 → `🔨 succeeded`，显示 Image 路径
3. `build/<machine>/` 目录存在但无 image → `❌ failed`（或 `⏳ partial`）
4. `build/<machine>/` 不存在 → `— never`

### Section 3：Tips（动态提示）

根据上下文显示 1-2 行提示，不做 tips wall：

| 条件 | 提示 |
|---|---|
| 主仓 missing | `💡 Run 'ob init' to get started.` |
| 有 init-done machine 但从未 build | `💡 Run 'ob build' to build a machine.` |
| 一切正常 | 不显示 |

### Edge cases

| 场景 | 行为 |
|---|---|
| 主仓 missing | 只显示 Section 1（Status: missing）+ Section 3 Tips，跳过 Section 2 |
| 网络不通 | Upstream 字段显示 `⚠️ unreachable (skipped)`，不阻塞 |
| 无 machine | Section 2 显示 `(none)`，Tips 提示 `ob init` |

### 性能约束

- **唯一网络操作**：主仓 `git fetch origin`（单仓库，通常 < 5s）
- **不做**：`du -sh`（编译目录太大太慢）、子仓库版本比对（bitbake 自管）
- **全部本地操作**：读 source.lock、per-machine .lock、.init-done、git config、deploy 目录探测

### 不做的事

- ❌ 不加 source drift 检测（`ob init` 入口已硬拦）
- ❌ 不加磁盘占用（`du -sh` 在编译目录上太慢）
- ❌ 不加 `ob status <machine>` 子命令（保持简单）
- ❌ 不改 `openbmc-source.lock` 文件格式（status 直接读 live 数据，不依赖 lock 存全量）
