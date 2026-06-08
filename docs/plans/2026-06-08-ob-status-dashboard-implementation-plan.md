# `ob status` 总览面板重构实施计划

## 目标

将 `cmd_status()` 从"读 source.lock 并比对 origin"的单一功能，重构为三段式总览面板：主仓信息 → Machine 列表 + 展开 → 动态 Tips。

## 架构快照

- **思路**：将现有的单体 `cmd_status()` 拆成三个独立的输出函数（`status_section_main_repo`、`status_section_machines`、`status_section_tips`），由新的 `cmd_status()` 依次调用。
- **衔接**：现有 helper（`read_lock_field`、`step_header`、`info`、`warn`、`verbose`）全部复用，不改动。新增函数放在 `cmd_status()` 上方，与现有 `cmd_build`/`cmd_init` 同级。
- **唯一网络操作**：主仓 `git fetch origin`（带超时），网络不通时降级为提示，不阻塞。

## 输入工件

- 设计文档：`docs/specs/2026-06-08-ob-status-dashboard-design.md`
- 当前实现：`ob` 文件 `cmd_status()` 函数（第 569–638 行）

## 文件结构与职责

- Modify: `ob` — 重写 `cmd_status()`，新增 `status_section_main_repo`、`status_section_machines`、`status_section_tips` 三个函数

## 任务清单

### Task 1: 新增 `status_section_main_repo` 函数

- 目标：实现 Section 1（主仓信息块），输出 Status / Source / Local path / Branch / Commit / Upstream / First init
- 涉及文件：`ob`（在 `cmd_status()` 上方插入新函数）
- 验证范围：在已有主仓的环境中运行 `./ob status`，确认 Section 1 输出包含全部 7 个字段

- [ ] Step 1: 确认当前 `cmd_status` 输出（作为改动前基线）
  - Run: `cd /home/iasi/ob-harness && ./ob status`
  - Expected: 输出旧格式（"OpenBMC main repository binding status" + 绑定信息），尚无 Section 1 的 key-value 块

