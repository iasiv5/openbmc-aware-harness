# QEMU Custom 路径重构实施计划

## 目标

重构 `ob` 脚本中 QEMU 相关流程，使 custom 路径支持：
1. 手动放置 QEMU 二进制 + pc-bios（交互引导拷贝）
2. SoC 自动检测（AST2700 / AST2600）
3. SoC 级参数模板（AST2700 bootloader 链 vs AST2600 简易启动）
4. 消除 community/custom 两条路径的代码重复

## 架构快照

**改动前**：`cmd_start_qemu()` 解析 QB_ 变量后，分派到 `cmd_start_qemu_community()` 和 `cmd_start_qemu_custom()`，两者代码 100% 相同（QEMU 命令构造完全一致），都用 `-machine` + `-drive` 的简单模板，community 通过 URL 下载二进制，custom 也是同一套下载逻辑。

**改动后**：
```
cmd_start_qemu()
  ├─ resolve_qb_vars()          ← 增加 SoC 回退
  ├─ detect_soc_type()          ← 新函数，双重验证
  ├─ derive_qemu_machine_name() ← 新函数，xxx-yyy → xxx-bmc
  ├─ ensure_qemu_binary()       ← 拆分：community 下载 vs custom 本地拷贝
  ├─ ensure_qemu_firmware()     ← 简化：检查 pc-bios 存在即可
  ├─ find_ast2700_bootloaders() ← 新函数，AST2700 专用（按需调用）
  ├─ build_qemu_cmd()           ← 新函数，按 SoC 选模板
  └─ _qemu_post_launch()        ← 不变
```

community 和 custom 的启动流程合并为统一路径，差异点只在二进制获取方式和命令模板参数。

## 输入工件

- 设计决策：本次 grilling session 中逐条确认的决策汇总（见上文对话）
- 参考实现：`workspace/openbmc/start-qemu.sh`（用户已验证可正常启动 AST2700/AST2600 的脚本）

## 文件结构与职责

- Modify: `ob`
  - 新增函数：`detect_soc_type()`、`derive_qemu_machine_name()`、`ensure_qemu_binary_custom()`、`find_ast2700_bootloaders()`、`build_qemu_cmd()`
  - 修改函数：`resolve_qb_vars()`、`ensure_qemu_binary()`、`ensure_qemu_firmware()`、`cmd_start_qemu()`
  - 删除函数：`cmd_start_qemu_community()`、`cmd_start_qemu_custom()`（逻辑合并入统一流程）

## 任务清单

### Task 1: 新增 `detect_soc_type()` 函数

- 目标：实现 SoC 类型检测，支持 QB_ 变量推断 + deploy 目录文件检测 + machine conf include 链三重来源，后两者交叉验证
- 涉及文件：`ob`
- 验证范围：函数能正确返回 `ast2700` 或 `ast2600`，并在两个来源矛盾时报错

- [ ] Step 1: 在 `resolve_qb_vars()` 函数之后（约 L767）插入新函数 `detect_soc_type()`

  函数逻辑：
  1. 如果 `QB_SYSTEM_NAME` 已设置：
     - `qemu-system-aarch64` → 推断 SOC_TYPE=ast2700
     - `qemu-system-arm` → 推断 SOC_TYPE=ast2600
     - 返回 0
  2. Deploy 目录文件检测：
     - 检查 `$BUILD_DIR/tmp/deploy/images/$MACHINE/bl31-ast2700.bin` 是否存在
     - 存在 → deploy_hint=ast2700，否则 → deploy_hint=ast2600
  3. Machine conf include 链检测：
     - 用 `find` 定位 `$OPENBMC_DIR/meta-*/conf/machine/$MACHINE.conf`
     - `grep -r 'ast2700-sdk.inc'` 该 conf 的 include 链（递归检查 conf 和 include 的文件）
     - 命中 → conf_hint=ast2700
     - 未命中但 `grep -r 'ast2600'` 命中 → conf_hint=ast2600
     - 都未命中 → conf_hint="" (无法判断)
  4. 交叉验证：
     - 如果 deploy_hint 和 conf_hint 都非空且不一致 → error 退出
     - 如果 deploy_hint 非空 → SOC_TYPE=deploy_hint
     - 如果 conf_hint 非空 → SOC_TYPE=conf_hint
     - 都空 → error 退出（无法确定 SoC 类型）
  5. 根据推断的 SOC_TYPE 设置 `QB_SYSTEM_NAME`（如果还是空的）：
     - ast2700 → QB_SYSTEM_NAME=qemu-system-aarch64
     - ast2600 → QB_SYSTEM_NAME=qemu-system-arm
  6. 导出全局变量 `SOC_TYPE`

