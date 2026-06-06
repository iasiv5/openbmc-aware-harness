# DL_DIR/git2/ Bare Mirror 缓存机制 实施计划

## 目标

在 `ob init` 的 `clone_sub_repos()` 中，先将每个 git repo 克隆为 bare mirror 到 `DL_DIR/git2/`，再从 bare mirror 本地克隆 working tree 到 `src/<machine>/`。同时移除已废弃的 `OB_GIT_REFERENCE_DIR` 机制。

## 架构快照

本次改造涉及 `ob` 脚本中的四个函数和一个新函数：

```
新增: resolve_effective_dl_dir()
  → 替代原 resolve_git_reference_root() 的职责
  → 直接从 local.conf 读 DL_DIR，不依赖中间变量

保留: derive_bitbake_git_mirror_path(base_dir, src_uri)
  → 函数体不变，但调用方传入 <effective_dl_dir>/git2/ 而非 reference_root

改造: clone_sub_repos()
  → 每个 repo 的流程变为: bare mirror → working tree → checkout srcrev
  → 旧数据兼容：working tree 存在但 mirror 不存在时，从远程补建 mirror

清理: ensure_bootstrap_local_conf()
  → 移除 Python 代码中 OB_GIT_REFERENCE_DIR 读写逻辑

清理: resolve_git_reference_root()
  → 整体移除

增强: print_report()
  → 新增 mirror 统计行
```

## 输入工件

- 设计文档：`docs/specs/2026-06-06-dl-dir-git2-mirror-design.md`

## 文件结构与职责

- Modify: `ob` — 唯一需要改动的文件
  - `resolve_effective_dl_dir()` — 新增，计算有效 DL_DIR（读 local.conf → 默认值 → 可写检查 → fallback）
  - `resolve_git_reference_root()` — 移除整个函数
  - `ensure_bootstrap_local_conf()` — 移除 OB_GIT_REFERENCE_DIR 相关 Python 逻辑和 bash 日志输出
  - `clone_sub_repos()` — 核心改造，加入 bare mirror 阶段
  - `print_report()` — 新增 mirror 统计
  - `derive_bitbake_git_mirror_path()` — 函数体不变

## 任务清单

### Task 1: 新增 `resolve_effective_dl_dir()` 函数

- 目标：提供一个函数，返回当前 build 环境的有效 DL_DIR 绝对路径
- 涉及文件：`ob`
- 验证范围：函数能正确从 local.conf 读取 DL_DIR；local.conf 无 DL_DIR 时返回 harness 默认值；路径不可写时 fallback

- [ ] Step 1: 确认当前 `read_local_conf_var()` 可以正确读取 `DL_DIR`

  `read_local_conf_var()` 已存在于 `ob` 脚本（`read_local_conf_var` 函数），可直接复用。

- Run: `grep -A5 'read_local_conf_var()' /bmc/iasi/ob-harness-community/ob | head -10`
- Expected: 函数定义可见，接受 local_conf 路径和变量名两个参数

- [ ] Step 2: 在 `resolve_git_reference_root()` 函数之后（约第 176 行之后）插入新函数

  插入位置：`ob` 中 `derive_bitbake_git_mirror_path()` 函数定义之前（第 177 行），紧跟 `resolve_git_reference_root()` 之后。

  ```bash
  resolve_effective_dl_dir() {
      local local_conf="$BUILD_DIR/conf/local.conf"
      local default_dl_dir="$WORKSPACE_DIR/downloads"
      local dl_dir=""

      # 1. 尝试从 local.conf 读取 DL_DIR
      dl_dir=$(read_local_conf_var "$local_conf" "DL_DIR" 2>/dev/null || true)
      dl_dir=$(trim_whitespace "$dl_dir")

      # 2. 未找到则使用 harness 默认值
      if [[ -z "$dl_dir" ]]; then
          dl_dir="$default_dl_dir"
      fi

      # 3. 可写检查：尝试创建临时文件
      mkdir -p "$dl_dir" 2>/dev/null
      if ! touch "$dl_dir/.ob-init-writable-test" 2>/dev/null; then
          warn "DL_DIR not writable: $dl_dir — falling back to $default_dl_dir"
          dl_dir="$default_dl_dir"
          mkdir -p "$dl_dir"
      else
          rm -f "$dl_dir/.ob-init-writable-test"
      fi

      echo "$dl_dir"
  }
  ```

- Change: 在 `ob` 第 177 行处插入新函数 `resolve_effective_dl_dir()`

