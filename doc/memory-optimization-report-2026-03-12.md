# sugarcoin Umami Lite 内存优化分析报告

日期: 2026-03-12

## 1. 背景

当前版本已经对区块索引做过一轮高收益优化:

- `m_block_index` 从 `std::unordered_map<uint256, CBlockIndex>` 改成了紧凑型 `BlockMap`
- `CBlockIndex` 改为内联保存 `hash`
- 去掉了 `CBlockIndex` 上的 PoW cache
- `BlockMap` 桶从指针改成了 1-based `uint32_t` 索引

这一轮优化已经明显降低了常驻内存，但从代码结构看，剩余内存占用仍然主要集中在以下几类对象:

- UTXO 缓存 `CCoinsViewCache`
- mempool 及其多索引、祖先/后代关系和附加索引
- 网络层每连接缓冲区
- 可选索引功能和重组缓冲池

本报告的目标不是泛泛列点，而是按“收益 / 风险 / 实施复杂度”给出下一步值得做的优化方向。

## 2. 主要内存热点判断

### 2.1 UTXO 缓存

相关位置:

- [src/coins.h](/C:/Users/ini/Desktop/git/sugarchain/umami/src/coins.h)
- [src/coins.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/coins.cpp)
- [src/validation.h](/C:/Users/ini/Desktop/git/sugarchain/umami/src/validation.h)
- [src/validation.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/validation.cpp)

当前结构:

- `CCoinsMap = std::unordered_map<COutPoint, CCoinsCacheEntry, SaltedOutpointHasher>`
- `CCoinsCacheEntry` 包含 `Coin coin` 和 `flags`
- `CCoinsViewCache::DynamicMemoryUsage()` 直接把 `cacheCoins` 和 coin 内部脚本内存都计入

判断:

- 这是全节点常驻内存中的绝对大头之一
- 当 `-dbcache` 较大时，UTXO 缓存会主动吃满预算
- 单条 coin 体积不一定夸张，但数量巨大，哈希表节点开销和装载因子开销会非常可观

### 2.2 Mempool

相关位置:

- [src/txmempool.h](/C:/Users/ini/Desktop/git/sugarchain/umami/src/txmempool.h)
- [src/txmempool.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/txmempool.cpp)
- [src/node/mempool_args.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/node/mempool_args.cpp)

当前结构:

- `mapTx` 是 `boost::multi_index_container`
- 同时按 `txid`、`wtxid`、descendant score、entry time、ancestor score 建了多索引
- 还维护了 `vTxHashes`
- 还维护了 `mapNextTx`
- 祖先/后代关系集合常驻
- Sugar 分支另外加了 `mapAddress`、`mapAddressInserted`、`mapSpent`、`mapSpentInserted`

判断:

- 默认 `-maxmempool=300MB`，这部分上限本身就不低
- 对于开启地址/花费索引的节点，mempool 元数据比上游更重
- 这里不是单个结构太大，而是“同一笔交易被多个索引和关系结构重复引用”

### 2.3 网络层缓冲和孤儿池

相关位置:

- [src/net.h](/C:/Users/ini/Desktop/git/sugarchain/umami/src/net.h)
- [src/net_processing.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/net_processing.cpp)

当前结构:

- `CNode` 上有 `vSendMsg`、`vRecvMsg`、`m_msg_process_queue`
- 默认 `-maxorphantx=100`
- 默认 `-blockreconstructionextratxn=100`

判断:

- 正常情况下不是最大头
- 但在高连接数、慢 peer、流量尖峰或异常 gossip 条件下，会形成明显波峰
- 这类占用波动大，适合做“上限控制”和“懒分配”，不适合做复杂结构重写

### 2.4 可选索引

相关位置:

- [src/init.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/init.cpp)
- [src/index/txindex.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/index/txindex.cpp)
- [src/index/coinstatsindex.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/index/coinstatsindex.cpp)
- [src/index/blockfilterindex.cpp](/C:/Users/ini/Desktop/git/sugarchain/umami/src/index/blockfilterindex.cpp)