- Change: 在 `ob` 的 `resolve_qb_vars()` 函数之后插入新函数

- [ ] Step 2: 验证函数能被调用

  ```bash
  # 在 ob 脚本末尾 main() 之前临时加一行测试：
  # 在 cmd_start_qemu 中 detect_soc_type 调用处验证
  bash -n ob && echo "syntax OK"
  ```

  - Run: `bash -n ob`
  - Expected: 无语法错误，输出 `syntax OK`

### Task 2: 新增 `derive_qemu_machine_name()` 函数

- 目标：QEMU `-M` 参数值的推导，QB_MACHINE 优先，回退到 machine 名转换规则
- 涉及文件：`ob`
- 验证范围：对 `b865g8-bytedance` 返回 `b865g8-bmc`，对已有 QB_MACHINE 的 machine 返回 QB_MACHINE 值

- [ ] Step 1: 在 `detect_soc_type()` 之后插入新函数 `derive_qemu_machine_name()`

  函数逻辑：
  1. 如果 `QB_MACHINE_NAME` 已非空（从 bitbake 解析得到）→ 直接返回
  2. 回退规则：
     - 取 `$MACHINE` 的第一个 `-` 前的部分
     - `local prefix="${MACHINE%%-*}"`
     - 如果 `prefix == "$MACHINE"`（没有 `-`）→ error 退出（无法推导 QEMU machine 名）
     - 否则 `QB_MACHINE_NAME="${prefix}-bmc"`
  3. info 日志输出推导结果

- Change: 插入新函数

- [ ] Step 2: 验证语法

  - Run: `bash -n ob`
  - Expected: `syntax OK`

### Task 3: 新增 `ensure_qemu_binary_custom()` 函数

- 目标：custom 路径的 QEMU 二进制获取——检测本地文件，不存在时交互引导用户提供绝对路径
- 涉及文件：`ob`
- 验证范围：当 `qemu-bin/custom/<arch>` 已存在时直接返回；不存在时能正确交互并拷贝

- [ ] Step 1: 在 `derive_qemu_machine_name()` 之后插入新函数 `ensure_qemu_binary_custom()`

  函数逻辑：
  1. 调用 `derive_qemu_paths` 确定目标路径
  2. 如果 `$QEMU_BIN_FILE` 存在且可执行 → `return 0`
  3. 进入交互模式（`[[ ! -t 0 ]]` 时报错退出）
  4. 第一步交互：提示用户输入 QEMU binary 绝对路径
     ```
     Enter absolute path to QEMU binary (qemu-system-aarch64):
     ```
     - 验证输入非空、不以 `-` 开头（不是 flag）、是绝对路径（以 `/` 开头）
     - 验证文件存在且是普通文件（`-f`）
     - 拷贝到 `$QEMU_BIN_FILE`，`chmod +x`
  5. 第二步交互：提示用户输入 pc-bios 目录绝对路径
     ```
     Enter absolute path to pc-bios directory:
     ```
     - 验证是绝对路径、目录存在（`-d`）
     - 如果目标 `$QEMU_BIN_DIR/pc-bios` 已存在 → `rm -rf` 后重建
     - `cp -r` 到 `$QEMU_BIN_DIR/pc-bios`
  6. 写 manifest 文件
  7. info 输出完成信息

