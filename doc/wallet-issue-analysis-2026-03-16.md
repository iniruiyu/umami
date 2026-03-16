# 钱包问题分析与解决报告

日期: 2026-03-16

## 分析范围

本次重点核查以下几笔改动及其关联路径:

- `512b05271` `Pack block index metadata and shrink BlockMap buckets`
- `be6ee3f26` `[0.1.2] Tolerate missing wallet tip block / 容错缺失的钱包tip区块`
- `786714e17` `[0.1.3] Fix wallet receive crash in Qt / 修复Qt钱包收币崩溃`
- `8a3a711bf` `[0.1.3] Fix wallet send fallback and sender display / 修复钱包转出回退与发送方显示`

## 结论摘要

### 1. 无法转出

这是最近几笔改动里最明确、最直接的问题链。

- `786714e17` 只修了 Qt 收币状态查询路径里的 `findBlock(...)` 异常逃逸。
- 但建交易时，`src/wallet/spend.cpp` 的 `IsCurrentForAntiFeeSniping()` 仍然保留了 `CHECK_NONFATAL(chain.findBlock(...))`。
- 当钱包记录的 tip 哈希在当前内存区块索引里查不到时，发送流程会在这里直接中断。
- `8a3a711bf` 已把这里改成失败时 `return false`，并回退到 `nLockTime = 0` 的发送路径，因此这个提交是“无法转出”问题的直接修复。

代码定位:

- `src/wallet/spend.cpp:746`
- `src/wallet/spend.cpp:757`
- `src/wallet/spend.cpp:795`
- `src/wallet/spend.cpp:1018`

### 2. 现在会显示“转账人的地址”

这是 `8a3a711bf` 的新增 UI 行为，不是发送失败或交易状态异常的根因。

- `src/qt/transactiondesc.cpp` 新增了 `FormatInputSenders()`。
- 逻辑会遍历入账交易的每个输入，优先从钱包内前序交易取地址，取不到时再调用 `getrawtransaction` 推断前序输出地址。
- 然后把这些地址拼成 `From:` 显示在交易详情 HTML 里。

代码定位:

- `src/qt/transactiondesc.cpp:47`
- `src/qt/transactiondesc.cpp:64`
- `src/qt/transactiondesc.cpp:77`
- `src/qt/transactiondesc.cpp:225`

说明:

- 这是“展示层变化”，不修改钱包余额、不改交易状态、不改可花费性。
- 但它是启发式推断，不等于严格意义上的“真实付款人”。
- 多输入交易、自转账、找零混入时，这里的 `From:` 可能只是输入地址集合，不一定等价于单一发送人。

### 3. “流入交易被抛弃”

按当前代码看，最近这几笔补丁本身不会自动把“普通外部流入交易”标记成 `abandoned`。

状态来源如下:

- Qt 详情页在 `status.is_abandoned == true` 时显示 `abandoned`。
- `status.is_abandoned` 来自 `wtx.isAbandoned()`。
- `wtx.isAbandoned()` 只有在交易状态是 `TxStateInactive{abandoned=true}` 时才会返回真。

代码定位:

- `src/qt/transactiondesc.cpp:135`
- `src/qt/transactionrecord.cpp:208`
- `src/wallet/interfaces.cpp:85`
- `src/wallet/interfaces.cpp:100`
- `src/wallet/transaction.h:50`
- `src/wallet/transaction.h:79`
- `src/wallet/transaction.h:317`

根据代码推断，普通入账交易出现 `abandoned`，更可能来自以下路径:

- 手工调用过 `abandontransaction`
- Qt 右键菜单里对该交易执行过 `Abandon transaction`
- 钱包数据库里已经持久化了 `abandoned` 状态
- 少数 coinbase/后代交易会被代码主动标记为 abandoned，但这不符合“普通流入转账”场景

### 4. 一个额外的 UI/交互风险

当前 Qt 菜单允许对“任何深度为 0 且不在 mempool 的钱包交易”启用 `Abandon`，并没有限制它必须是“我发出的交易”。

- `TransactionCanBeAbandoned()` 只判断:
  - 不是已 abandoned
  - 深度为 0
  - 不在 mempool
- Qt 菜单直接据此启用 `Abandon`

代码定位:

- `src/wallet/wallet.cpp:1246`
- `src/wallet/wallet.cpp:1250`
- `src/qt/transactionview.cpp:398`
- `src/qt/transactionview.cpp:401`
- `src/qt/transactionview.cpp:422`

这意味着:

- 如果一笔“收到但当前不在本地 mempool 的交易”出现在列表里，它理论上也可能被用户手工标记为 abandoned。
- 一旦被标记，界面上就会显示 `Abandoned`，而这笔交易的输出在选币阶段也不会进入可花费集合。

这条路径和“流入交易被抛弃 + 无法转出”症状是吻合的。
这部分属于基于代码路径的推断，不是我从运行日志里直接观测到的事实。

## 为什么“被抛弃”会导致不能转出

可花费筛选在 `AvailableCoins()` 中有两层限制:

- 深度小于 0 的交易直接跳过
- 深度等于 0 且不在 mempool 的交易也直接跳过

代码定位:

- `src/wallet/spend.cpp:229`
- `src/wallet/spend.cpp:235`

所以只要一笔流入交易处于:

- `abandoned`
- 或者更宽泛地说，`depth == 0 && !InMempool()`

它的输出就不会参与选币，自然也就“看得到余额或记录，但花不出去”。

## 还存在的同类残留问题

这次修复只补了 Qt 收币路径和发送建交易路径，但同类 `findBlock/findAncestorByHeight + CHECK_NONFATAL` 还残留在其他钱包路径里。

典型位置:

- `src/wallet/wallet.cpp:1803`
- `src/wallet/wallet.cpp:2694`
- `src/wallet/wallet.cpp:2741`
- `src/wallet/rpc/transactions.cpp:31`
- `src/wallet/rpc/backup.cpp:535`
- `src/wallet/rpc/backup.cpp:744`
- `src/wallet/rpc/backup.cpp:1373`
- `src/wallet/rpc/backup.cpp:1672`

这些点的风险是:

- `gettransaction` / 交易详情 RPC 仍可能因区块索引缺项而中断
- dump/import/descriptor import/backup 相关 RPC 仍可能复现同类异常
- rescan 或 key birth time 推断路径仍有潜在炸点

## 其它次级问题

### 1. sender 地址展示有性能和准确性风险

- 每个输入都可能触发一次 `getrawtransaction`
- 没有缓存，同一前序交易多个输入会重复查
- `txindex` 关闭时会静默失败，显示结果不稳定
- 即使查到地址，也只是“输入地址集合”，不一定能严格代表付款人

### 2. 最近修复缺少自动化测试

当前仓库里没有看到覆盖以下场景的针对性测试:

- wallet tip 哈希缺失时 Qt 状态查询不崩
- wallet tip 哈希缺失时建交易仍可继续
- sender 推断展示在 `txindex` 开/关时的行为
- 收款交易不能被错误暴露为可 abandon 的 UI 操作

## 建议修复顺序

### P0

1. 限制 `Abandon transaction` 只允许用于“本钱包发出的交易”，不要对普通收款交易开放。
2. 把剩余钱包路径里的 `CHECK_NONFATAL(findBlock/findAncestorByHeight)` 统一改成“可回退、可降级”的容错逻辑。

### P1

1. 为 `sender` 展示增加缓存，避免一个详情页触发多次重复 RPC。
2. 把 `From` 文案改成更保守的描述，例如“Inputs”或“Possible sender inputs”，避免误导。

### P2

1. 增加针对 `missing wallet tip hash` 的单元测试或功能测试。
2. 增加“收款交易不应被错误 abandon”的 UI/钱包测试。

## 最终判断

- “无法转出”是最近补丁链里已经定位清楚的问题，直接根因在发送建交易路径未同步补上 `findBlock` 容错，`8a3a711bf` 已修。
- “显示转账人的地址”是 `8a3a711bf` 的新增展示逻辑，本身不是故障，但存在准确性和性能副作用。
- “流入交易被抛弃”不是这几笔新补丁直接造成的自动行为；更像是钱包已有状态、手工 abandon、或 UI 对收款交易错误开放了 abandon 能力所导致。
- 代码里仍有多处同类 `findBlock/findAncestorByHeight` 强断言残留，后续仍可能在 RPC、备份、导入、重扫等路径继续冒出同根问题。
