# ob start-qemu QEMU binary URL 配置化实施计划

## 目标

在 `ob` 脚本中，把 QEMU binary 的下载地址从「community 硬编码 jenkins + custom 靠 `meta-*/qemu-binary-url.sh` glob」改造为「按 `<source_label>.<QB_SYSTEM_NAME>` 维度落到 `workspace/qemu-bin/qemu-binary-urls.conf` 配置文件」，并让 `ensure_qemu_binary` 按架构（`qemu-system-arm` / `qemu-system-aarch64`）选择 binary、按优先级（env > 配置 > 自动填/交互输入）确定 URL；同时修复 `source_label` 在 init 侧从未被正确推导的问题。设计决策已批准，记录在 [docs/specs/2026-06-10-qemu-binary-url-config-design.md](docs/specs/2026-06-10-qemu-binary-url-config-design.md)。

## 架构快照

改造集中在单文件 `ob`，沿用现有函数组织与 `info/error/verbose/warn` 辅助、`trim_whitespace`、`normalize_repo_url`、`read_source_label`、`PROMPT_PREFIX` 等既有设施。核心：

- **架构来源**：`resolve_qb_vars` 通过 `bitbake -e` 新增解析 `QB_SYSTEM_NAME`（缺失即 `exit 1`，遵循 ADR 0002 无 fallback 约定），值为 `qemu-system-arm`（AST2600，如 romulus）或 `qemu-system-aarch64`（AST2700，如 astar-zzz）。
- **binary 落点**：`derive_qemu_paths` 由硬编码 `qemu-bin/<source>/qemu-system-arm` 改为 `qemu-bin/<source>/<QB_SYSTEM_NAME>`，使 arm/aarch64 在同一 source 下共存。
- **URL 配置**：新增 `qemu-binary-urls.conf`，key 为 `<source_label>.<QB_SYSTEM_NAME>`，由新增 `read_qemu_url_config` / `write_qemu_url_config`（+ `derive_qemu_url_config_path`）读写（upsert）。
- **URL 优先级**：`ensure_qemu_binary` 重构为 env(`OB_QEMU_BINARY_URL`) > 配置文件 > 自动填（community+arm 用 jenkins 默认并写回）/ 交互输入（custom 任意架构；community+aarch64 先打印告知再走交互）。非 tty 不 `read`，改打印可执行报错；空输入干净退出。
- **init 侧修复**：新增 `derive_source_label`，在 `write_source_lock` 写入前调用，按 `normalize_repo_url "$OPENBMC_REPO_URL"` 是否**精确等于** `github.com/openbmc/openbmc` 推导 `community`/`custom`。
- **调用顺序**：`cmd_start_qemu` 把 `resolve_qb_vars` 提前到 `ensure_qemu_binary` 之前（因为后者现在依赖 `QB_SYSTEM_NAME`），并从两个分发函数 `cmd_start_qemu_community` / `cmd_start_qemu_custom` 中移除各自的 `resolve_qb_vars` 与 `local QB_*` 声明。

> 说明：设计文档「影响范围」表把 start-qemu 简化成单点 `cmd_start_qemu`；实际是 `cmd_start_qemu`（主流程）+ `cmd_start_qemu_community` + `cmd_start_qemu_custom`（两个几乎相同的分发函数，仅 step_header 文案不同，QB 变量靠 bash 动态作用域共享）。本计划按真实三函数结构展开。

## 文件结构与职责

- Modify: `ob` — 全局变量区新增 `QB_MACHINE_NAME` / `QB_MEM_SIZE_FLAG` / `QB_SYSTEM_NAME` / `QEMU_URL_CONFIG_FILE`；新增 `derive_source_label`、`derive_qemu_url_config_path`、`read_qemu_url_config`、`write_qemu_url_config`；修改 `write_source_lock`、`resolve_qb_vars`、`derive_qemu_paths`、`ensure_qemu_binary`、`cmd_start_qemu`、`cmd_start_qemu_community`、`cmd_start_qemu_custom`
- Create（运行时）: `workspace/qemu-bin/qemu-binary-urls.conf`
- 不改：QEMU 命令行参数、端口转发、PID 管理、`cmd_stop_qemu`、`cmd_status`（`derive_qemu_paths` 仅被 `ensure_qemu_binary`[L439] 和 `cmd_start_qemu`[L2529] 两处调用，stop/status 不调用，故 arch 化不影响它们）