- [ ] Step 3: 验证新函数可被调用

- Run: `grep -n 'resolve_effective_dl_dir' /bmc/iasi/ob-harness-community/ob`
- Expected: 函数定义行和新插入位置可见

- [ ] Step 4: 可选 checkpoint commit

- Run: `git add ob && git commit -m "feat(ob): add resolve_effective_dl_dir() function"`

---

### Task 2: 移除 `resolve_git_reference_root()` 和 `ensure_bootstrap_local_conf()` 中的 OB_GIT_REFERENCE_DIR 逻辑

- 目标：清除所有 OB_GIT_REFERENCE_DIR 相关代码，为后续 `clone_sub_repos()` 改造扫清障碍
- 涉及文件：`ob` 中两个位置
- 验证范围：`ob` 脚本中不再包含 `OB_GIT_REFERENCE_DIR` 字符串；`ob init --dry-run` 不报错

- [ ] Step 1: 确认所有 OB_GIT_REFERENCE_DIR 引用点

- Run: `grep -n 'OB_GIT_REFERENCE_DIR' /bmc/iasi/ob-harness-community/ob`
- Expected: 约 12 行命中，分布在 `resolve_git_reference_root()`、`ensure_bootstrap_local_conf()` 内的 Python 代码、以及 bash 日志输出中

- [ ] Step 2: 移除 `resolve_git_reference_root()` 函数体（`ob` 第 153-176 行）

  将整个函数替换为空操作或直接删除。由于后续 Task 3 会移除所有调用点，这里直接删除函数定义。

- Change: 删除 `ob` 第 153-176 行（`resolve_git_reference_root()` 整个函数）

- [ ] Step 3: 清理 `ensure_bootstrap_local_conf()` 内嵌 Python 中的 OB_GIT_REFERENCE_DIR 逻辑

  在 `ob` 的 `ensure_bootstrap_local_conf()` 函数中（约第 446-620 行），内嵌 Python 代码处理 local.conf 写入。需要移除以下 Python 变量和逻辑：

  - `ob_git_reference_dir_value` 变量声明和赋值
  - `ob_git_reference_dir_line` 变量声明
  - `ob_git_reference_dir_action` 变量声明
  - `ob_git_reference_dir_log_value` 变量声明
  - `ob_git_reference_dir_log_detail` 变量声明
  - `elif key == "OB_GIT_REFERENCE_DIR"` 分支
  - `if dl_dir_value:` 块中生成 `ob_line` 和写入 `lines` 的逻辑
  - `elif ob_git_reference_dir_value:` 块
  - Python 输出中的 `OB_GIT_REFERENCE_DIR` 打印行

  同时清理外层 bash 中匹配 `OB_GIT_REFERENCE_DIR` 的 case 分支（约第 591-605 行）。

- Change: 在 `ob` 的 `ensure_bootstrap_local_conf()` 中移除所有 `OB_GIT_REFERENCE_DIR` 相关代码

- [ ] Step 4: 清理 `inject_externalsrc()` 中引用 OB_GIT_REFERENCE_DIR 的注释

  `ob` 第 1563 行和第 1593 行附近有注释提到 `OB_GIT_REFERENCE_DIR`。更新这些注释，移除对已废弃变量的引用。

- Change: 更新 `ob` 中 `inject_externalsrc()` 函数内的注释

- [ ] Step 5: 验证无残留引用

- Run: `grep -n 'OB_GIT_REFERENCE_DIR' /bmc/iasi/ob-harness-community/ob`
- Expected: 无输出（0 行命中）

- [ ] Step 6: 验证 `ob init --dry-run` 不报语法错误

- Run: `bash -n /bmc/iasi/ob-harness-community/ob`
- Expected: 无输出（语法检查通过）

- [ ] Step 7: 可选 checkpoint commit

- Run: `git add ob && git commit -m "refactor(ob): remove OB_GIT_REFERENCE_DIR mechanism"`

---

### Task 3: 重构 `clone_sub_repos()` 为 bare mirror 先行流程

- 目标：将 `clone_sub_repos()` 的核心循环改造为 "bare mirror → working tree → checkout" 三阶段流程
- 涉及文件：`ob` 中 `clone_sub_repos()` 函数（约第 1183-1488 行）
- 验证范围：`ob init --dry-run romulus` 输出包含 mirror 阶段信息；`ob init romulus` 执行后 `workspace/downloads/git2/` 包含 bare mirror

