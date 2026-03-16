# Baseline Rollback And Memory Rework Checklist
# 基线回退与低风险内存重做执行清单

日期 / Date: 2026-03-16

## Goal / 目标

- Restore a known-good validation baseline before `512b05271`.
- 回到 `512b05271` 之前的可验证稳定基线。
- Keep current work isolated from rollback verification.
- 把当前工作线与回退验证线隔离，避免互相污染。
- Rebuild memory optimizations in low-risk layers only after the baseline is validated.
- 只有在基线验证通过后，才按低风险顺序重新做内存优化。

## Branch Plan / 分支规划

### Branch A: strict pre-512 baseline / 严格 pre-512 基线

- Branch name: `x/strict-pre-512-baseline`
- Source: `bbb842ce4` which is the parent of `512b05271`
- Purpose: verify behavior on the exact historical baseline before the risky memory container change
- 用途：验证“风险内存容器改动之前”的精确历史基线行为

### Branch B: minimal revert on current line / 当前线最小回退验证分支

- Branch name: `x/current-line-revert-memory-wallet`
- Source: current `HEAD`
- Revert commits in this order:
- 按以下顺序回退提交：
  1. `8a3a711bf`
  2. `786714e17`
  3. `be6ee3f26`
  4. `512b05271`
- Purpose: verify whether the current line becomes stable after removing the risky memory change and its wallet/UI fallback patches
- 用途：验证当前线上去掉风险内存改动及其钱包/UI 补丁后，是否恢复稳定

## Execution Order / 执行顺序

1. Create this checklist file and keep it on the current working branch.
1. 先把本清单文件落盘，保留在当前工作分支。
2. Create `x/strict-pre-512-baseline` at `bbb842ce4`.
2. 以 `bbb842ce4` 创建 `x/strict-pre-512-baseline`。
3. Create `x/current-line-revert-memory-wallet` from current `HEAD`.
3. 以当前 `HEAD` 创建 `x/current-line-revert-memory-wallet`。
4. On Branch B, revert `8a3a711bf`, `786714e17`, `be6ee3f26`, `512b05271`.
4. 在 Branch B 上按顺序回退 `8a3a711bf`、`786714e17`、`be6ee3f26`、`512b05271`。
5. Record both branch pointers for verification.
5. 记录两个分支的指针，供后续验证使用。
6. Only after baseline verification passes, start low-risk memory rework.
6. 只有基线验证通过后，才开始低风险内存重做。

## Verification Gate / 验收门槛

- Wallet open with existing wallet data / 用现有钱包数据打开钱包
- Receive transaction / 收款
- Send transaction / 转出
- Restart and send again / 重启后再次转出
- Transaction detail page / 交易详情页
- `gettransaction`
- `dumpwallet`
- `importdescriptors`
- `rescan`

Any failure below blocks the optimization from entering the stable line:
以下任一失败都阻止优化进入稳定线：

- missing `findBlock` / `findAncestorByHeight` lookups
- `findBlock` / `findAncestorByHeight` 查找缺失
- abandoned or misclassified incoming transactions
- 流入交易被错误标记为 abandoned 或状态错误
- receive/send regressions
- 收款/转出回归
- RPC regressions in wallet backup/import/detail paths
- 钱包备份/导入/详情相关 RPC 回归

## Low-Risk Rework Order / 低风险重做顺序

### Step 1: `CBlockIndex` layout only / 仅做 `CBlockIndex` 结构瘦身

- field ordering
- 字段重排
- padding reduction
- 减少对齐填充
- move or drop pure cache-only fields if behavior does not change
- 在不改语义前提下外移或裁掉纯缓存字段

### Step 2: cold data split / 冷数据拆表

- move low-frequency metadata to side tables
- 把低频元数据移到 side table
- do not change block lookup semantics
- 不改变区块索引查找语义

### Step 3: container work only after tests exist / 容器层优化放到最后

- no custom open-addressing `BlockMap` until regression coverage is in place
- 在回归测试补齐前，不再引入自定义 open-addressing `BlockMap`
- container changes must be isolated in their own branch and commit
- 容器层改动必须单独分支、单独提交、单独验证

## Deliverables After Execution / 执行后的交付物

- `x/strict-pre-512-baseline`
- `x/current-line-revert-memory-wallet`
- exact commit pointers for both branches
- 两个分支的精确提交指针
- next verification command list
- 下一步验证命令/检查项列表

## Execution Result / 实际执行结果

