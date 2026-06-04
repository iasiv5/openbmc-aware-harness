# parse_bitbake_deps.py Tinfoil 优化实施计划

> **修订说明（2026-06-04 实施后更新）：** 原设计假设 Tinfoil `all_recipes()` 可直接获取 ~569 个 target deps，但实测返回 ~4492 个。调整策略为保留 `bitbake -g` 生成 `pn-buildlist`，Tinfoil 仅查询该文件中的 recipes。所有 4 个实施任务已完成并验证。本文档已更新以反映实际实现方案。

## 目标

将 `ob init` Step 5（生成依赖图）中 `bitbake -e × 569` 的瓶颈（~14 min）替换为 Tinfoil 单进程查询（~37s），整体耗时从 ~17 min 降到 ~3.5 min，保持 `deps.json` 输出完全兼容。

## 架构快照

**旧流程：** `bitbake -g`（~3 min）生成 `pn-buildlist` → `parse_bitbake_deps.py` 逐行读 recipe 名 → 对每个 recipe 启动 `bitbake -e` 子进程（569 × ~1.5s ≈ 14 min）

**新流程：** `bitbake -g`（~3 min，保留）生成 `pn-buildlist` → `parse_bitbake_deps.py --build-dir` 使用 bitbake Tinfoil API 在单进程内查询 `pn-buildlist` 中的 569 个 recipes（一次 init ~7s + 569 × ~0.06s ≈ 37s）。`ob` 脚本中 `generate_dep_graph()` 保留 `bitbake -g` 调用，Python 脚本参数从 `--pn-buildlist` 改为 `--build-dir`。

`ob` 步骤编号不变（仍是 Step 1-8），仅 Step 5 内部实现替换。

## 输入工件

- 设计文档：`docs/specs/2026-06-04-tinfoil-deps-optimization-design.md`
- 现有脚本：`ob` 中 `generate_dep_graph()` 函数
- 现有 Python：`tools/parse_bitbake_deps.py`

## 文件结构与职责

- **修改** `tools/parse_bitbake_deps.py`：新增 Tinfoil 查询路径（`--build-dir`），保留旧 `bitbake -e` 路径（`--pn-buildlist`）作为回退
- **修改** `ob` 中 `generate_dep_graph()` 函数：保留 `bitbake -g` 调用，将 `--pn-buildlist` 参数替换为 `--build-dir`
- **修改** `ob` 中 `usage()` 函数：从 `--help` 中移除 `--skip-deps`
  cd /bmc/iasi/openbmc-aware-harness
  cp workspace/openbmc/build/<MACHINE>/deps.json /tmp/deps_baseline.json

  - Run: `wc -l /tmp/deps_baseline.json`
  - Expected: 显示行数（~2000+），文件存在且非空

- [ ] **Step 2: 重写 parse_bitbake_deps.py**

  核心改动：
  - 新增 `query_deps_tinfoil(build_dir, machine)` 函数：使用 `bb.tinfoil.Tinfoil` 的 `parse_recipe()` + `getVar('SRC_URI')` / `getVar('SRCREV')` 替代 `query_recipe_env()` 的子进程调用
  - 新增 `--build-dir` 参数：Tinfoil 需要从中读取 `conf/bblayers.conf`，将 `bitbake.lib` 路径加入 `sys.path`
  - `--pn-buildlist` 降级为可选：传入时仍走旧 `bitbake -e` 路径（回退兼容），不传时走 Tinfoil 路径
  - 保留所有现有辅助函数（`extract_git_srcuris`, `extract_repo_name`, `convert_to_https_url`）不变
  - 保留 `signal.signal(SIGINT, ...)` 中断处理
  - 保留 stderr 进度输出格式（`\rQuerying recipe N/total: name...`）
  - stdout JSON 输出字段不变：`name, src_uri, srcrev, recipe, clone_url, branch`

  Tinfoil 初始化关键代码：
  sys.path.insert(0, os.path.join(openbmc_dir, 'bitbake', 'lib'))
  from bb.tinfoil import Tinfoil

  # 从 pn-buildlist 读取精确的 recipe 列表（~569 个）
  pn_buildlist = os.path.join(build_dir, 'pn-buildlist')
  if os.path.isfile(pn_buildlist):
      recipe_names = [l.strip() for l in open(pn_buildlist) if l.strip()]
  else:
      # 降级：使用 all_recipes() + native 过滤（较慢，~2750 个）
      # ...

  real_stdout = sys.stdout
  sys.stdout = sys.stderr  # Tinfoil.prepare() 会向 stdout 写日志，污染 JSON

  with Tinfoil() as tinfoil:
      tinfoil.prepare(config_only=False)
      sys.stdout = real_stdout  # 恢复 stdout 给后续输出
      for recipe_name in recipe_names:
          try:
              d = tinfoil.parse_recipe(recipe_name)
          except Exception:
              continue  # 跳过无法解析的 recipe
          src_uri = d.getVar('SRC_URI') or ''
          srcrev = d.getVar('SRCREV') or ''
          # ... extract git URIs same as before

  - Run: `bash -n tools/parse_bitbake_deps.py` 不适用（Python），改用 `python3 -m py_compile tools/parse_bitbake_deps.py`
  - Expected: 编译通过，无 syntax error