- [ ] Step 1: 确认当前 `clone_sub_repos()` 的 while 循环入口和变量读取结构

  当前结构（需保留）：
  - 读取 deps.json 中每个 repo 的 name, srcrev, clone_url, branch, src_uri
  - URL 展开和重写逻辑（`clone_url` 中 `${VAR}` 替换、`_url_rewrites` 表）

  这些前置处理逻辑保持不变。

- [ ] Step 2: 在 while 循环开始前，调用 `resolve_effective_dl_dir()` 获取有效 DL_DIR

  在 `clone_sub_repos()` 函数开头，替换当前的 `reference_root` 逻辑：

  **移除（约第 1205-1208 行）：**
  ```bash
  local reference_root=""
  reference_root=$(resolve_git_reference_root 2>/dev/null || true)
  if [[ -n "$reference_root" ]]; then
      info "Step 5 git reference cache: $reference_root"
  fi
  ```

  **替换为：**
  ```bash
  local effective_dl_dir=""
  effective_dl_dir=$(resolve_effective_dl_dir)
  local mirror_base="$effective_dl_dir/git2"
  info "Step 5 mirror cache: $mirror_base"
  ```

- Change: 替换 `clone_sub_repos()` 开头的 reference_root 逻辑为 effective_dl_dir 逻辑

- [ ] Step 3: 重写 while 循环内的克隆逻辑为三阶段

  while 循环内，在 URL 展开和重写之后、srcrev checkout 之前，将当前的"直接 clone working tree"逻辑替换为以下三阶段。

  新增全局统计数组（在 `clone_sub_repos()` 函数开头，`STATUS_*` 数组旁）：

  ```bash
  STATUS_MIRROR_NEW=()
  STATUS_MIRROR_EXISTING=()
  ```

  **阶段 A：Bare mirror**

  在 URL 重写逻辑之后，插入：

  ```bash
  # --- Phase A: Ensure bare mirror exists in DL_DIR/git2/ ---
  local mirror_path=""
  mirror_path=$(derive_bitbake_git_mirror_path "$mirror_base" "$src_uri" 2>/dev/null || true)

  if [[ -z "$mirror_path" ]]; then
      # Cannot derive mirror path (malformed SRC_URI) — skip mirror, clone directly
      verbose "Cannot derive mirror path for $name, cloning working tree directly"
  elif [[ -d "$mirror_path" ]]; then
      # Mirror exists — fetch updates
      verbose "Fetching updates for mirror: $mirror_path"
      git -C "$mirror_path" fetch --all 2>/dev/null || warn "Failed to fetch mirror for $name, continuing"
      STATUS_MIRROR_EXISTING+=("$name")
  else
      # Mirror missing — create full bare clone from remote
      verbose "Creating bare mirror: $clone_url -> $mirror_path"
      mkdir -p "$(dirname "$mirror_path")"
      if git clone --bare "$clone_url" "$mirror_path" 2>>"$_clone_err"; then
          STATUS_MIRROR_NEW+=("$name")
      else
          rm -rf "$mirror_path" 2>/dev/null
          warn "Failed to create bare mirror for $name, will clone working tree directly"
          mirror_path=""
      fi
  fi
  ```

  **阶段 B：Working tree**

  替换当前从第 1301 行开始的 `if [[ -d "$local_path/.git" ]]` 到 `fi` 的大段 clone 逻辑。新逻辑：

  ```bash
  # --- Phase B: Ensure working tree exists ---
  if [[ -d "$local_path/.git" ]]; then
      # Working tree exists — fetch updates
      verbose "Fetching updates for $name..."
      if ! git -C "$local_path" fetch --all 2>/dev/null; then
          warn "Failed to fetch $name, continuing with existing state."
      fi
  elif [[ -n "$mirror_path" && -d "$mirror_path" ]]; then
      # No working tree, but mirror exists — clone from mirror (local, fast)
      verbose "Cloning working tree from mirror: $mirror_path -> $local_path"
      local wt_clone_flags=()
      if [[ -n "$branch" ]]; then
          wt_clone_flags+=(--single-branch --branch "$branch")
      fi
      if ! git clone "${wt_clone_flags[@]}" "$mirror_path" "$local_path" 2>/dev/null; then
          rm -rf "$local_path" 2>/dev/null
          warn "Clone from mirror failed for $name, falling back to remote..."
          # Fallback: clone directly from remote
          _clone_from_remote "$clone_url" "$local_path" "$branch" "$_clone_err" || {
              STATUS_FAILED+=("$name (clone failed)")
              failed=$((failed + 1))
              continue
          }
      fi
  else
      # No mirror — clone directly from remote (backward compatible)
      _clone_from_remote "$clone_url" "$local_path" "$branch" "$_clone_err" || {
          STATUS_FAILED+=("$name (clone failed)")
          failed=$((failed + 1))
          continue
      }
  fi
  ```

  **阶段 C：Checkout** — 保持现有 checkout 逻辑不变（约第 1438-1478 行）。

