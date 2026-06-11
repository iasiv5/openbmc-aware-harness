# ob init "Previously initialized" 分割栏 + 时间显示统一转 UTC+8 实施计划

## 目标

1. 在 `ob init` 列出所有可用 machine 后、用户输入 prompt 前，插入分割栏显示曾经 init 过的 machine（序号+名称+init 时间），帮用户快速定位老 machine 的序号。
2. 将 `ob` 脚本中所有时间显示从 UTC 统一转为 UTC+8（存储层不改）。

## 架构快照

- 新增一个 `format_timestamp()` 函数，统一将 ISO 8601 UTC 字符串转为 `YYYY-MM-DD HH:MM UTC+8` 格式。所有显示点都调它。
- 新增一个 `print_previously_initialized()` 函数，负责发现 `.init-done` machine、匹配序号、输出分割栏。
- `resolve_machine()` 中插入 `print_previously_initialized()` 调用，位置在编号列表输出后、prompt 前。
- 快速路径（命令行指定 machine）也调用分割栏。

## 输入工件

- 设计决策来自 grill-with-docs 会话，13 条决策全部锁定，无设计文档

## 文件结构与职责

- Modify: `ob`（单文件，以下用符号锚点定位）
  - `format_timestamp()` — 新增函数，放在通用 helper 区域（`trim_whitespace` 之后）
  - `print_previously_initialized()` — 新增函数，放在 `resolve_machine` 之前
  - `status_section_main_repo` → `first_init` 格式化（约 L1115）
  - `status_section_machines` → `init_time` 格式化（约 L1215）
  - `status_section_machines` → `build_time` 格式化（约 L1226）
  - `cmd_build` → `init_times` 数组填充（约 L1352-1355）
  - `resolve_machine` → 交互路径插入分割栏（编号列表之后、prompt 之前）
  - `resolve_machine` → 快速路径插入分割栏（`print_available_machines` 之后）

## 任务清单

### Task 1: 新增 `format_timestamp()` 函数

- 目标：提供统一的时间格式化能力，将 ISO 8601 UTC 转为 `YYYY-MM-DD HH:MM UTC+8`
- 涉及文件：`ob`
- 验证范围：函数存在、输入输出符合预期

- [ ] Step 1: 写当前状态检查
  - Run: `grep -n 'format_timestamp' /bmc/iasi/ob-harness/ob`
  - Expected: 无输出（函数不存在）

- [ ] Step 2: 确认失败
  - 上一步已确认函数不存在

- [ ] Step 3: 在 `trim_whitespace()` 函数之后（约 L69 之后）插入 `format_timestamp()`

  ```bash
  # Convert ISO 8601 UTC timestamp to local display format (UTC+8).
  # Input:  "2026-06-06T17:13:41Z" or empty/unparseable
  # Output: "2026-06-07 01:13 UTC+8" or "<unknown>"
  format_timestamp() {
      local raw="$1"

      if [[ -z "$raw" ]]; then
          echo "<unknown>"
          return 0
      fi

      # Strip trailing Z, parse with date, convert to UTC+8
      local clean="${raw%Z}"
      local converted
      converted=$(date -d "${clean}" '+%Y-%m-%dT%H:%M:%S' 2>/dev/null) || {
          echo "<unknown>"
          return 0
      }

      # Add 8 hours for UTC+8
      local ts_utc8
      ts_utc8=$(date -d "${converted} + 8 hours" '+%Y-%m-%d %H:%M UTC+8' 2>/dev/null) || {
          echo "<unknown>"
          return 0
      }

      echo "$ts_utc8"
  }
  ```

- Change: 新增 `format_timestamp()` 函数

- [ ] Step 4: 验证函数存在且逻辑正确
  - Run: `source <(sed -n '/^format_timestamp()/,/^}/p' /bmc/iasi/ob-harness/ob) && format_timestamp "2026-06-06T17:13:41Z"`
  - Expected: `2026-06-07 01:13 UTC+8`
  - Run: `source <(sed -n '/^format_timestamp()/,/^}/p' /bmc/iasi/ob-harness/ob) && format_timestamp ""`
  - Expected: `<unknown>`