- [ ] **Step 3: 在 build 环境中生成新版输出并对比**

  cd /bmc/iasi/openbmc-aware-harness/workspace/openbmc
  bash -c '
  cd build/<MACHINE>
  set +u
  source ../../../setup <MACHINE> . 2>/dev/null
  python3 ../../../tools/parse_bitbake_deps.py --build-dir . --machine <MACHINE> > /tmp/deps_new.json
  echo "exit: $?"
  '

  对比脚本：
  python3 -c "
  import json, sys
  old = sorted(json.load(open('/tmp/deps_baseline.json')), key=lambda x: x['name'])
  new = sorted(json.load(open('/tmp/deps_new.json')), key=lambda x: x['name'])
  old_names = [d['name'] for d in old]
  new_names = [d['name'] for d in new]
  print(f'Old: {len(old)} repos, New: {len(new)} repos')
  missing = set(old_names) - set(new_names)
  extra = set(new_names) - set(old_names)
  if missing: print(f'Missing in new: {sorted(missing)}')
  if extra: print(f'Extra in new: {sorted(extra)}')
  if not missing and not extra:
      print('Name sets match!')
  # Spot-check fields
  for i in range(min(3, len(old))):
      if old[i] != new[i]:
          print(f'DIFF at [{i}]:')
          print(f'  old: {old[i]}')
          print(f'  new: {new[i]}')
  "

  - Run: 上述对比命令
  - Expected: repo 数量一致（差值 ≤ 5，因 Tinfoil 可能有更精确的 recipe 解析），name 集合基本匹配，抽查字段无差异

- [ ] **Step 4: 测量耗时**

  cd /bmc/iasi/openbmc-aware-harness/workspace/openbmc
  bash -c '
  cd build/<MACHINE>
  set +u
  source ../../../setup <MACHINE> . 2>/dev/null
  time python3 ../../../tools/parse_bitbake_deps.py --build-dir . --machine <MACHINE> > /dev/null
  '
  - Expected: real < 60s（Tinfoil 部分实测 ~37s；若含 bitbake -g 则 ~3.5 min）

### Task 2: ob 脚本 — 更新 generate_dep_graph()（✅ 已完成）

- **目标：** 将 `ob` 中的 `generate_dep_graph()` 的 `parse_bitbake_deps.py --pn-buildlist` 调用替换为 `parse_bitbake_deps.py --build-dir --machine`，保留 `bitbake -g` 调用
- **Files：** `ob`（`generate_dep_graph()` 函数）
- **验证范围：** `bash -n ob` 语法检查通过
  cd /bmc/iasi/openbmc-aware-harness
  sed -n '978,1017p' ob
  - Expected: 看到 `bitbake -g obmc-phosphor-image` 调用（保留）和 `--build-dir` 参数（替代 `--pn-buildlist`）

- [x] **Step 2: 更新 generate_dep_graph() 的 Python 调用参数**

  核心改动：
  - **保留** `bitbake -g obmc-phosphor-image` 调用（~3 min，生成 pn-buildlist）
  - 将 `python3 "$HARNESS_ROOT/tools/parse_bitbake_deps.py" --pn-buildlist "$BUILD_DIR/pn-buildlist" --machine "$MACHINE"` 改为 `python3 "$HARNESS_ROOT/tools/parse_bitbake_deps.py" --build-dir "$BUILD_DIR" --machine "$MACHINE"`
  - 更新 `info` 消息：更新耗时说明为 "~3 min for dependency graph + ~30s for repo extraction"
  - 保留 dry-run 分支、tmp 文件原子写入、`mv` 重命名逻辑和 `dep_count` 统计
  - 保留 `source setup` 环境 re-enter

  - Run: `bash -n ob`
  - Expected: 无语法错误