- Change: 替换 `clone_sub_repos()` while 循环内的克隆逻辑为三阶段流程

- [ ] Step 4: 提取远程克隆辅助函数 `_clone_from_remote()`

  阶段 B 的 fallback 和 "no mirror" 路径需要一个从远程克隆的函数。将当前代码中的 clone + shallow fallback 逻辑提取为 `_clone_from_remote()`：

  ```bash
  _clone_from_remote() {
      local clone_url="$1"
      local local_path="$2"
      local branch="$3"
      local clone_err="$4"

      local full_flags=()
      if [[ -n "$branch" ]]; then
          full_flags+=(--single-branch --branch "$branch")
      fi

      if is_private_url "$clone_url"; then
          verbose "Full cloning (internal) $clone_url -> $local_path"
      else
          verbose "Full cloning (external) $clone_url -> $local_path"
      fi

      if ! git clone "${full_flags[@]}" "$clone_url" "$local_path" 2>>"$clone_err"; then
          rm -rf "$local_path" 2>/dev/null
          warn "Full clone failed, retrying shallow..."
          sleep 2
          local shallow_flags=(--depth=1)
          if [[ -n "$branch" ]]; then
              shallow_flags+=(--single-branch --branch "$branch")
          fi
          if ! git clone "${shallow_flags[@]}" "$clone_url" "$local_path" 2>>"$clone_err"; then
              rm -rf "$local_path" 2>/dev/null
              error "Failed to clone from $clone_url"
              local _last_err
              _last_err=$(grep -iE 'fatal|error|could not' "$clone_err" 2>/dev/null | tail -1 | sed 's/^[[:space:]]*//')
              if [[ -n "$_last_err" ]]; then
                  error "  git: $_last_err"
              fi
              return 1
          fi
          STATUS_SHALLOW+=("$(basename "$local_path")")
      fi
      return 0
  }
  ```

  插入位置：`clone_sub_repos()` 函数之前。

- Change: 在 `clone_sub_repos()` 之前新增 `_clone_from_remote()` 辅助函数

- [ ] Step 5: 更新 `clone_sub_repos()` 尾部的统计输出

  在 `info "Sub-repos: ${#STATUS_SUCCESS[@]} succeeded, $failed failed."` 之后添加：

  ```bash
  info "Mirrors: ${#STATUS_MIRROR_NEW[@]} new, ${#STATUS_MIRROR_EXISTING[@]} existing in $mirror_base"
  ```

- Change: 在 `clone_sub_repos()` 末尾添加 mirror 统计行

- [ ] Step 6: 验证语法正确

- Run: `bash -n /bmc/iasi/ob-harness-community/ob`
- Expected: 无输出（语法检查通过）

- [ ] Step 7: 用 `--dry-run` 验证流程不报错

  注意：`--dry-run` 模式下 Step 5 会提前 return，不会进入新逻辑。需要确认 `--dry-run` 的提前返回仍然正常。

- Run: `cd /bmc/iasi/ob-harness-community && ./ob init romulus --dry-run 2>&1 | tail -5`
- Expected: 输出包含 `[DRY-RUN] Would clone/fetch repositories listed in`，无报错

- [ ] Step 8: 可选 checkpoint commit

- Run: `git add ob && git commit -m "feat(ob): refactor clone_sub_repos() to bare-mirror-first flow"`

---

### Task 4: 更新 `print_report()` 添加 mirror 统计

- 目标：在 Step 8 报告中展示 mirror 数量、路径和新建/已有分布
- 涉及文件：`ob` 中 `print_report()` 函数（约第 1641-1721 行）
- 验证范围：`ob init` 执行后 report 包含 mirror 信息行