当前结构:

- `txindex`
- `coinstatsindex`
- `blockfilterindex`
- Sugar 自带的 `addressindex` / `spentindex` / `timestampindex`

判断:

- 这些功能更多占用磁盘和缓存，但也会抬高同步期和运行期内存曲线
- 其中 Sugar 自带索引会直接增加 mempool 与区块接入路径的附加对象数量

## 3. 候选优化方案

## 3.1 方案 A: 把 `CCoinsMap` 从 `std::unordered_map` 换成紧凑型开放寻址结构

目标:

- 复制区块索引优化思路，降低 UTXO 缓存容器开销

原因:

- `unordered_map` 对大量小对象非常浪费
- 每个节点都有桶、链节点、分配器和对齐损耗
- UTXO 缓存恰好是“海量条目 + key/value 较小 + 查找频繁”的典型场景

预期收益:

- 很可能是后续最大的单项收益来源之一
- 当 `-dbcache` 较大时，收益会非常明显
- 理论上有机会再省数百 MB 到 1GB+，取决于缓存规模和装载率

实现难度:

- 高

风险:

- `CCoinsViewCache` 的一致性要求非常高，`DIRTY/FRESH` 出错就是共识级事故
- 需要完整处理插入、擦除、rehash、迭代、批量 flush、snapshot load
- 现有逻辑高度依赖 `unordered_map` 语义，替换要非常谨慎

可行性判断:

- 技术上完全可行
- 但这不是“先做”的最佳项，除非准备投入完整验证周期

结论:

- 高收益
- 高风险
- 适合作为中期优化，不适合作为下一刀

## 3.2 方案 B: 压缩 `Coin / CCoinsCacheEntry` 本体

目标:

- 减少 UTXO 缓存中每条 entry 的固定体积

可考虑的方向:

- 检查 `Coin` 字段布局，压缩对齐空洞
- 把 `flags` 与更适合打包的字段合并
- 如果有只在 flush 或 debug 路径使用的状态，考虑外移

预期收益:

- 中等到高
- 因为条目数量巨大，即使每条只减 8 到 16 字节，也可能形成数百 MB 级收益

实现难度:

- 中

风险:

- 低于替换整个 UTXO 容器
- 但仍需谨慎处理序列化、缓存状态机和 spent/unspent 语义

可行性判断:

- 这是 UTXO 方向里风险最可控的一项

结论:

- 推荐优先级高
- 是继 `CBlockIndex` 瘦身之后最自然的延续方向

## 3.3 方案 C: Mempool 分层，拆出“轻节点模式”

目标:

- 明确把默认运行模式调成“更省内存”的全节点配置

可考虑的方向:

- 降低默认 `-maxmempool`，例如从 300MB 下调到更保守值
- 下调 `-maxorphantx`
- 下调 `-blockreconstructionextratxn`
- 把地址/花费相关 mempool 附加索引做成更严格的按需启用

预期收益:

- 立竿见影
- 代码改动小
- 对极端大内存场景帮助有限，但对多数普通节点有效

实现难度:

- 低

风险:

- 功能和性能取舍明显
- 降低 mempool 上限会影响交易保留时间、fee 估计稳定性和传播体验
- 降低 orphan / reconstruction 缓冲会影响异常网络场景表现

可行性判断:

- 很高

结论:

- 如果目标是“尽快再降一截实际占用”，这是最快见效方案
- 但它更像运营配置优化，不是核心数据结构优化

## 3.4 方案 D: 给 Sugar 自带索引增加严格的延迟初始化和可选禁用

目标:

- 避免无需求节点承受额外索引常驻成本

现状:

- `-addressindex`
- `-spentindex`
- `-timestampindex`

这些索引已经是可选开关，但 Sugar 分支在 mempool 上还维护了附加映射:

- `mapAddress`
- `mapAddressInserted`
- `mapSpent`
- `mapSpentInserted`

优化方向:

- 确认这些 mempool 附加结构只在对应索引启用时才真正分配和更新
- 如果当前是无条件存在、条件使用，就改成按需构造
- 评估是否能把 `std::map` 改成更紧凑的结构

预期收益:

- 中等
- 对不开索引的普通节点收益明显
- 对必须开启这些索引的服务节点收益有限

实现难度:

- 中

风险:

- 低到中
- 主要风险在索引一致性和重组回滚路径

可行性判断:

- 很高

结论:

- 值得尽快核实
- 这是“代码改动不大，但可能白拿一部分内存”的方向

## 3.5 方案 E: Mempool 内部附加结构瘦身

目标:

- 降低每笔交易在 mempool 中的元数据体积

可考虑的方向:

- 重新评估 `vTxHashes` 是否必须始终完整常驻
- 检查祖先/后代集合的容器类型是否过重
- 对 `mapNextTx` 做更紧凑表示
- 评估 Sugar 的 `mapAddress*` / `mapSpent*` 是否可由事务内局部向量替代一部分常驻映射

预期收益:

- 中等

实现难度:

- 中到高

风险:

- 中
- mempool 对一致性和性能都敏感，改坏后会直接影响接收交易、驱逐、重组、打包

可行性判断:

- 可做
- 但需要基于实际 mempool 样本测量后再动手，不能盲改

结论:

- 适合作为第二阶段优化

## 3.6 方案 F: 网络缓冲上限和懒分配

目标:

- 降低高连接数场景的瞬时内存波峰

优化方向:

- 更严格限制 `vSendMsg` 和 `m_msg_process_queue` 的积压
- 对大消息缓冲使用对象池或 slab
- 降低默认 orphan / compact block reconstruction 缓冲

预期收益:

- 低到中
- 更偏峰值控制，不是常驻内存大头

实现难度:

- 中

风险:

- 中
- 容易影响吞吐、延迟和慢节点兼容性

可行性判断:

- 可行，但优先级不高

结论:

- 适合最后处理

## 4. 推荐实施顺序

### 第一优先级

1. 压缩 `Coin / CCoinsCacheEntry` 本体
2. 核实并收紧 Sugar 附加索引的按需分配
3. 适度下调默认运行参数:
   - `-maxmempool`
   - `-maxorphantx`
   - `-blockreconstructionextratxn`

原因:

- 风险可控
- 收益明确
- 不需要重写共识敏感容器

### 第二优先级

1. Mempool 内部元数据瘦身
2. 重新设计 Sugar 的 mempool 地址/花费附加结构

原因:

- 不一定有 UTXO 那么大的收益
- 但比替换 `CCoinsMap` 更安全

### 第三优先级

1. 把 `CCoinsMap` 改成紧凑容器

原因:

- 这可能是未来最大的单项收益
- 但也是最容易引入隐蔽一致性问题的重构

## 5. 我对下一步的明确建议

如果目标是“继续明显下降实际常驻内存，同时把风险压住”，下一步最值得做的是:

1. 先量化 `sizeof(Coin)` 和 `sizeof(CCoinsCacheEntry)`，确认结构体本体是否还有 8 到 16 字节的可压缩空间。
2. 检查 Sugar 的 `mapAddress` / `mapSpent` 系列结构是否在未开启对应索引时仍然常驻维护。
3. 再决定是否下调默认 `-maxmempool` / `-maxorphantx` / `-blockreconstructionextratxn`。

不建议立刻进入 `CCoinsMap` 容器替换。原因很简单:

- 收益可能很高
- 但验证成本和共识风险也最高
- 当前阶段更适合先把“低风险但仍有百 MB 级收益”的项目吃干净

## 6. 简短结论

在区块索引已经瘦身之后，剩余最值得投入的方向是:

- UTXO 条目本体压缩
- Sugar 附加索引按需化
- mempool 元数据瘦身

其中真正应该先做的不是再换一个大容器，而是先把 UTXO entry 和 Sugar 额外索引里的固定浪费找出来。这条路线更稳，验证成本也明显更低。