- Change: 插入新函数

- [ ] Step 2: 验证语法

  - Run: `bash -n ob`
  - Expected: `syntax OK`

### Task 4: 拆分 `ensure_qemu_binary()` 为分派器

- 目标：将现有 `ensure_qemu_binary()`（L517-637）改为分派器，community 走下载逻辑，custom 走本地拷贝
- 涉及文件：`ob`（修改 `ensure_qemu_binary` 函数体，L517-637）
- 验证范围：分派器根据 source_label 正确调用不同子函数

- [ ] Step 1: 将现有 `ensure_qemu_binary()` 函数体重命名为 `ensure_qemu_binary_community()`

  - 把 L517 的 `ensure_qemu_binary()` 改名为 `ensure_qemu_binary_community()`
  - 函数体内容不变

- [ ] Step 2: 新建 `ensure_qemu_binary()` 作为分派器

  ```bash
  ensure_qemu_binary() {
      local label
      label=$(read_source_label)
      if [[ "$label" == "community" ]]; then
          ensure_qemu_binary_community
      else
          ensure_qemu_binary_custom
      fi
  }
  ```

- [ ] Step 3: 验证语法和调用链

  - Run: `bash -n ob`
  - Expected: `syntax OK`
  - Run: `grep -n 'ensure_qemu_binary' ob | grep -v '^#'`
  - Expected: 所有调用点仍指向 `ensure_qemu_binary`（分派器），新函数 `ensure_qemu_binary_community` 和 `ensure_qemu_binary_custom` 只在内部调用

### Task 5: 新增 `find_ast2700_bootloaders()` 函数

- 目标：从 deploy 目录自动查找 AST2700 所需的 4 个 bootloader 文件
- 涉及文件：`ob`
- 验证范围：对 b865g8-bytedance 机器能找到全部 4 个文件并输出路径

- [ ] Step 1: 在 `ensure_qemu_binary_custom()` 之后插入新函数 `find_ast2700_bootloaders()`

  函数逻辑：
  1. 输入：deploy 目录 `$BUILD_DIR/tmp/deploy/images/$MACHINE`
  2. 按固定文件名查找：
     - `BOOTLOADER_UBOOT_NODTB="$deploy_dir/u-boot-nodtb.bin"`
     - `BOOTLOADER_UBOOT_DTB="$deploy_dir/u-boot.dtb"`
     - `BOOTLOADER_BL31="$deploy_dir/bl31.bin"`
     - `BOOTLOADER_OPTEE="$deploy_dir/optee/tee-raw.bin"`
  3. 逐个验证文件存在（`-f`），缺失则 error 退出并提示具体缺哪个
  4. 导出全局变量：`BOOTLOADER_UBOOT_NODTB`、`BOOTLOADER_UBOOT_DTB`、`BOOTLOADER_BL31`、`BOOTLOADER_OPTEE`
  5. 获取 u-boot-nodtb.bin 的文件大小（`stat --format=%s`），导出 `BOOTLOADER_UBOOT_SIZE`（用于计算 u-boot.dtb 的加载偏移）

- Change: 插入新函数

- [ ] Step 2: 验证语法

  - Run: `bash -n ob`
  - Expected: `syntax OK`

### Task 6: 新增 `build_qemu_cmd()` 函数（SoC 模板分派）

- 目标：根据 SoC 类型构建完整的 QEMU 命令数组，替代原来散落在 community/custom 两个函数里的重复命令构造
- 涉及文件：`ob`
- 验证范围：对 AST2700 生成包含 `-device loader` 链的命令；对 AST2600 生成简单命令