- [ ] Step 1: 在 report 的 "Sub-repos cloned" 统计块之后插入 mirror 统计

  在 `print_report()` 函数中，`echo "Sub-repos cloned: ${#STATUS_SUCCESS[@]}"` 块之后，添加：

  ```bash
  echo "Mirrors: ${#STATUS_MIRROR_NEW[@]} new, ${#STATUS_MIRROR_EXISTING[@]} existing"
  if [[ ${#STATUS_MIRROR_NEW[@]} -gt 0 ]]; then
      echo "  Mirror cache: $mirror_base"
  fi
  ```

  注意：`mirror_base` 变量在 `clone_sub_repos()` 中是 local 的。需要将 `mirror_base` 提升为文件级变量，或在 `clone_sub_repos()` 结束时写入一个变量供 `print_report()` 读取。

  **实现方式**：在 `clone_sub_repos()` 开头，将 `mirror_base` 的赋值改为文件级变量。在 `ob` 的全局变量区域（约第 7-25 行）添加：

  ```bash
  MIRROR_BASE=""
  ```

  然后在 `clone_sub_repos()` 中赋值：

  ```bash
  MIRROR_BASE="$effective_dl_dir/git2"
  ```

  `print_report()` 中引用 `$MIRROR_BASE`。

- Change: 在全局变量区添加 `MIRROR_BASE=""`；在 `print_report()` 中添加 mirror 统计块

- [ ] Step 2: 验证语法

- Run: `bash -n /bmc/iasi/ob-harness-community/ob`
- Expected: 无输出

- [ ] Step 3: 可选 checkpoint commit

- Run: `git add ob && git commit -m "feat(ob): add mirror statistics to print_report()"`

---

### Task 5: 端到端验证

- 目标：确认完整 `ob init romulus` 流程在真实环境中正确执行，`downloads/git2/` 包含 bare mirror
- 涉及文件：无代码改动，纯验证
- 验证范围：设计文档中定义的 6 条成功标准

- [ ] Step 1: 备份当前 workspace 状态（可选）

  如果 workspace 中已有 `src/romulus/` 的 working tree，这些不需要删除——新逻辑会检测已有 working tree 并补建 mirror。

- [ ] Step 2: 执行 `ob init romulus`

- Run: `cd /bmc/iasi/ob-harness-community && ./ob init romulus`
- Expected:
  - Step 5 输出包含 `mirror cache:` 行
  - 每个 repo 的日志显示 bare mirror 创建或已有
  - 最终 report 包含 mirror 统计行

- [ ] Step 3: 验证 `downloads/git2/` 包含 bare mirror

- Run: `ls /bmc/iasi/ob-harness-community/workspace/downloads/git2/ | head -10`
- Expected: 看到 `github.com.openbmc.*` 等目录名

- Run: `ls /bmc/iasi/ob-harness-community/workspace/downloads/git2/ | wc -l`
- Expected: 数量 >= deps.json 中的 repo 数量（部分 SRC_URI 无法解析的可能跳过）

- [ ] Step 4: 验证 mirror 是 bare repo

- Run: `git -C /bmc/iasi/ob-harness-community/workspace/downloads/git2/github.com.openbmc.bmcweb.git rev-parse --is-bare-repository`
- Expected: `true`

- [ ] Step 5: 验证 working tree 仍然正常

- Run: `git -C /bmc/iasi/ob-harness-community/workspace/src/romulus/bmcweb rev-parse HEAD`
- Expected: 输出一个有效的 commit hash

- [ ] Step 6: 最终 checkpoint commit

- Run: `git add -A && git commit -m "feat(ob): DL_DIR/git2/ bare mirror cache — verified"`

## 执行纪律

- 开始实现前，先批判性复查整份计划；如果发现缺项、矛盾、命名不一致或验证命令无效，先修计划
- 按任务顺序执行，不要无声跳步、合并步或改变任务目标
- 每完成一个任务，都运行该任务定义的验证
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下来说明，不要猜
- 当前在 `main` 分支；开始实现前先创建 feature branch
- 全部任务完成后，运行最终验证并输出修改摘要

## 最终验证

```bash
# 1. 语法检查
bash -n ob

# 2. 确认 OB_GIT_REFERENCE_DIR 完全移除
grep -c 'OB_GIT_REFERENCE_DIR' ob
# Expected: 0

# 3. 确认新函数存在
grep -c 'resolve_effective_dl_dir' ob
# Expected: >= 2 (定义 + 调用)

# 4. 确认 bare mirror 目录存在且有内容
ls workspace/downloads/git2/ | wc -l
# Expected: > 0

# 5. 确认 mirror 是 bare repo
git -C workspace/downloads/git2/$(ls workspace/downloads/git2/ | head -1) rev-parse --is-bare-repository
# Expected: true
```
