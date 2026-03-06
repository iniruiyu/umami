# Umami 节点内存占用异常问题报告（4300 万区块场景）

## 1. 背景
- 项目：`sugarchain-project/umami`
- 场景：节点高度约 `43,000,000` 区块，运行过程中内存占用偏高。
- 时间：`2026-03-06`

## 2. 现象描述
- 在启用扩展索引（尤其是 `-addressindex` / `-spentindex`）的节点中，内存会随着 mempool 交易进出持续增长。
- 即使设置了 `-maxmempool`，进程总内存仍可能明显超过预期。

## 3. 根因分析

### 根因 A：mempool 扩展索引未随交易驱逐清理（逻辑泄漏）
- 交易进入 mempool 时会写入：
  - `mapAddress` / `mapAddressInserted`
  - `mapSpent` / `mapSpentInserted`
- 但在交易离开 mempool 的主路径 `CTxMemPool::removeUnchecked()` 中，原实现未调用：
  - `removeAddressIndex(txid)`
  - `removeSpentIndex(txid)`
- 结果：这些 map 会残留历史条目，形成持续增长的内存占用。

### 根因 B：mempool 内存统计低估（限额失真）
- `CTxMemPool::DynamicMemoryUsage()` 原实现仅统计了：
  - `mapTx`、`mapNextTx`、`mapDeltas`、`vTxHashes`、`cachedInnerUsage`
- 未统计扩展索引 map：
  - `mapAddress`、`mapAddressInserted`
  - `mapSpent`、`mapSpentInserted`
- 结果：`LimitMempoolSize()` 基于低估值执行，`-maxmempool` 无法准确约束实际总占用。

## 4. 修复方案与代码变更

### 变更 1：交易移除时同步清理扩展索引
- 文件：`src/txmempool.cpp`
- 函数：`CTxMemPool::removeUnchecked`
- 新增逻辑：在拿到 `hash` 后调用：
  - `removeAddressIndex(hash)`
  - `removeSpentIndex(hash)`

### 变更 2：将扩展索引纳入内存统计
- 文件：`src/txmempool.cpp`
- 函数：`CTxMemPool::DynamicMemoryUsage`
- 新增统计项：
  - `memusage::DynamicUsage(mapAddress)`
  - `memusage::DynamicUsage(mapAddressInserted)`
  - `memusage::DynamicUsage(mapSpent)`
  - `memusage::DynamicUsage(mapSpentInserted)`

## 5. 影响评估
- 正向影响：
  - 避免 mempool 扩展索引残留造成的内存持续增长。
  - 让 `-maxmempool` 更接近“真实总使用量”的控制预期。
- 兼容性：
  - 仅影响 mempool 内存索引生命周期与计量，不改共识逻辑。
  - 对区块数据、链状态、磁盘索引格式无破坏性变更。

## 6. 验证建议（线上/测试）
1. 使用较小阈值压测（例如 `-maxmempool=100`）并持续发送交易，观察 RSS 是否稳定在合理范围。
2. 观察驱逐日志（mempool trim）后，进程内存是否出现回落或稳定。
3. 对比修复前后：
   - 同等流量下峰值内存
   - 稳态内存
   - 长时间运行后的增长斜率

## 7. 生产参数建议（4300 万区块节点）
如果目标是“稳定运行 + 低内存”，建议：

```ini
# sugarchain.conf 建议（按机器内存再微调）
dbcache=300
maxmempool=120
maxorphantx=40
mempoolexpiry=72

# 如无明确查询需求，关闭高开销索引
txindex=0
addressindex=0
spentindex=0
timestampindex=0
coinstatsindex=0
blockfilterindex=0
```

说明：
- `dbcache` 越大，链验证更快，但常驻内存更高。
- `maxmempool` 是直接影响内存的第一开关。
- 索引类选项应按业务需求开启，避免“开着不用”。

## 8. 风险与回滚
- 风险：低。修改点集中在 mempool 辅助索引与统计逻辑。
- 回滚：回退本次提交即可恢复原行为。

## 9. 结论
本次问题的核心不是区块高度本身，而是 mempool 扩展索引的生命周期管理与内存计量缺失。修复后可显著改善长时间运行的内存可控性，并使 `-maxmempool` 的限制更可信。