- [ ] Step 1: 在 `find_ast2700_bootloaders()` 之后插入新函数 `build_qemu_cmd()`

  函数签名：`build_qemu_cmd <image_file> <ssh_port> <redfish_port> <ipmi_port> <http_port> <serial_log>`

  共享部分（所有 SoC）：
  ```bash
  local image_file="$1" ssh_port="$2" redfish_port="$3" ipmi_port="$4" http_port="$5" serial_log="$6"

  # Port forwarding
  local hostfwd_args=""
  hostfwd_args+="hostfwd=tcp::${ssh_port}-:22,"
  hostfwd_args+="hostfwd=tcp::${redfish_port}-:443,"
  hostfwd_args+="hostfwd=udp::${ipmi_port}-:623"
  if [[ -n "$http_port" ]]; then
      hostfwd_args+=",hostfwd=tcp::${http_port}-:80"
  fi

  QEMU_CMD=(
      "$QEMU_BIN_FILE"
      "-machine" "$QB_MACHINE_NAME"
  )
  ```

  SoC 分支：

  **AST2700 分支**：
  ```bash
  if [[ "$SOC_TYPE" == "ast2700" ]]; then
      find_ast2700_bootloaders
      QEMU_CMD+=(
          "-device" "loader,force-raw=on,addr=0x400000000,file=$BOOTLOADER_UBOOT_NODTB"
          "-device" "loader,force-raw=on,addr=$((0x400000000 + BOOTLOADER_UBOOT_SIZE)),file=$BOOTLOADER_UBOOT_DTB"
          "-device" "loader,force-raw=on,addr=0x430000000,file=$BOOTLOADER_BL31"
          "-device" "loader,force-raw=on,addr=0x430080000,file=$BOOTLOADER_OPTEE"
          "-device" "loader,cpu-num=0,addr=0x430000000"
          "-device" "loader,cpu-num=1,addr=0x430000000"
          "-device" "loader,cpu-num=2,addr=0x430000000"
          "-device" "loader,cpu-num=3,addr=0x430000000"
          "-smp" "4"
      )
  fi
  ```

  **AST2600 分支**：无额外参数（bootloader 已打包在 MTD image 中）

  **公共尾部**：
  ```bash
  # QB_MEM（-m 参数）：有则加，无则不加
  if [[ -n "$QB_MEM_SIZE_FLAG" ]]; then
      local -a qemu_mem_args=()
      read -r -a qemu_mem_args <<< "$QB_MEM_SIZE_FLAG"
      QEMU_CMD+=("${qemu_mem_args[@]}")
  fi

  QEMU_CMD+=(
      "-drive" "file=$image_file,format=raw,if=mtd"
      "-net" "nic,netdev=net0"
      "-netdev" "user,id=net0,$hostfwd_args"
      "-serial" "file:$serial_log"
      "-serial" "null"
      "-monitor" "none"
      "-display" "none"
  )
  if [[ -d "$QEMU_PCBIOS_DIR" ]]; then
      QEMU_CMD+=("-L" "$QEMU_PCBIOS_DIR")
  fi
  QEMU_CMD+=("-daemonize")
  ```

- Change: 插入新函数

- [ ] Step 2: 验证语法

  - Run: `bash -n ob`
  - Expected: `syntax OK`

### Task 7: 重构 `cmd_start_qemu()` 统一流程

- 目标：将 `cmd_start_qemu_community()` + `cmd_start_qemu_custom()` 合并进 `cmd_start_qemu()`，消除重复，用新的 SoC 模板驱动启动
- 涉及文件：`ob`（修改 `cmd_start_qemu` L2621-2774，删除 `cmd_start_qemu_community` L2780-2849 和 `cmd_start_qemu_custom` L2855-2923）
- 验证范围：`ob start-qemu --dry-run` 能生成正确的 QEMU 命令

