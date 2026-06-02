# OpenBMC 主仓单一来源锁定 —— 实施计划

## 目标

在 `tools/ob` 中实现「一个 harness 只绑定一个 OpenBMC 主仓来源」的硬契约：首次克隆写来源锁文件，后续 `ob init` 做篡改检测（lock vs 现有树 git origin）与本次来源比对（显式来源 vs lock），不一致即在 clone 前停止；并新增只读 `ob status` 展示绑定状态。

依据设计：[docs/specs/2026-06-02-obmc-single-source-lock-design.md](docs/specs/2026-06-02-obmc-single-source-lock-design.md)。

> 实施结果：已完成，以下任务与验证命令已按当前脚本行为同步。

## 架构快照

全部改动落在单文件 [tools/ob](tools/ob)（bash）。新增能力以独立函数实现，挂接到三个既有锚点：

- `select_openbmc_repo_url()`：交互分支处补一个 `SOURCE_LABEL` 赋值（社区→`community`，自定义→`custom`）。
- `clone_openbmc()`：克隆成功后调用 `write_source_lock`；已存在 `.git` 时先调用 `verify_source`，由它决定放行或停止，再走原有 skip 逻辑。
- `parse_args()` + `main()`：把命令从「只接受 init」扩展为「init | status」分发。

锁文件落到 `$CONFIGS_DIR/openbmc-source.lock`（即 `workspace/configs/openbmc-source.lock`，`CONFIGS_DIR` 已在 `detect_harness_root()` 定义）。

## 文件结构与职责

| 文件 | 改动 | 职责 |
|---|---|---|
| `tools/ob` | 修改 | 新增 5 个函数 + 改 3 个锚点 + 改 `usage()`；不新建文件 |

新增函数（建议紧挨现有 `is_valid_repo_url()` 与 `select_openbmc_repo_url()` 之后，集中放在「Functions」区）：

- `normalize_repo_url <url>` — 输出规范化 `host/path`
- `write_source_lock` — 写 `openbmc-source.lock`
- `read_lock_field <key>` — 从 lock 读单个字段值
- `verify_source` — 篡改检测 + 本次来源比对，决定放行/退出
- `cmd_status` — `ob status` 只读输出

新增全局变量：`SOURCE_LABEL=""`、`SOURCE_LOCK_FILE=""`（在 `detect_harness_root()` 内赋值 `SOURCE_LOCK_FILE="$CONFIGS_DIR/openbmc-source.lock"`）。

## 任务清单

### Task 1：加全局变量与 lock 路径

- 在全局变量区（`CONFIGS_DIR=""` 附近）新增 `SOURCE_LABEL=""` 和 `SOURCE_LOCK_FILE=""`。
- 在 `detect_harness_root()` 末尾（`CONFIGS_DIR="$WORKSPACE_DIR/configs"` 之后）加 `SOURCE_LOCK_FILE="$CONFIGS_DIR/openbmc-source.lock"`。
- **验证**：`bash -n tools/ob` 退出 0；`grep -n 'SOURCE_LOCK_FILE=' tools/ob` 显示两处（声明 + 赋值）。

### Task 2：实现 `normalize_repo_url`

按设计规范化规则：去协议头（`https://`/`http://`/`ssh://`/`git://`）、`git@host:path`→`host/path`、去用户名@与端口、去末尾 `.git` 与 `/`、转小写。

- 紧接 `is_valid_repo_url()` 之后新增函数，`echo` 规范化结果。
- **验证**（函数级单点测试，bash 内联）：
`OB_NO_MAIN=1 bash -c 'source tools/ob; for u in "git@github.com:openbmc/openbmc.git" "https://github.com/openbmc/openbmc" "ssh://git@github.com:22/OpenBMC/OpenBMC.git/"; do normalize_repo_url "$u"; done'`
预期三行全部输出 `github.com/openbmc/openbmc`。

### Task 3：实现 `write_source_lock`

写入字段：`normalized_source`、`origin_url`、`source_label`、`machine_first_init`、`created_at`（UTC ISO8601）。来源 URL 取 `$OPENBMC_REPO_URL`，规范化后写 `normalized_source`。`--dry-run` 时只打印不写盘。

- **验证**：`bash -n tools/ob` 退出 0；逻辑验证并入 Task 7 的首次 clone dry-run 场景。

### Task 4：实现 `read_lock_field`

`read_lock_field <key>`：从 `$SOURCE_LOCK_FILE` 读 `key=value`，忽略 `#` 注释行，echo value；文件或 key 不存在则 echo 空、返回非 0。

- **验证**：`bash -n tools/ob` 退出 0；逻辑验证并入 Task 8 status 测试。

### Task 5：实现 `verify_source`

按设计控制流：

1. 若 `$OPENBMC_DIR/.git` 不存在 → 直接 return 0（交给 clone 流程）。
2. 读 lock：无 lock → 回退读 `git -C "$OPENBMC_DIR" remote get-url origin`，规范化后补写 lock（origin 缺失则 warn 跳过补写），return 0（放行）。
3. 有 lock → 取 `lock_src=$(read_lock_field normalized_source)`。
4. **篡改检测**：读现有树 origin → 规范化为 `tree_src`；origin 缺失 warn 跳过；`tree_src != lock_src` → 打印篡改文案，`exit 1`。
5. **本次来源比对**：仅当 `OPENBMC_REPO_URL` 已被本次显式来源填充时，规范化为 `req_src`；`req_src != lock_src` → 打印冲突文案（建议另开 harness copy），`exit 1`。
6. 否则 return 0。