### Task 2: 改造 `ob status` 中的 3 个时间显示点

- 目标：将 `status_section_main_repo` 的 `first_init`、`status_section_machines` 的 `init_time` 和 `build_time` 改为调用 `format_timestamp()`
- 涉及文件：`ob` 中 `status_section_main_repo`（约 L1109-1115）和 `status_section_machines`（约 L1210-1226）
- 验证范围：`ob status` 输出中时间格式为 `YYYY-MM-DD HH:MM UTC+8`

- [ ] Step 1: 确认当前时间格式为 UTC
  - Run: `grep -n 'UTC' /bmc/iasi/ob-harness/ob | grep -E '(1115|1215|1226)'`
  - Expected: 3 行匹配，输出包含 `UTC` 但不含 `UTC+8`

- [ ] Step 2: 确认当前状态
  - 上一步确认 3 处均使用 UTC 格式

- [ ] Step 3: 修改 `status_section_main_repo` 中 `first_init` 格式化（约 L1115）
  - 将:
    ```bash
    first_init=$(echo "$raw_time" | sed -E 's/T([0-9]{2}:[0-9]{2}):.*/ \1 UTC/' | sed 's/Z//')
    ```
  - 改为:
    ```bash
    first_init=$(format_timestamp "$raw_time")
    ```

- [ ] Step 4: 修改 `status_section_machines` 中 `init_time` 格式化（约 L1215）
  - 将:
    ```bash
    init_time=$(echo "$raw_init_time" | sed -E 's/T([0-9]{2}:[0-9]{2}):.*/ \1 UTC/' | sed 's/Z//')
    ```
  - 改为:
    ```bash
    init_time=$(format_timestamp "$raw_init_time")
    ```

- [ ] Step 5: 修改 `status_section_machines` 中 `build_time` 格式化（约 L1226）
  - 将:
    ```bash
    build_time=$(stat -c '%Y' "${m_image[$m]}" 2>/dev/null | xargs -I{} date -u -d @{} '+%Y-%m-%d %H:%M UTC' 2>/dev/null || echo "-")
    ```
  - 改为:
    ```bash
    build_time=$(stat -c '%Y' "${m_image[$m]}" 2>/dev/null | xargs -I{} date -d @{} '+%Y-%m-%dT%H:%M:%S' 2>/dev/null | xargs format_timestamp || echo "-")
    ```
  - 注意：`build_time` 来源是 Unix timestamp（`stat`），需要先转成 ISO 再交给 `format_timestamp`。也可直接用 `date -d @{} +8hours` 计算：
    ```bash
    build_time=$(stat -c '%Y' "${m_image[$m]}" 2>/dev/null | xargs -I{} date -d "@{} + 8 hours" '+%Y-%m-%d %H:%M UTC+8' 2>/dev/null || echo "-")
    ```

- Change: 3 处时间格式化代码替换为 `format_timestamp()` 调用（或等效 UTC+8 计算）

- [ ] Step 6: 验证
  - Run: `cd /bmc/iasi/ob-harness && bash ob status 2>/dev/null | grep -E '(First init|Init time|Build time)'`
  - Expected: 时间格式包含 `UTC+8`，不含单独的 `UTC`

### Task 3: 改造 `ob build` 中的 init 时间显示

- 目标：将 `cmd_build` 中直接显示原始 ISO 时间戳改为调用 `format_timestamp()`
- 涉及文件：`ob` 中 `cmd_build`（约 L1351-1355）
- 验证范围：`ob build` machine 列表中时间格式为 `YYYY-MM-DD HH:MM UTC+8`

- [ ] Step 1: 确认当前显示原始 ISO
  - Run: `grep -n 'init_times\+=' /bmc/iasi/ob-harness/ob | head -5`
  - Expected: 约 L1355，直接赋值原始 ISO 字符串

- [ ] Step 2: 确认当前状态
  - 上一步确认 init_times 未格式化