- [ ] Step 1: 修改 `cmd_start_qemu()` 的后半段（L2679 起，`resolve_qb_vars` 调用之后）

  新流程替换从 `resolve_qb_vars` 到函数末尾的部分：

  ```bash
  # 原：resolve_qb_vars
  resolve_qb_vars
  # 新增：SoC 检测（补充 QB_ 变量的回退）
  detect_soc_type
  # 新增：QEMU machine 名推导
  derive_qemu_machine_name
  # 原逻辑保留：二进制获取（分派到 community 下载或 custom 本地拷贝）
  ensure_qemu_binary
  # 原逻辑保留：固件检查
  ensure_qemu_firmware
  ```

  后续部分（image 查找、端口检测、现有实例检查）保持不变。

  在原有实例检查之后、原分派逻辑（`if community ... else ...`）的位置，替换为：

  ```bash
  # ── Build QEMU command (SoC-specific template) ──
  build_qemu_cmd "$image_file" "$ssh_port" "$redfish_port" "$ipmi_port" "$http_port" "$serial_log"

  step_header "Starting QEMU for '$MACHINE' ($SOC_TYPE)"
  echo "  Machine   : $QB_MACHINE_NAME"
  echo "  SoC       : $SOC_TYPE"
  echo "  Binary    : $QEMU_BIN_FILE"
  echo "  Image     : $image_file"
  echo "  Serial log: $serial_log"
  echo ""

  verbose "Command: setsid ${QEMU_CMD[*]}"

  if [[ "$DRY_RUN" -eq 1 ]]; then
      info "[DRY-RUN] Would run: setsid ${QEMU_CMD[*]}"
      exit 0
  fi

  local qemu_stderr
  qemu_stderr=$(mktemp "${TMPDIR:-/tmp}/qemu-stderr-XXXXXX")
  if ! setsid "${QEMU_CMD[@]}" >"$qemu_stderr" 2>&1; then
      error "QEMU failed to start."
      local qemu_err_msg
      qemu_err_msg=$(grep -v "^qemu-system.*: warning:" "$qemu_stderr" 2>/dev/null || true)
      if [[ -n "$qemu_err_msg" ]]; then
          error "$(echo "$qemu_err_msg" | head -5)"
      fi
      error "Check serial log: $serial_log"
      error "Verify QEMU binary: $QEMU_BIN_FILE"
      rm -f "$qemu_stderr"
      exit 1
  fi
  rm -f "$qemu_stderr"

  _qemu_post_launch "$QB_MACHINE_NAME" "$ssh_port" "$redfish_port" "$ipmi_port" "$http_port" "$serial_log"
  ```

- [ ] Step 2: 删除 `cmd_start_qemu_community()` 和 `cmd_start_qemu_custom()` 两个函数（L2780-2923 整段删除）

- [ ] Step 3: 修改 `resolve_qb_vars()` 使其在 QB_ 变量缺失时不报错退出，而是设为空

  修改点（`ob` 中 `resolve_qb_vars` 函数体，约 L679-767）：
  - `QB_MACHINE` 缺失时：不再 `error + exit`，改为 warn 并留空 `QB_MACHINE_NAME=""`
  - `QB_MEM` 缺失时：不再 `error + exit`，改为 verbose 输出并留空 `QB_MEM_SIZE_FLAG=""`
  - `QB_SYSTEM_NAME` 缺失时：不再 `error + exit`，改为 warn 并留空 `QB_SYSTEM_NAME=""`（后续由 `detect_soc_type` 回退填充）

- [ ] Step 4: 验证语法

  - Run: `bash -n ob`
  - Expected: `syntax OK`

### Task 8: 简化 `ensure_qemu_firmware()`

- 目标：移除 `ensure_qemu_firmware()` 中的 vbootrom stub 生成逻辑（AST2700 的 vbootrom 已由用户提供的 pc-bios 包含），简化为 pc-bios 目录存在性检查
- 涉及文件：`ob`（修改 `ensure_qemu_firmware` 函数，L661-677）
- 验证范围：当 pc-bios 目录存在时正常返回，不存在时给出提示