## 环境前提

- 验证命令为 bash（当前环境 Linux）。`bash -n`、`grep`、`sed`、`mktemp` 可用。
- 涉及 `bitbake -e` 的运行时验证较慢且要求对应 machine 已 `ob init`（构建环境健康）；这些标记为「运行时验证（需构建环境）」。
- 不依赖 `bitbake` 的函数级单元测试（用 `sed` 抽函数 + `eval` 在隔离 shell 调用）是**必过门禁**，应优先执行。
- 当前 workspace 的 OpenBMC 源是企业源（`git.example.com/...`），全局推导为 `custom`；故 astar-zzz 与 romulus 都会走 custom 交互输入路径。community 自动填分支只有在用 `github.com/openbmc/openbmc` 源 init 的副本上才会触发。

## 任务清单

### Task 1: 全局变量区新增 QB / URL 配置变量

- 目标：为后续 Task 提供全局声明，避免 `set -u` 下未绑定变量报错，并把原先散落在分发函数里的 `QB_MACHINE_NAME` / `QB_MEM_SIZE_FLAG` 提升为全局（供提前调用的 `resolve_qb_vars` 写、分发函数读）
- 涉及文件：`ob`（全局变量 QEMU 区，`QEMU_PID_FILE=""` 之后，约 L39）
- 验证范围：`bash -n` 通过；新变量已声明

- [ ] Step 1: 确认当前全局区无这些变量
- Run: `grep -c 'QB_SYSTEM_NAME\|QEMU_URL_CONFIG_FILE' /bmc/iasi/ob-harness/ob`
- Expected: 输出 `0`
- [ ] Step 2: 在 `QEMU_PID_FILE=""` 行（约 L39）之后追加

  # QEMU QB vars (resolved via bitbake -e) + binary URL config
  QB_MACHINE_NAME=""        # derived: <name> from "-machine <name>"
  QB_MEM_SIZE_FLAG=""       # derived: "-m <size>"
  QB_SYSTEM_NAME=""         # derived: qemu-system-arm | qemu-system-aarch64
  QEMU_URL_CONFIG_FILE=""   # derived: workspace/qemu-bin/qemu-binary-urls.conf

  - Change: 在 QEMU 全局变量块末尾追加 4 个声明
- [ ] Step 3: 同步把 `QEMU_BIN_FILE` 行（约 L37）的注释 `qemu-system-arm` 改为 `<QB_SYSTEM_NAME>`，避免误导
  - Change: 修改注释文本
- [ ] Step 4: 语法与声明检查
- Run: `bash -n /bmc/iasi/ob-harness/ob && grep -c 'QB_SYSTEM_NAME\|QEMU_URL_CONFIG_FILE' /bmc/iasi/ob-harness/ob`
- Expected: 无语法错误；计数 `>= 2`

### Task 2: 新增 `derive_source_label` 并接入 `write_source_lock`（修复 init 侧）

- 目标：让 `openbmc-source.lock` 的 `source_label` 由 origin URL 确定性推导，而非保持空/默认
- 涉及文件：`ob`（`write_source_lock()` L822；新增函数置于其前）
- 验证范围：函数级单元测试通过（community/custom/镜像三例）

- [ ] Step 1: 确认尚无该函数
- Run: `grep -c 'derive_source_label' /bmc/iasi/ob-harness/ob`
- Expected: 输出 `0`
- [ ] Step 2: 在 `write_source_lock()`（L822）之前新增函数

  derive_source_label() {
      local normalized
      normalized="$(normalize_repo_url "$OPENBMC_REPO_URL")"
      if [[ "$normalized" == "github.com/openbmc/openbmc" ]]; then
          SOURCE_LABEL="community"
      else
          SOURCE_LABEL="custom"
      fi
  }

  - 注意：比较目标是**完整 `host/path`** `github.com/openbmc/openbmc`（`DEFAULT_OPENBMC_REPO_URL` 经 `normalize_repo_url` 去 scheme/去 `.git`/小写后的结果），**不是** `github.com/openbmc`
  - Change: 新增函数