- [ ] Step 3: 修改 `cmd_build` 中 `init_times` 填充（约 L1351-1355）
  - 将:
    ```bash
    local init_time
    if ! IFS= read -r init_time < "$init_done_file"; then
        init_time="<unknown>"
    fi
    init_times+=("$init_time")
    ```
  - 改为:
    ```bash
    local init_time raw_init_time
    if ! IFS= read -r raw_init_time < "$init_done_file"; then
        raw_init_time=""
    fi
    init_time=$(format_timestamp "$raw_init_time")
    init_times+=("$init_time")
    ```

- Change: `init_times` 数组填充改为先格式化再存入

- [ ] Step 4: 验证
  - Run: `cd /bmc/iasi/ob-harness && bash ob build` （交互环境，有 init-done machine 时观察列表）
  - Expected: machine 列表中时间格式包含 `UTC+8`

### Task 4: 新增 `print_previously_initialized()` 函数

- 目标：实现分割栏逻辑——发现 `.init-done` machine，匹配编号，输出黄色分割栏
- 涉及文件：`ob`
- 验证范围：函数存在、输出格式正确

- [ ] Step 1: 写当前状态检查
  - Run: `grep -n 'print_previously_initialized' /bmc/iasi/ob-harness/ob`
  - Expected: 无输出（函数不存在）

- [ ] Step 2: 确认失败
  - 上一步确认函数不存在

- [ ] Step 3: 在 `resolve_machine()` 函数之前插入 `print_previously_initialized()`

  ```bash
  # Print "Previously initialized" separator showing init-done machines
  # with their original index from the full machine list.
  # Args:
  #   $1: machine name array (nameref)
  #   $2: CONFIGS_DIR path
  print_previously_initialized() {
      local -n _mpi_arr="$1"
      local _mpi_configs_dir="$2"

      # Discover init-done machines
      local -A _mpi_done=()
      local _mpi_f _mpi_mname _mpi_raw_time _mpi_fmt_time
      for _mpi_f in "$_mpi_configs_dir"/*.init-done; do
          [[ -f "$_mpi_f" ]] || continue
          _mpi_mname=$(basename "$_mpi_f" .init-done)
          _mpi_raw_time=$(head -1 "$_mpi_f" 2>/dev/null || true)
          _mpi_fmt_time=$(format_timestamp "$_mpi_raw_time")
          _mpi_done["$_mpi_mname"]="$_mpi_fmt_time"
      done

      # No init-done machines → nothing to show
      if [[ ${#_mpi_done[@]} -eq 0 ]]; then
          return 0
      fi

      # Print separator
      echo ""
      echo -e "  ${YELLOW}─── Previously initialized ───${NC}"

      # Match init-done machines to their index in the full list
      local _mpi_i=0
      local _mpi_total=${#_mpi_arr[@]}
      local _mpi_idx_width=${#_mpi_total}
      for _mpi_m in "${_mpi_arr[@]}"; do
          _mpi_i=$((_mpi_i + 1))
          if [[ -n "${_mpi_done[$_mpi_m]:-}" ]]; then
              printf "  %${_mpi_idx_width}d) %-20s %s\n" "$_mpi_i" "$_mpi_m" "${_mpi_done[$_mpi_m]}"
          fi
      done
  }
  ```

- Change: 新增 `print_previously_initialized()` 函数

- [ ] Step 4: 验证函数存在
  - Run: `grep -n 'print_previously_initialized' /bmc/iasi/ob-harness/ob | head -3`
  - Expected: 至少 2 行（函数定义 + nameref 行）

### Task 5: 在 `resolve_machine()` 交互路径插入分割栏调用

- 目标：编号列表输出后、prompt 前，调用 `print_previously_initialized()`
- 涉及文件：`ob` 中 `resolve_machine()` 函数
- 验证范围：`ob init` 无参数运行时，prompt 前出现分割栏

- [ ] Step 1: 确认当前编号列表和 prompt 之间无分割栏
  - Run: `grep -n -A2 'column -c' /bmc/iasi/ob-harness/ob | grep -E '(column|read -r -p)'`
  - Expected: `column` 和 `read -r -p` 之间无分割栏

- [ ] Step 2: 确认当前状态
  - 上一步确认无分割栏