- [ ] Step 1: 简化函数体

  ```bash
  ensure_qemu_firmware() {
      QEMU_PCBIOS_DIR="$QEMU_BIN_DIR/pc-bios"
      if [[ ! -d "$QEMU_PCBIOS_DIR" ]]; then
          warn "QEMU pc-bios directory not found: $QEMU_PCBIOS_DIR"
          warn "Custom QEMU requires pc-bios/. Ensure it was provided during binary setup."
      fi
  }
  ```

- [ ] Step 2: 验证语法

  - Run: `bash -n ob`
  - Expected: `syntax OK`

### Task 9: 全流程集成验证

- 目标：验证重构后 `ob start-qemu` 的完整流程（dry-run 模式）
- 涉及文件：`ob`
- 验证范围：dry-run 输出包含正确的 SoC 检测结果、QEMU 命令模板和所有参数

- [ ] Step 1: 语法检查

  - Run: `bash -n ob`
  - Expected: 无输出（语法正确）

- [ ] Step 2: 验证帮助信息

  - Run: `ob start-qemu --help 2>&1 | head -30`
  - Expected: 正常输出 start-qemu 的 usage 信息，无报错

- [ ] Step 3: 验证新函数存在、旧函数已删除

  ```bash
  for fn in detect_soc_type derive_qemu_machine_name ensure_qemu_binary_custom find_ast2700_bootloaders build_qemu_cmd; do
      grep -q "^${fn}()" ob && echo "PASS: $fn" || echo "FAIL: $fn"
  done
  for fn in cmd_start_qemu_community cmd_start_qemu_custom; do
      grep -q "^${fn}()" ob && echo "FAIL: $fn still exists" || echo "PASS: $fn removed"
  done
  ```
  - Expected: 全部 PASS

- [ ] Step 4: 验证 b865g8-bytedance machine 的 SoC 检测和命令构建（dry-run）

  ```bash
  ob start-qemu b865g8-bytedance --dry-run 2>&1 | head -20
  ```
  - Expected: 输出包含 `ast2700` SoC 检测结果和正确的 `qemu-system-aarch64` 命令（含 `-device loader` 链）

- [ ] Step 5: Checkpoint commit

  ```bash
  git add ob && git commit -m "refactor(ob): 重构 QEMU 启动流程，支持 SoC 模板和 custom 路径本地二进制"
  ```

## 执行纪律

- 开始实现前，先批判性复查整份计划；如果发现缺项、矛盾、命名不一致或验证命令无效，先修计划
- 按任务顺序执行（Task 1 → Task 9），不要无声跳步、合并步或改变任务目标
- 每完成一个任务，都运行该任务定义的验证（`bash -n ob` + 该任务的特定检查）
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下来说明，不要猜
- 当前分支 `feat/start-qemu`，非 main 分支，无需额外确认
- 全部任务完成后，运行 Task 9 的最终验证并输出修改摘要

## 最终验证

```bash
# 1. 语法
bash -n ob && echo "PASS: syntax" || echo "FAIL: syntax"

# 2. 帮助
ob start-qemu --help 2>&1 && echo "PASS: help" || echo "FAIL: help"

# 3. Dry-run for custom AST2700 machine
ob start-qemu b865g8-bytedance --dry-run 2>&1 | grep -E '(ast2700|qemu-system-aarch64|loader)' && echo "PASS: AST2700 template" || echo "FAIL: AST2700 template"

# 4. 函数存在性检查
for fn in detect_soc_type derive_qemu_machine_name ensure_qemu_binary_custom find_ast2700_bootloaders build_qemu_cmd; do
    grep -q "^${fn}()" ob && echo "PASS: $fn exists" || echo "FAIL: $fn missing"
done

# 5. 旧函数已删除
for fn in cmd_start_qemu_community cmd_start_qemu_custom; do
    grep -q "^${fn}()" ob && echo "FAIL: $fn still exists" || echo "PASS: $fn removed"
done
```

预期结果：全部 PASS。