- [ ] Step 3: 在 `write_source_lock` 内、`timestamp="$(date ...)"` 之后、`if [[ "$DRY_RUN" -eq 1 ]]` 之前插入一行 `derive_source_label`，使 dry-run 与实写两条路径都反映推导结果
  - Change: 在 `write_source_lock` 计算 `normalized_source`/`timestamp` 之后插入 `derive_source_label` 调用
- [ ] Step 4: 函数级单元测试（必过门禁，不依赖 bitbake / 现有 lock）
- Run:
  cd /bmc/iasi/ob-harness && eval "$(sed -n '/^normalize_repo_url() {/,/^}/p; /^derive_source_label() {/,/^}/p' ob)" && \
  OPENBMC_REPO_URL="https://github.com/openbmc/openbmc.git"; derive_source_label; echo "1 expect community: $SOURCE_LABEL"; \
  OPENBMC_REPO_URL="git@git.example.com:corp/openbmc-ami/base-tech/openbmc.git"; derive_source_label; echo "2 expect custom: $SOURCE_LABEL"; \
  OPENBMC_REPO_URL="ssh://git@git.openbmc.org/openbmc/openbmc.git"; derive_source_label; echo "3 expect custom: $SOURCE_LABEL"
- Expected:
  1 expect community: community
  2 expect custom: custom
  3 expect custom: custom
- 说明：本 workspace 中 openbmc 仓库已存在，`verify_source` 仅在 lock 缺失时写锁、`clone_openbmc` 命中 `.git` 即提前返回，二者都不会触发 `write_source_lock`，所以**不能**用 `ob init -d` dry-run 验证本任务；隔离函数测试是唯一可靠门禁。

### Task 3: 新增 `qemu-binary-urls.conf` 读写函数

- 目标：提供按 `<source_label>.<QB_SYSTEM_NAME>` key 的 URL upsert/读取能力
- 涉及文件：`ob`（新增 3 个函数，置于 `derive_qemu_paths()` L429 附近）
- 验证范围：函数级单元测试通过（写、覆盖、读、未命中返回空）

- [ ] Step 1: 确认尚无这些函数
- Run: `grep -c 'read_qemu_url_config\|write_qemu_url_config\|derive_qemu_url_config_path' /bmc/iasi/ob-harness/ob`
- Expected: 输出 `0`
- [ ] Step 2: 新增三个函数（建议置于 `derive_qemu_paths` 之后）

  derive_qemu_url_config_path() {
      QEMU_URL_CONFIG_FILE="$WORKSPACE_DIR/qemu-bin/qemu-binary-urls.conf"
  }

  # read_qemu_url_config <source> <arch>  → echo URL（无则空）
  read_qemu_url_config() {
      local source="$1" arch="$2" key key_re url
      key="${source}.${arch}"
      key_re="${key//./\\.}"
      derive_qemu_url_config_path
      [[ -f "$QEMU_URL_CONFIG_FILE" ]] || return 0
      url=$(grep "^${key_re}=" "$QEMU_URL_CONFIG_FILE" 2>/dev/null | tail -1 | cut -d= -f2-)
      trim_whitespace "$url"
  }

  # write_qemu_url_config <source> <arch> <url>  → upsert
  write_qemu_url_config() {
      local source="$1" arch="$2" url="$3" key key_re tmp
      key="${source}.${arch}"
      key_re="${key//./\\.}"
      derive_qemu_url_config_path
      mkdir -p "$(dirname "$QEMU_URL_CONFIG_FILE")"
      if [[ ! -f "$QEMU_URL_CONFIG_FILE" ]]; then
          echo "# qemu binary download URLs — auto-managed by 'ob start-qemu'" > "$QEMU_URL_CONFIG_FILE"
          echo "# key: <source_label>.<QB_SYSTEM_NAME>" >> "$QEMU_URL_CONFIG_FILE"
      fi
      if grep -q "^${key_re}=" "$QEMU_URL_CONFIG_FILE"; then
          tmp=$(mktemp "${TMPDIR:-/tmp}/qemu-url-conf-XXXXXX")
          grep -v "^${key_re}=" "$QEMU_URL_CONFIG_FILE" > "$tmp"
          echo "${key}=${url}" >> "$tmp"
          mv "$tmp" "$QEMU_URL_CONFIG_FILE"
      else
          echo "${key}=${url}" >> "$QEMU_URL_CONFIG_FILE"
      fi
  }

  - 注意：key 含 `.`，grep 前用 `key_re="${key//./\\.}"` 转义，避免 `.` 作为正则通配误命中
  - Change: 新增三个函数