- `main` -> `8a3a711bf`
- `x/strict-pre-512-baseline` -> `bbb842ce4`
- `x/current-line-revert-memory-wallet` -> `bdd0194b7`

Revert chain on `x/current-line-revert-memory-wallet`:
`x/current-line-revert-memory-wallet` 上的回退链如下：

1. `db3fed54c` `[0.1.3-1] Revert wallet send fallback and sender display patch / 回退钱包转出回退与发送方显示补丁`
2. `5a6ab254e` `[0.1.3-2] Revert Qt wallet receive crash fallback patch / 回退Qt钱包收币崩溃回退补丁`
3. `b8a3e33f2` `[0.1.2-3] Revert missing wallet tip tolerance patch / 回退缺失钱包tip容错补丁`
4. `bdd0194b7` `[0.1.0-4] Revert packed block index metadata and compact BlockMap / 回退压缩区块索引元数据与紧凑BlockMap`

## Verification Status / 验证状态

### `x/strict-pre-512-baseline`

- `python build_msvc\\msvc-autogen.py` succeeds.
- `msvc-autogen.py` 可以正常生成工程文件。
- Minimal MSVC build is blocked on this machine by missing dependency resolution for `event2/http.h`.
- 该分支在本机做最小 MSVC 构建时，卡在 `event2/http.h` 依赖解析，当前无法直接完成二进制验证。
- This means the strict historical baseline is still the cleanest code candidate, but not yet the easiest locally runnable candidate.
- 这说明严格历史基线依然是代码上最干净的候选线，但不是当前机器上最容易直接跑通的候选线。

### `x/current-line-revert-memory-wallet`

- `python build_msvc\\msvc-autogen.py` succeeds.
- `msvc-autogen.py` 可以正常生成工程文件。
- `sugarchaind.exe` and `sugarchain-cli.exe` build successfully after normalizing the shell environment to remove duplicate `Path/PATH`.
- 在清理 shell 里的重复 `Path/PATH` 环境变量后，`sugarchaind.exe` 和 `sugarchain-cli.exe` 已成功编出。
- Foreground `regtest` startup reaches `Done loading`.
- 前台 `regtest` 启动可以跑到 `Done loading`。
- Automated wallet smoke validation is not yet clean:
- 自动钱包烟雾验证目前还不算通过：
  1. block generation needs a larger RPC timeout than the initial smoke script used
  2. `debug.log` still shows `[smokewallet] ComputeTimeSmart: found ... in block ... not in index`
- This means the current-line revert branch is buildable here, but it is not yet proven to be the stable functional baseline.
- 这意味着当前线最小回退分支在本机可编译，但还不能证明它已经恢复到稳定功能基线。

### `x/current-line-revert-sendfix`

- Branch base: `x/current-line-revert-memory-wallet`
- 分支基线：`x/current-line-revert-memory-wallet`
- Branch head after focused functional fixes: `8f10a451d`
- 在补入最小功能修复后的分支头：`8f10a451d`
- Added back only the send fallback in `src/wallet/spend.cpp`.
- 仅恢复了 `src/wallet/spend.cpp` 里的发送回退逻辑。
- Added back only the RPC blocktime fallback in `src/wallet/rpc/transactions.cpp`.
- 仅恢复了 `src/wallet/rpc/transactions.cpp` 里的 RPC 区块时间回退逻辑。
- Verified outcomes:
- 已验证结果：
  1. minimal MSVC build succeeds
  2. `regtest` node starts and serves RPC
  3. wallet send succeeds after mining mature balance
  4. `gettransaction` succeeds on the sent transaction before restart
- Extended verification artifacts:
- 扩展验证产物：
  1. `verify-current-revert\\verify-artifacts\\sendfix-matrix-summary-20260316-145507.json`
  2. `verify-current-revert\\verify-artifacts\\sendfix-matrix-dump-20260316-145507.txt`
- Extended verification findings:
- 扩展验证结论：
  1. `dumpwallet` still crashes with `findAncestorByHeight(...)` in `src/wallet/wallet.cpp:2694 (GetKeyBirthTimes)`
  2. after restart with `-wallet=smokewallet`, wallet metadata loads but trusted balance and `listunspent` both come back as zero
  3. `sendtoaddress` after restart fails with `Insufficient funds` until `rescanblockchain 0` is executed
  4. `rescanblockchain 0` restores spendable balance, so the remaining bug is in wallet load state reconstruction, not raw chain data
  5. `importdescriptors` is still unverified in the Windows PowerShell harness because the JSON argument needs a cleaner invocation path; this is a test harness gap, not yet a confirmed product regression