错误文案沿用设计「错误提示文案」章节两段（来源冲突 / 篡改）。

- **验证**：`bash -n tools/ob` 退出 0；行为验证并入 Task 7。

### Task 6：接线 `clone_openbmc` 与捕获 `SOURCE_LABEL`

- 在 `clone_openbmc()` 开头、`.git` 存在分支之前调用 `verify_source`（它内部决定放行/退出）；保留现有「已存在则 skip + 提示 git pull」逻辑。
- 在 `.git` 不存在分支，`git clone` 成功之后调用 `write_source_lock`。
- 在 `select_openbmc_repo_url()`：选项 1（社区）后置 `SOURCE_LABEL="community"`，选项 2（自定义 URL）后置 `SOURCE_LABEL="custom"`；`--openbmc-url`/`OB_OPENBMC_URL` 分支不设（留空）。
- **验证**：`bash -n tools/ob` 退出 0；`grep -n 'verify_source\|write_source_lock\|SOURCE_LABEL=' tools/ob` 显示接线点齐全。

### Task 7：dry-run 行为回归（init 路径）

用 `--dry-run` 验证不写盘、分支正确。

- **验证**（已存在主仓、无来源锁的旧 harness 场景）：
`cd /bmc/iasi/workspace/openbmc-aware-harness`

`rm -f workspace/configs/openbmc-source.lock`

`./tools/ob init romulus --dry-run --openbmc-url https://github.com/openbmc/openbmc.git`
预期：打印 `Would write source lock`，随后继续现有 init 的 dry-run 输出；不会实际写入 `workspace/configs/openbmc-source.lock`。
- 若本机已存在 `workspace/openbmc/.git`：再跑一次显式**异源** dry-run，预期命中 `verify_source` 在 clone 前停止、退出非 0、打印冲突文案。
- 若需要真实补写来源锁：执行 `./tools/ob init romulus --skip-fetch`；预期在 Step 2 完成 lock bootstrap，然后继续原有 Step 3-8 流程。

### Task 8：实现 `ob status` 子命令与命令分发

- 在脚本顶部 `main "$@"` 调用处加 `OB_NO_MAIN` 守卫：`[[ -n "${OB_NO_MAIN:-}" ]] || main "$@"`，便于函数级测试 source 不触发主流程（支撑 Task 2/4 验证）。
- `parse_args()`：将 `if [[ "$command" != "init" ]]` 改为支持 `init|status` 分发；`status` 接受可选 `[<machine>]`，不强制。
- `main()`：按命令分发——`init` 走现有 8 步；`status` 调 `cmd_status` 后返回。
- `cmd_status`：按设计五状态表读 lock + 现有树 origin，输出绑定来源/标签/原始 URL/首次 machine/时间/当前 origin/一致性，篡改态退出非 0，其余 0。
- `usage()`：补 `status [<machine>]` 说明与示例。
- **验证**：
`bash -n tools/ob && echo SYNTAX_OK`

`./tools/ob --help | grep -q status && echo HELP_OK`

`./tools/ob status; echo "status_exit=$?"`
预期：语法 OK；help 文本含 `status`；在已绑定场景下输出 `Bound source` / `Origin URL` / `Current origin ... (origin matches)` / `Status: OK`，退出码为 0。
- 回归 Task 2：`OB_NO_MAIN=1 bash -c 'source tools/ob; for u in git@github.com:openbmc/openbmc.git https://github.com/openbmc/openbmc ssh://git@github.com:22/OpenBMC/OpenBMC.git/; do normalize_repo_url "$u"; done'` 三行均为 `github.com/openbmc/openbmc`。

## 执行纪律

- 每个 Task 后跑 `bash -n tools/ob` 确认语法。
- 若环境有 `shellcheck`，每个 Task 后追加 `shellcheck tools/ob`（无则跳过，不阻塞）。
- 不改动现有 8 步 init 逻辑的行为，只在 `clone_openbmc` 增加前置校验与克隆后写 lock。
- 不新建文件、不引入 YAML 或其他依赖。
- checkpoint commit 时机：Task 5 完成（核心校验逻辑）后一次；Task 8 完成（status + 分发）后一次。

## 最终验证
```bash
cd /bmc/iasi/workspace/openbmc-aware-harness
bash -n tools/ob && echo SYNTAX_OK
command -v shellcheck >/dev/null && shellcheck tools/ob || echo "shellcheck skipped"
OB_NO_MAIN=1 bash -c 'source tools/ob; \
  for u in "git@github.com:openbmc/openbmc.git" "https://github.com/openbmc/openbmc" "ssh://git@github.com:22/OpenBMC/OpenBMC.git/"; do \
    normalize_repo_url "$u"; done'   # 三行均 github.com/openbmc/openbmc
./tools/ob --help | grep -q status && echo HELP_OK
./tools/ob status; echo "status_exit=$?"
rm -f workspace/configs/openbmc-source.lock
./tools/ob init romulus --dry-run --openbmc-url https://github.com/openbmc/openbmc.git
test ! -f workspace/configs/openbmc-source.lock && echo DRYRUN_NO_WRITE_OK
./tools/ob init romulus --skip-fetch
./tools/ob status; echo "status_exit=$?"
```
预期：语法 OK、规范化三行一致、help 含 status、dry-run 只打印不写盘；真实 `--skip-fetch` 会在已有主仓场景补写 `workspace/configs/openbmc-source.lock`，随后 `status` 显示已绑定来源且退出码为 0。