- [ ] Step 3: 在 `resolve_machine()` 中，编号列表输出（`}| column ...`）之后、`local selected` 之前，插入调用

  找到（约 L1876 之后）:
  ```bash
  } | column -c "$term_cols" 2>/dev/null

  local selected
  ```
  改为:
  ```bash
  } | column -c "$term_cols" 2>/dev/null

  print_previously_initialized machine_arr "$CONFIGS_DIR"

  local selected
  ```

- Change: 在编号列表和 prompt 之间插入分割栏调用

- [ ] Step 4: 验证
  - Run: `cd /bmc/iasi/ob-harness && echo "q" | bash ob init 2>&1 | grep -A5 'Previously initialized'`
  - Expected: 出现 `─── Previously initialized ───` 及下方 init-done machine 列表（如果有）

### Task 6: 在 `resolve_machine()` 快速路径插入分割栏调用

- 目标：命令行指定 machine 时，`print_available_machines` 之后也显示分割栏
- 涉及文件：`ob` 中 `resolve_machine()` 快速路径（约 L1847-1851）
- 验证范围：`ob init <已init的machine>` 输出包含分割栏

- [ ] Step 1: 确认快速路径无分割栏
  - Run: `sed -n '1847,1851p' /bmc/iasi/ob-harness/ob`
  - Expected: 只有 `print_available_machines` + `info "Machine ... confirmed"` + `return 0`

- [ ] Step 2: 确认当前状态
  - 上一步确认无分割栏

- [ ] Step 3: 修改快速路径

  找到（约 L1847-1851）:
  ```bash
  if [[ -n "$MACHINE" ]] && echo "$machines" | grep -qx -- "$MACHINE"; then
      print_available_machines
      info "Machine '$MACHINE' confirmed."
      return 0
  fi
  ```
  改为:
  ```bash
  if [[ -n "$MACHINE" ]] && echo "$machines" | grep -qx -- "$MACHINE"; then
      print_available_machines
      print_previously_initialized machine_arr "$CONFIGS_DIR"
      info "Machine '$MACHINE' confirmed."
      return 0
  fi
  ```

- Change: 在快速路径的 `print_available_machines` 后插入分割栏调用

- [ ] Step 4: 验证
  - Run: `cd /bmc/iasi/ob-harness && bash ob init romulus -d 2>&1 | grep -A5 'Previously initialized'`
  - Expected: 出现分割栏及 init-done machine 列表（前提：romulus 有 `.init-done` 文件）

## 执行纪律

- 开始实现前，先批判性复查整份计划；如果发现缺项、矛盾、命名不一致或验证命令无效，先修计划
- 按任务顺序执行，不要无声跳步、合并步或改变任务目标
- 每完成一个任务，都运行该任务定义的验证
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下来说明，不要猜
- 当前分支为 `feat/start-qemu`，不是 main，可以直接实现
- 全部任务完成后，运行最终验证并输出修改摘要

## 最终验证

```bash
# 1. format_timestamp 函数存在且可调用
bash -c 'source <(sed -n "/^format_timestamp()/,/^}/p" ob) && format_timestamp "2026-06-06T17:13:41Z"'
# Expected: 2026-06-07 01:13 UTC+8

# 2. format_timestamp 容错
bash -c 'source <(sed -n "/^format_timestamp()/,/^}/p" ob) && format_timestamp ""'
# Expected: <unknown>

# 3. ob status 时间格式
bash ob status 2>/dev/null | grep -E '(First init|Init time|Build time)'
# Expected: 所有时间显示包含 UTC+8

# 4. ob init 交互路径分割栏（需要 init-done machine 存在）
bash ob init 2>&1 | grep -A5 'Previously initialized'
# Expected: 出现黄色分割栏 + init-done machine 序号+名称+时间

# 5. ob init 快速路径分割栏（用 -d 避免 side effect）
bash ob init romulus -d 2>&1 | grep -A5 'Previously initialized'
# Expected: 同上

# 6. 无 init-done machine 时不显示分割栏（清空 configs 后测试，或新仓库场景）
# Expected: 输出中不含 "Previously initialized"

# 7. 语法检查
bash -n ob
# Expected: 无错误输出
```