- [ ] Step 3: 函数级单元测试（必过门禁）
- Run:
  cd /bmc/iasi/ob-harness && tmpws=$(mktemp -d "${TMPDIR:-/tmp}/obws-XXXXXX") && \
  eval "$(sed -n '/^trim_whitespace() {/,/^}/p; /^derive_qemu_url_config_path() {/,/^}/p; /^read_qemu_url_config() {/,/^}/p; /^write_qemu_url_config() {/,/^}/p' ob)" && \
  WORKSPACE_DIR="$tmpws"; \
  write_qemu_url_config custom qemu-system-aarch64 "https://example/aarch64"; \
  write_qemu_url_config community qemu-system-arm "https://example/arm"; \
  write_qemu_url_config custom qemu-system-aarch64 "https://example/aarch64-v2"; \
  echo "read1 expect v2: [$(read_qemu_url_config custom qemu-system-aarch64)]"; \
  echo "read2 expect arm: [$(read_qemu_url_config community qemu-system-arm)]"; \
  echo "miss  expect empty: [$(read_qemu_url_config custom qemu-system-arm)]"; \
  echo "--- conf ---"; cat "$tmpws/qemu-bin/qemu-binary-urls.conf"; rm -rf "$tmpws"
- Expected:
  read1 expect v2: [https://example/aarch64-v2]
  read2 expect arm: [https://example/arm]
  miss  expect empty: []
  且 conf 中 `custom.qemu-system-aarch64` 只出现一行（已被 upsert 覆盖）

### Task 4: `resolve_qb_vars` 增解析 `QB_SYSTEM_NAME`

- 目标：从 `bitbake -e` 输出解析架构对应的 QEMU 二进制名
- 涉及文件：`ob`（`resolve_qb_vars()` L574）
- 验证范围：`bash -n` 通过；解析与 reset 代码就位；（运行时）astar 得 `qemu-system-aarch64`

- [ ] Step 1: 确认尚未解析该变量
- Run: `grep -c 'QB_SYSTEM_NAME=' /bmc/iasi/ob-harness/ob`
- Expected: 输出 `0`
- [ ] Step 2: 在 reset 块（`QB_MACHINE_NAME=""` / `QB_MEM_SIZE_FLAG=""`，约 L598-599）追加一行 `QB_SYSTEM_NAME=""`
  - Change: reset 块加一行
- [ ] Step 3: 在 `QB_MEM_SIZE_FLAG="$qb_mem_raw"`（约 L640）之后、`verbose "  QB_MACHINE → ..."` 之前，追加解析块

  # Extract QB_SYSTEM_NAME (qemu-system-arm | qemu-system-aarch64)
  local qb_system_raw
  qb_system_raw=$(echo "$bitbake_output" | grep '^QB_SYSTEM_NAME=' | head -1 | cut -d= -f2-)
  qb_system_raw=$(trim_whitespace "$qb_system_raw")
  qb_system_raw="${qb_system_raw#\"}"; qb_system_raw="${qb_system_raw%\"}"
  qb_system_raw="${qb_system_raw#\'}"; qb_system_raw="${qb_system_raw%\'}"
  if [[ -z "$qb_system_raw" ]]; then
      error "QB_SYSTEM_NAME not defined for machine '$MACHINE'."
      error "Ensure the machine conf inherits qemuboot (expected qemu-system-arm or qemu-system-aarch64)."
      exit 1
  fi
  QB_SYSTEM_NAME="$qb_system_raw"

  - Change: 新增解析块；并在末尾 verbose 区追加 `verbose "  QB_SYSTEM_NAME → $QB_SYSTEM_NAME"`
- [ ] Step 4: 语法检查
- Run: `bash -n /bmc/iasi/ob-harness/ob && grep -c 'QB_SYSTEM_NAME' /bmc/iasi/ob-harness/ob`
- Expected: 无语法错误；计数 `>= 3`（全局声明 + reset + 解析/verbose）
- [ ] Step 5（运行时验证，需构建环境）: 对已 init 的 machine 确认解析值
- Run: `cd /bmc/iasi/ob-harness/workspace/openbmc && set +u; source setup astar-zzz build/astar-zzz >/dev/null 2>&1 && bitbake -e 2>/dev/null | grep '^QB_SYSTEM_NAME='`
- Expected: `QB_SYSTEM_NAME="qemu-system-aarch64"`（romulus 则为 `qemu-system-arm`）。若环境未 init，跳过本步并说明

### Task 5: `derive_qemu_paths` 按架构派生 binary 路径

- 目标：binary 落点改为 `qemu-bin/<source>/<QB_SYSTEM_NAME>`
- 涉及文件：`ob`（`derive_qemu_paths()` L429，`QEMU_BIN_FILE` 赋值 L433）
- 验证范围：`bash -n` 通过；路径使用 `QB_SYSTEM_NAME`

- [ ] Step 1: 确认当前为硬编码
- Run: `sed -n '429,435p' /bmc/iasi/ob-harness/ob`
- Expected: 看到 `QEMU_BIN_FILE="$QEMU_BIN_DIR/qemu-system-arm"`
- [ ] Step 2: 将 `QEMU_BIN_FILE` 赋值改为基于架构（带 fallback，供 stop/status 等未解析 QB 的上下文防御性使用）

  derive_qemu_paths() {
      local label arch
      label=$(read_source_label)
      arch="${QB_SYSTEM_NAME:-qemu-system-arm}"
      QEMU_BIN_DIR="$WORKSPACE_DIR/qemu-bin/$label"
      QEMU_BIN_FILE="$QEMU_BIN_DIR/$arch"
      QEMU_PIDS_DIR="$WORKSPACE_DIR/qemu-bin/.pids"
      QEMU_PID_FILE="$QEMU_PIDS_DIR/${MACHINE}.pid"
  }

  - Change: 修改 `local` 行与 `QEMU_BIN_FILE` 赋值行
- [ ] Step 3: 语法与内容检查
- Run: `bash -n /bmc/iasi/ob-harness/ob && grep -c 'QEMU_BIN_FILE="$QEMU_BIN_DIR/$arch"' /bmc/iasi/ob-harness/ob`
- Expected: 无语法错误；计数 `1`

### Task 6: 重构 `ensure_qemu_binary` URL 解析与下载

- 目标：按 env > 配置 > 自动填/交互 确定 URL；community+aarch64 告知后走交互；非 tty 报可执行错误；空输入干净退出；统一下载并应用附带修复
- 涉及文件：`ob`（`ensure_qemu_binary()` L438-L572）
- 验证范围：`bash -n` 通过；旧 meta glob 移除；附带修复就位；新行为字符串可被 grep 命中

- [ ] Step 1: 确认当前依赖 meta glob 且 mktemp/curl 为旧写法
- Run: `grep -n 'qemu-binary-url.sh\|mktemp /tmp/qemu-binary\|"\$custom_url" 2>/dev/null' /bmc/iasi/ob-harness/ob`
- Expected: 命中 meta glob、`mktemp /tmp/qemu-binary-XXXXXX`、custom curl 的 `2>/dev/null`
- [ ] Step 2: 用如下结构替换 `ensure_qemu_binary` 中 `mkdir -p "$QEMU_BIN_DIR"`（L450）之后到函数结束 `}`（L572）之间的全部内容（保留 L439-L450 的 `derive_qemu_paths` / already-exists 短路 / `label` / `manifest` / `mkdir`）

      local arch="${QB_SYSTEM_NAME:-qemu-system-arm}"
      local qemu_url=""

      # ── URL priority: env > config > auto-fill/interactive ──
      if [[ -n "${OB_QEMU_BINARY_URL:-}" ]]; then
          qemu_url="$OB_QEMU_BINARY_URL"
          write_qemu_url_config "$label" "$arch" "$qemu_url"
      else
          qemu_url="$(read_qemu_url_config "$label" "$arch")"
      fi

      if [[ -z "$qemu_url" ]]; then
          if [[ "$label" == "community" && "$arch" == "qemu-system-arm" ]]; then
              qemu_url="https://jenkins.openbmc.org/job/latest-qemu-x86/lastSuccessfulBuild/artifact/qemu/build/qemu-system-arm"
              write_qemu_url_config "$label" "$arch" "$qemu_url"
              info "Using OpenBMC Jenkins default QEMU URL (recorded in $QEMU_URL_CONFIG_FILE)"
          else
              if [[ "$label" == "community" && "$arch" == "qemu-system-aarch64" ]]; then
                  info "Community source provides no aarch64 QEMU binary."
                  info "Provide a custom download URL below, or press Enter / Ctrl-C to abort."
              fi
              if [[ ! -t 0 ]]; then
                  error "QEMU binary URL not configured for '${label}.${arch}'."
                  error "Set OB_QEMU_BINARY_URL, or add a line '${label}.${arch}=<url>' to:"
                  error "  $QEMU_URL_CONFIG_FILE"
                  exit 1
              fi
              local _input=""
              read -r -p "$(echo -e "${PROMPT_PREFIX} Enter QEMU binary URL for ${label}.${arch}: ")" _input || true
              _input="$(trim_whitespace "$_input")"
              if [[ -z "$_input" ]]; then
                  info "No URL provided — aborting QEMU binary setup."
                  exit 0
              fi
              if [[ ! "$_input" =~ ^https?:// ]]; then
                  error "Invalid URL (must start with http:// or https://): $_input"
                  exit 1
              fi
              qemu_url="$_input"
              write_qemu_url_config "$label" "$arch" "$qemu_url"
          fi
      fi

      info "Downloading QEMU binary..."
      verbose "  URL: $qemu_url"

      local tmp_download
      tmp_download=$(mktemp "${TMPDIR:-/tmp}/qemu-binary-XXXXXX")
      if ! curl -fSL -o "$tmp_download" "$qemu_url"; then
          error "Failed to download QEMU binary from: $qemu_url"
          rm -f "$tmp_download"
          exit 1
      fi

      # Detect file type: tarball or single binary (reuse existing logic)
      local file_type
      file_type=$(file -b "$tmp_download" 2>/dev/null || echo "")
      if echo "$file_type" | grep -qi "gzip\|xz"; then
          verbose "  Detected archive, extracting..."
          tar xf "$tmp_download" -C "$QEMU_BIN_DIR/" --strip-components=1 2>/dev/null \
              || tar xf "$tmp_download" -C "$QEMU_BIN_DIR/" 2>/dev/null
          rm -f "$tmp_download"
          local found_bin=""
          for candidate in "$QEMU_BIN_DIR/$arch" "$QEMU_BIN_DIR/bin/$arch"; do
              if [[ -f "$candidate" ]]; then found_bin="$candidate"; break; fi
          done
          if [[ -z "$found_bin" ]]; then
              error "Could not find $arch in extracted archive."
              exit 1
          fi
          [[ "$found_bin" == "$QEMU_BIN_FILE" ]] || mv "$found_bin" "$QEMU_BIN_FILE"
          chmod +x "$QEMU_BIN_FILE"
      else
          mv "$tmp_download" "$QEMU_BIN_FILE"
          chmod +x "$QEMU_BIN_FILE"
      fi

      local sha256=""
      sha256=$(sha256sum "$QEMU_BIN_FILE" | awk '{print $1}')
      cat > "$manifest" <<MANIFEST_EOF
  source=$label
  arch=$arch
  url=$qemu_url
  downloaded_at=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  sha256=$sha256
  MANIFEST_EOF

      info "QEMU binary ready: $QEMU_BIN_FILE"
      verbose "  SHA256: $sha256"

  - 这次重构**移除** `meta-*/qemu-binary-url.sh` glob 与原 community/custom 双分支下载，统一为单一下载路径（设计决策 5/6）；manifest 改记 `source` + `arch`，不再记 `jenkins_build`（可接受的轻微取舍，见边界）
  - Change: 替换 `ensure_qemu_binary` 函数体（保留前 5 行 setup）
- [ ] Step 3: 语法 + 移除/修复检查（必过门禁）
- Run:
  cd /bmc/iasi/ob-harness && bash -n ob && echo "syntax OK"; \
  echo "meta-glob (expect 0): $(grep -c 'qemu-binary-url.sh' ob)"; \
  echo "old mktemp /tmp (expect 0): $(grep -c 'mktemp /tmp/qemu-binary' ob)"; \
  echo "TMPDIR mktemp (expect >=1): $(grep -c 'mktemp "\${TMPDIR:-/tmp}/qemu-binary' ob)"; \
  echo "aarch64 notice (expect >=1): $(grep -c 'no aarch64 QEMU binary' ob)"; \
  echo "config helpers used (expect >=2): $(grep -c 'read_qemu_url_config\|write_qemu_url_config' ob)"
```raw
- Expected: `syntax OK`；meta-glob `0`；old mktemp `0`；TMPDIR mktemp `>=1`；aarch64 notice `>=1`；config helpers `>=2`

### Task 7: 调整 `cmd_start_qemu` 调用顺序并清理分发函数

- 目标：把 `resolve_qb_vars` 提前到 `ensure_qemu_binary` 之前（使后者拿到 `QB_SYSTEM_NAME`），并移除两个分发函数里重复的 `resolve_qb_vars` 与 `local QB_*`
- 涉及文件：`ob`（`cmd_start_qemu` L2434 区段；`cmd_start_qemu_community` L2586；`cmd_start_qemu_custom` L2650）
- 验证范围：`bash -n` 通过；`resolve_qb_vars` 调用点从 2 处变为 1 处且位于主流程

- [ ] Step 1: 确认当前 `resolve_qb_vars` 在两个分发函数中各调用一次
- Run: `grep -n 'resolve_qb_vars' /bmc/iasi/ob-harness/ob`
- Expected: 3 处（定义 L574 + community L2597 + custom L2661）
- [ ] Step 2: 在 `cmd_start_qemu` 的「Prerequisite 1: machine init-done」检查之后、「Prerequisite 4: QEMU binary」`ensure_qemu_binary` 调用之前，插入一行 `resolve_qb_vars`
  - Change: 在 `cmd_start_qemu` 主流程插入 `resolve_qb_vars`（位于 init-done 检查与 `ensure_qemu_binary` 之间）
- [ ] Step 3: 从 `cmd_start_qemu_community`（L2586）移除其 `# ── Resolve QB vars ...` 注释、`local QB_MACHINE_NAME=""`、`local QB_MEM_SIZE_FLAG=""`（L2595-2596）与 `resolve_qb_vars`（L2597）
  - Change: 删除该 4 行块（变量改由全局 + 主流程提供）
- [ ] Step 4: 从 `cmd_start_qemu_custom`（L2650）移除同样的注释 + `local QB_MACHINE_NAME=""` + `local QB_MEM_SIZE_FLAG=""`（L2659-2660）+ `resolve_qb_vars`（L2661）
  - Change: 删除该 4 行块
- [ ] Step 5: 语法 + 调用点检查
- Run: `bash -n /bmc/iasi/ob-harness/ob && grep -c 'resolve_qb_vars' /bmc/iasi/ob-harness/ob && grep -c 'local QB_MACHINE_NAME' /bmc/iasi/ob-harness/ob`
- Expected: 无语法错误；`resolve_qb_vars` 计数 `2`（定义 + 主流程唯一调用）；`local QB_MACHINE_NAME` 计数 `0`
- [ ] Step 6: 确认提前调用位置正确（`resolve_qb_vars` 行号应小于 `cmd_start_qemu` 中 `ensure_qemu_binary` 调用行号）
- Run: `grep -n 'resolve_qb_vars\|ensure_qemu_binary' /bmc/iasi/ob-harness/ob | sed -n '1,6p'`
- Expected: 主流程中 `resolve_qb_vars` 调用出现在 `ensure_qemu_binary` 调用之前

### Task 8: 端到端验证（custom aarch64 路径）

- 目标：在当前 custom workspace 下验证整体行为（首次弹菜单写回、二次复用、非 tty 报错）
- 涉及文件：无（仅运行验证）
- 验证范围：依据环境可执行性逐项验证；不可执行项明确标注跳过原因

- [ ] Step 1: 全脚本语法
- Run: `bash -n /bmc/iasi/ob-harness/ob && echo "syntax OK"`
- Expected: `syntax OK`
- [ ] Step 2: 非交互报错（无 tty，配置缺失时不阻塞）。前置：确保 `workspace/qemu-bin/qemu-binary-urls.conf` 无 `custom.qemu-system-aarch64` 且无 `OB_QEMU_BINARY_URL`，binary 未缓存
- Run: `cd /bmc/iasi/ob-harness && printf '' | ./ob start-qemu astar-zzz 2>&1 | grep -i 'not configured\|OB_QEMU_BINARY_URL'; echo "exit=${PIPESTATUS[1]}"`
- Expected: 打印「URL not configured...」可执行报错并提示 `OB_QEMU_BINARY_URL` / 配置文件路径（注：此步要求 astar 已 init 且 `resolve_qb_vars` 能跑通；若未 init，跳过并说明）
- [ ] Step 3（交互，需 tty + 网络）: 人工对 astar 运行，输入可达 URL，确认下载成功且 `qemu-binary-urls.conf` 出现 `custom.qemu-system-aarch64=<URL>`；再次运行不再弹菜单。标注为人工验证项
- [ ] Step 4: 最终整体校验（见「最终验证」）

## 执行纪律

- 开始实现前，先批判性复查整份计划；如发现缺项、矛盾、命名不一致或验证命令无效，先修计划
- 按任务顺序执行，不要无声跳步、合并步或改变任务目标
- 每完成一个任务，都运行该任务定义的验证；函数级单元测试为必过门禁，bitbake 运行时验证按环境可用性执行
- 编辑后务必 `bash -n ob` 复核；本仓库 `.md` 文件用编辑工具有静默失败史，但本计划只改 `ob`（非 .md），仍建议改后 `grep`/`sed` 复核磁盘内容
- 遇到阻塞、重复失败或计划与仓库现实不符，立即停下来说明，不要猜
- 如果当前在 `main` 分支且用户未明确同意，开始实现前先确认或新建 feature 分支
- 不改 QEMU 命令行参数、端口转发、PID 管理逻辑；不引入 per-machine source 属性；`source_label` 维持 workspace 全局
- 全部任务完成后，运行最终验证并输出修改摘要

## 最终验证

- Run: `bash -n /bmc/iasi/ob-harness/ob && echo "syntax OK"`
- Expected: `syntax OK`
- Run: `grep -c 'derive_source_label\|derive_qemu_url_config_path\|read_qemu_url_config\|write_qemu_url_config' /bmc/iasi/ob-harness/ob`
- Expected: `>= 8`（4 个新函数：各 1 处定义 + 至少 1 处调用）
- Run: `grep -c 'qemu-binary-url.sh' /bmc/iasi/ob-harness/ob`
- Expected: `0`（旧 meta glob 已移除）
- Run: `grep -c 'mktemp /tmp/qemu-binary' /bmc/iasi/ob-harness/ob`
- Expected: `0`（已改用 `${TMPDIR:-/tmp}`）
- Run: `grep -n 'resolve_qb_vars' /bmc/iasi/ob-harness/ob | wc -l`
- Expected: `2`（定义 + 主流程唯一调用）
- Run: `grep -c 'local QB_MACHINE_NAME' /bmc/iasi/ob-harness/ob`
- Expected: `0`（分发函数不再各自声明）
- Run: Task 2 与 Task 3 的函数级单元测试
- Expected: 全部断言符合预期

```