### Task 3: ob 脚本 — 降级 --skip-deps 为隐性选项

- **目标：** 从 `--help` 输出中移除 `--skip-deps`，但保留代码中的功能实现（应急出口）
- **Files：** `ob`（`usage()` 函数，第 175-196 行）
- **验证范围：** `--help` 不再显示 `--skip-deps`

- [ ] **Step 1: 从 usage() 中移除 --skip-deps 帮助行**

  删除 `  -s, --skip-deps       Skip dependency graph generation (reuse existing deps.json)` 这一行。保留 `parse_args()` 中的 `-s|--skip-deps` case 分支和 `main()` 中的 `SKIP_DEPS` 逻辑。

  - Run: `bash ob --help 2>&1 | grep -c skip-deps`
  - Expected: 0（不再出现在帮助中）

- [ ] **Step 2: 确认 --skip-deps 仍可解析**

  cd /bmc/iasi/openbmc-aware-harness
  bash -c 'source ob --help' 2>&1 || true
  # 更直接的验证：bash -n 确认 SKIP_DEPS 变量和 parse_args 逻辑仍在
  grep -n 'SKIP_DEPS\|--skip-deps' ob
### Task 4: 端到端验证

- **目标：** 用新版 `ob` 跑完整的 `ob init <MACHINE> --skip-deps --dry-run` 验证流程不变
- **Files：** 不涉及文件修改
- **验证范围：** dry-run 输出中 Step 5 的行为符合预期

- [ ] **Step 1: dry-run 验证**
  cd /bmc/iasi/openbmc-aware-harness
  bash ob init <MACHINE> --dry-run 2>&1 | grep -A2 "Step 5"

  (注：Step 编号不变——Step 5 是 "Generating dependency graph"，generate_dep_graph 的 step_header 会更新。)

  - Expected: Step 5 仍显示 "Generating dependency graph..."，但后续 info 消息不再提到 "5-10 minutes"

- [ ] **Step 2: 验证 Step 5 使用新参数调用**

  grep "parse_bitbake_deps.py" ob

  - Expected: 看到 `--build-dir "$BUILD_DIR"` 而不是 `--pn-buildlist`

## 执行纪律

- 开始实现前，先复查整份计划；如果发现缺项、矛盾或验证命令无效，先修计划
- 按任务顺序执行，不要无声跳步、合并步或改变任务目标
- 每完成一个任务，都运行该任务定义的验证
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下来说明，不要猜
- 如果当前就在 `main` 或 `master`，且用户没有明确同意，开始实现前先确认
- 全部任务完成后，运行最终验证并输出修改摘要

## 最终验证

全部任务完成后，执行以下确认：

```bash
cd /bmc/iasi/openbmc-aware-harness

# 1. 语法检查
bash -n ob && echo "ob syntax OK"
python3 -m py_compile tools/parse_bitbake_deps.py && echo "parse_bitbake_deps.py syntax OK"

# 2. help 不含 --skip-deps
bash ob --help 2>&1 | grep skip-deps; echo "exit:$?"
# Expected: exit:1 (grep 找不到)

# 3. 旧 --skip-deps 功能仍可用（隐性）
grep -c 'SKIP_DEPS' ob
# Expected: >= 3

# 4. 确认 generate_dep_graph 保留 bitbake -g 并使用 --build-dir
grep -n 'bitbake -g\|--build-dir\|--pn-buildlist' ob | head -10
# Expected: 看到 bitbake -g 调用和 --build-dir 参数，无 --pn-buildlist（legacy 模式除外）

# 5. 新版 deps.json 端到端可生成（需 build 环境）
cd workspace/openbmc
bash -c '
cd build/<MACHINE>
set +u
source ../../../setup <MACHINE> . 2>/dev/null
time python3 ../../../tools/parse_bitbake_deps.py --build-dir . --machine <MACHINE> > /tmp/deps_final.json 2>/tmp/deps_final_stderr.txt
echo "exit: $?"
tail -5 /tmp/deps_final_stderr.txt
'
# Expected: real < 60s (Tinfoil 部分), exit 0
```