- [ ] Step 2: 在 `cmd_status()` 上方新增 `status_section_main_repo()` 函数

  函数职责：
  1. 检测 `OPENBMC_DIR/.git` 是否存在 → 输出 `Status: present` 或 `Status: missing`
  2. 若主仓存在：
     - `git -C "$OPENBMC_DIR" remote get-url origin` → Source
     - `read_lock_field source_label` → 括号内 label
     - 输出 `Local path: $OPENBMC_DIR`
     - `git -C "$OPENBMC_DIR" rev-parse --abbrev-ref HEAD` → Branch
     - `git -C "$OPENBMC_DIR" log --oneline -1` → Commit
     - `git -C "$OPENBMC_DIR" fetch origin --quiet` → 比对 HEAD vs `origin/HEAD`，用 `git rev-list --left-right --count` 计算 ahead/behind；fetch 失败时输出 `⚠️ unreachable (skipped)`
     - `read_lock_field created_at` → First init（格式化为 `YYYY-MM-DD HH:MM UTC`）
  3. 若主仓 missing：只输出 `Status: missing`，其余字段不输出

  输出格式：
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
  ```

  使用现有 `step_header "OpenBMC Main Repository"` 作为标题分隔线。

  git fetch 超时策略：使用 `timeout 10 git -C "$OPENBMC_DIR" fetch origin --quiet 2>/dev/null`，超时或失败时 Upstream 显示 `⚠️ unreachable (skipped)`。

- Change: 在 `ob` 文件中 `cmd_status()` 上方新增 `status_section_main_repo()` 函数

- [ ] Step 3: 验证新函数输出
  - Run: `cd /home/iasi/ob-harness && ./ob status`
  - Expected: 此时 cmd_status 还没改，输出不变。需要手动在 bash 中 source 并调用 `status_section_main_repo` 来验证，或者等 Task 4 完成 cmd_status 改写后统一验证。

### Task 2: 新增 `status_section_machines` 函数

- 目标：实现 Section 2（Machine 总览表 + 逐 machine 展开）
- 涉及文件：`ob`（在 `status_section_main_repo` 下方插入新函数）
- 验证范围：运行 `./ob status`，确认输出包含总览表和每个 machine 的展开块

- [ ] Step 1: 确认现有 configs 目录下有哪些 machine 数据文件
  - Run: `ls /home/iasi/ob-harness/workspace/configs/*.init-done /home/iasi/ob-harness/workspace/configs/*.lock 2>/dev/null`
  - Expected: 列出 `gb200nvl-obmc.init-done`、`gb200nvl-obmc.lock`、`romulus.init-done`、`romulus.lock` 等文件

- [ ] Step 2: 新增 `status_section_machines()` 函数

  函数职责：

  **a) 发现 machine 列表：**
  - 遍历 `$CONFIGS_DIR/*.init-done`，从文件名提取 machine 名称
  - 同时遍历 `$CONFIGS_DIR/*.lock`（有 .lock 但无 .init-done = init 中断 / partial）
  - 合并去重，保持排序

  **b) 总览表输出：**
  ```
  ────────────────────────────────────────────────────────────
    Machines
  ────────────────────────────────────────────────────────────
    Machine            Init      Repos   Build
    gb200nvl-obmc      ✅ done    100    🔨 succeeded
    romulus             ✅ done    110    — never
  ```
  - Init 列：有 `.init-done` → `✅ done`；有 `.lock` 无 `.init-done` → `⏳ partial`；都无 → `—`
  - Repos 列：从 `<machine>.lock` 读 `sub_repos` 数组长度（`python3 -c "import json; print(len(json.load(open(...))['sub_repos']))"`）
  - Build 列：检测 `$OPENBMC_DIR/build/<machine>/tmp/deploy/images/<machine>/obmc-phosphor-image-<machine>.static.mtd` 是否存在
    - 存在 → `🔨 succeeded`
    - `$OPENBMC_DIR/build/<machine>` 存在但无 image → `❌ failed`
    - 不存在 → `— never`

  **c) 逐 machine 展开块：**
  ```
`
    ── gb200nvl-obmc ─────────────────────────────────
      Init time    : 2026-06-08 03:36 UTC
      OB commit    : 2d39837 phosphor-logging: srcrev bump...
      Repos        : 100
      Build        : 🔨 succeeded
      Image        : .../obmc-phosphor-image-gb200nvl-obmc.static.mtd
  ```
  - Init time：读 `.init-done` 第一行，格式化为 `YYYY-MM-DD HH:MM UTC`
  - OB commit：读 `<machine>.lock` 的 `openbmc_commit`，用 `git -C "$OPENBMC_DIR" log --oneline -1 <commit>` 获取短 hash + subject
  - Repos：同总览表
  - Build + Image：同总览表逻辑，succeeded 时额外显示 Image 完整路径
  - 无 `.lock` 的 machine（仅有部分数据）跳过展开块

  **d) 无 machine 时的处理：**
  ```
  ────────────────────────────────────────────────────────────
    Machines
  ────────────────────────────────────────────────────────────
    (none)
  ```

- Change: 在 `ob` 文件中 `status_section_main_repo` 下方新增 `status_section_machines()` 函数

- [ ] Step 3: 验证
  - 同 Task 1 Step 3，等 Task 4 统一验证

### Task 3: 新增 `status_section_tips` 函数

- 目标：实现 Section 3（动态 Tips），根据上下文输出 1-2 行提示
- 涉及文件：`ob`（在 `status_section_machines` 下方插入新函数）
- 验证范围：Tips 根据状态正确显示/隐藏

- [ ] Step 1: 新增 `status_section_tips()` 函数

  函数签名：`status_section_tips(repo_exists has_init_done_machines has_never_built_machines)`

  参数说明（全部为 `0`/`1`）：
  - `$1` `repo_exists`：主仓是否存在
  - `$2` `has_init_done_machines`：是否有 init-done 的 machine
  - `$3` `has_never_built_machines`：是否有 init-done 但从未 build 的 machine

  逻辑：
  ```
  if repo_missing → "💡 Run 'ob init' to get started."
  elif has_never_built → "💡 Run 'ob build' to build a machine."
  elif no_machines → "💡 Run 'ob init' to initialize a machine."
  else → 不输出
  ```

  每条 tip 最多输出一行，不做 tips wall。

- Change: 在 `ob` 文件中新增 `status_section_tips()` 函数

### Task 4: 重写 `cmd_status` 编排三段输出

- 目标：替换现有 `cmd_status()` 为新版本，依次调用三个 section 函数，处理 edge case
- 涉及文件：`ob`（替换 `cmd_status()` 函数，第 569–638 行）
- 验证范围：`./ob status` 输出完整三段式面板

- [ ] Step 1: 记录旧版输出作为对比基线
  - Run: `cd /home/iasi/ob-harness && ./ob status`
  - Expected: 旧版输出

- [ ] Step 2: 替换 `cmd_status()` 实现

  新版 `cmd_status()` 逻辑：
  ```bash
  cmd_status() {
      local repo_exists=0
      [[ -d "$OPENBMC_DIR/.git" ]] && repo_exists=1

      # Section 1: 主仓信息（始终显示）
      status_section_main_repo

      echo ""

      # Section 2: Machine 列表（主仓 missing 时也显示，可能是残留数据）
      status_section_machines

      # Section 3: Tips（动态）
      local has_init_done=0
      local has_never_built=0
      for f in "$CONFIGS_DIR"/*.init-done; do
          [[ -f "$f" ]] || continue
          has_init_done=1
          local mname
          mname=$(basename "$f" .init-done)
          local image_path="$OPENBMC_DIR/build/$mname/tmp/deploy/images/$mname/obmc-phosphor-image-$mname.static.mtd"
          if [[ ! -f "$image_path" ]]; then
              has_never_built=1
          fi
          break  # 只需要检测是否存在至少一个
      done

      status_section_tips "$repo_exists" "$has_init_done" "$has_never_built"
  }
  ```

  edge case 处理：
  - 主仓 missing：Section 1 只显示 `Status: missing`；Section 2 仍尝试显示（可能有残留的 .lock/.init-done）；Section 3 提示 `ob init`
  - 无 machine：Section 2 显示 `(none)`；Section 3 提示 `ob init`
  - 网络不通：Section 1 的 Upstream 字段显示 `⚠️ unreachable (skipped)`，其余字段正常

- Change: 替换 `ob` 文件第 569–638 行的 `cmd_status()` 函数

- [ ] Step 3: 运行并验证完整输出
  - Run: `cd /home/iasi/ob-harness && ./ob status`
  - Expected: 输出包含三段式面板（主仓信息块 + Machine 总览表 + 展开 + Tips），格式与设计文档 mockup 一致

- [ ] Step 4: 验证 `-v` 模式
  - Run: `cd /home/iasi/ob-harness && ./ob status -v`
  - Expected: 正常执行，verbose 模式不报错（`parse_args` 已处理 `-v`，`detect_harness_root` 正常设置路径）

- [ ] Step 5: Checkpoint commit
  - Run: `git add ob && git commit -m "feat(ob status): 重构为三段式总览面板

  替换旧版 cmd_status (仅读 source.lock 比对 origin)
  为三段式总览面板:
  - Section 1: 主仓信息 (status/source/branch/commit/upstream/init-time)
  - Section 2: Machine 总览表 + 逐 machine 展开 (init/repos/build/image)
  - Section 3: 动态 Tips

  设计文档: docs/specs/2026-06-08-ob-status-dashboard-design.md"`
  - Expected: commit 成功

## 执行纪律

- 开始实现前先复查整份计划；发现缺项、矛盾或验证命令无效时先修计划
- 按任务顺序执行，不跳步、不合并、不改任务目标
- 每完成一个任务都运行该任务的验证
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下说明，不猜
- 当前分支 `feat/ob-interactive-menu`，不需要切换
- 全部任务完成后运行最终验证并输出修改摘要

## 最终验证

- Run: `cd /home/iasi/ob-harness && ./ob status`
- Expected:
  - Section 1 显示 7 个字段（Status / Source / Local path / Branch / Commit / Upstream / First init）
  - Section 2 显示总览表（每个 machine 一行：Machine / Init / Repos / Build）
  - Section 2 显示每个 machine 的展开块（Init time / OB commit / Repos / Build / Image）
  - Section 3 根据状态显示 0-1 行 Tips
  - 整体格式与设计文档 mockup 一致
- Run: `cd /home/iasi/ob-harness && bash -n ob`
- Expected: 语法检查通过，无错误输出