- 这轮扩展验证说明：
  1. `dumpwallet` 仍会在 `src/wallet/wallet.cpp:2694 (GetKeyBirthTimes)` 触发 `findAncestorByHeight(...)` 崩溃
  2. 用 `-wallet=smokewallet` 重启后，wallet 元数据能加载，但 trusted balance 和 `listunspent` 都变成 0
  3. 重启后的 `sendtoaddress` 会先报 `Insufficient funds`，直到执行 `rescanblockchain 0`
  4. `rescanblockchain 0` 可以恢复可花余额，说明剩余问题在 wallet 加载态重建，不在底层链数据本身
  5. `importdescriptors` 在 Windows PowerShell 验证脚本里还没完全跑通，当前是测试脚本参数传递问题，还不能算产品回归

### `x/current-line-revert-blockmap`

- Branch base: `x/current-line-revert-sendfix`
- 分支基线：`x/current-line-revert-sendfix`
- Branch intent: restore pre-`6f57c49f7` `BlockMap` / `CBlockIndex` semantics, while keeping the already-proven wallet send and RPC fallbacks.
- 分支目标：恢复 `6f57c49f7` 之前的 `BlockMap` / `CBlockIndex` 语义，同时保留已验证有效的钱包发送与 RPC 回退。
- Applied changes:
- 已执行改动：
  1. restored `std::unordered_map<uint256, CBlockIndex, BlockHasher>` in `src/node/blockstorage.h`
  2. restored `phashBlock`-based `CBlockIndex` layout in `src/chain.h`
  3. kept the low-risk memory reduction from dropping the per-index PoW cache fields in `src/chain.h`
  4. kept `src/wallet/spend.cpp` and `src/wallet/rpc/transactions.cpp` tolerant fallbacks
- 执行后的核心状态：
  1. `src/node/blockstorage.h` 已恢复为 `std::unordered_map<uint256, CBlockIndex, BlockHasher>`
  2. `src/chain.h` 已恢复 `phashBlock` 方案
  3. `src/chain.h` 里“去掉每个 block index 的 PoW cache”这个低风险内存优化仍然保留
  4. `src/wallet/spend.cpp` 和 `src/wallet/rpc/transactions.cpp` 的容错回退仍然保留
- Verification artifacts:
- 验证产物：
  1. `verify-blockmap-rollback\\verify-artifacts\\blockmap-matrix-summary-20260316-162211.json`
  2. `verify-blockmap-rollback\\verify-artifacts\\blockmap-matrix-dump-20260316-162211.txt`
- Verified outcomes:
- 已验证结果：
  1. minimal MSVC build succeeds
  2. `dumpwallet` succeeds
  3. restart with `-wallet=smokewallet` preserves non-zero balance and non-empty `listunspent`
  4. `sendtoaddress` succeeds after restart without running `rescanblockchain`
- 已验证结果（中文）：
  1. 最小 MSVC 构建通过
  2. `dumpwallet` 正常
  3. 用 `-wallet=smokewallet` 重启后，余额和 `listunspent` 都正常保留
  4. 不执行 `rescanblockchain` 的情况下，重启后 `sendtoaddress` 也正常
  5. 这说明当前已复现的 block index / wallet load 回归，确实被 `6f57c49f7` 那层容器与索引语义变更触发

## Current Decision / 当前判断

- `6f57c49f7` is the first confirmed bad layer for the reproduced block index regressions.
- `6f57c49f7` 已经可以确认是当前这批 block index 回归的第一层坏点。
- The custom `BlockMap` / inline-hash `CBlockIndex` rewrite should stay out of the stable line until it has its own correctness proof and regression tests.
- 自定义 `BlockMap` / inline-hash `CBlockIndex` 重写，在没有独立正确性证明和回归测试之前，不应回到稳定线。
- The best current baseline is now `x/current-line-revert-blockmap`, not `x/current-line-revert-sendfix`.
- 当前最好的稳定基线已经变成 `x/current-line-revert-blockmap`，不再是 `x/current-line-revert-sendfix`。
- The safe memory-optimization direction is: keep the wallet fallbacks, keep low-risk `CBlockIndex` field reductions, but avoid changing block index container semantics.
- 安全的内存优化方向是：保留 wallet 容错、保留低风险 `CBlockIndex` 字段裁剪，但不要再改 block index 容器语义。
