# sugarcoin Umami Lite Memory Optimization Plan
# sugarcoin Umami Lite 内存优化双语实施方案

日期 / Date: 2026-03-12

## 1. 目标 / Goal

中文:

本方案将之前的内存分析收敛为一套可执行的最小修改路线。原则只有两个:

- 尽量少改代码
- 尽量不改变节点行为、协议、磁盘格式和已有运行参数语义

English:

This plan turns the previous memory analysis into an execution-oriented minimal-change roadmap. It follows two strict rules:

- Change as little code as possible
- Avoid changing node behavior, protocol rules, disk formats, and existing runtime semantics

## 2. 总体策略 / Overall Strategy

中文:

优先做“固定开销瘦身”和“按需启用”，暂不优先做大规模容器重构。原因很直接:

- 固定开销瘦身的风险更低
- 按需启用通常不影响共识和网络兼容性
- 大容器替换虽然收益高，但验证成本和回归风险最高

English:

Prioritize fixed-overhead reduction and lazy/on-demand allocation. Do not start with large container rewrites. The reasoning is simple:

- Fixed-overhead reduction has lower risk
- On-demand allocation usually does not affect consensus or network compatibility
- Large container rewrites may save more memory, but they carry the highest validation cost and regression risk

## 3. 模块一: UTXO 条目瘦身 / Module 1: Shrink UTXO Entry Footprint

### 3.1 目标 / Objective

中文:

在不替换 `CCoinsMap` 容器的前提下，先压缩 `Coin` 和 `CCoinsCacheEntry` 本体。

English:

Reduce the size of `Coin` and `CCoinsCacheEntry` first, without replacing the `CCoinsMap` container.

### 3.2 最小修改点 / Minimal Change Scope

中文:

- 检查 `Coin` 字段顺序和对齐
- 检查 `CCoinsCacheEntry.flags` 是否可与其他字段更紧凑排布
- 不改 `CCoinsMap` 类型
- 不改 flush 逻辑
- 不改序列化格式

English:

- Review field ordering and alignment in `Coin`
- Check whether `CCoinsCacheEntry.flags` can be packed more tightly with adjacent fields
- Do not change the `CCoinsMap` type
- Do not change flush logic
- Do not change serialization format

### 3.3 实施步骤 / Steps

1. 统计 `sizeof(Coin)`、`sizeof(CCoinsCacheEntry)`，确认当前基线。
2. 调整字段布局，优先消除对齐空洞。
3. 保持 `DIRTY/FRESH` 语义完全不变。
4. 编译并验证 `CCoinsViewCache::DynamicMemoryUsage()` 输出没有异常回退。

1. Measure `sizeof(Coin)` and `sizeof(CCoinsCacheEntry)` to establish a baseline.
2. Reorder fields to remove padding first.
3. Keep `DIRTY/FRESH` semantics unchanged.
4. Build and verify that `CCoinsViewCache::DynamicMemoryUsage()` shows no abnormal regression.

### 3.4 风险 / Risk

中文:

- `R1`
- 低到中
- 风险主要来自缓存状态机，不来自接口变化

English:

- `R1`
- Low to medium
- The risk is in cache state handling, not in interface changes

### 3.5 推荐优先级 / Priority

中文: 最高  
English: Highest

## 4. 模块二: Sugar 附加索引按需化 / Module 2: Make Sugar Extra Indexes Truly On-Demand

### 4.1 目标 / Objective

中文:

把 Sugar 分支额外引入的内存结构限制在“仅在启用对应功能时才维护”。

English:

Ensure Sugar-specific memory structures are only maintained when the corresponding feature is enabled.

### 4.2 最小修改点 / Minimal Change Scope

中文:

重点检查以下结构是否存在“无条件常驻、条件使用”的情况:

- `mapAddress`
- `mapAddressInserted`
- `mapSpent`
- `mapSpentInserted`

如果当前只是逻辑上判断开关，但对象始终存在，那么优先改成:

- 未启用时不插入
- 未启用时不更新
- 能延迟初始化就延迟初始化

English:

Focus on whether these structures exist unconditionally but are only used conditionally:

- `mapAddress`
- `mapAddressInserted`
- `mapSpent`
- `mapSpentInserted`

If the code currently guards usage but still keeps the objects alive all the time, change it to:

- No insertion when disabled
- No updates when disabled
- Lazy initialization where possible

### 4.3 实施步骤 / Steps

1. 梳理 `-addressindex`、`-spentindex`、`-timestampindex` 的使用路径。
2. 标出 mempool 里哪些结构在索引关闭时仍会分配或更新。
3. 将这些路径改为显式早返回。
4. 保持 RPC 行为和索引开启时的结果完全不变。

1. Trace the code paths for `-addressindex`, `-spentindex`, and `-timestampindex`.
2. Identify which mempool structures are still allocated or updated when indexes are disabled.
3. Turn those paths into explicit early returns.
4. Keep RPC behavior and enabled-index results unchanged.

### 4.4 风险 / Risk

中文:

- `R2`
- 低
- 只要不开启索引时的逻辑是“少做事”，风险就相对可控

English:

- `R2`
- Low
- As long as the disabled path only does less work, the risk stays controlled

### 4.5 推荐优先级 / Priority

中文: 高  
English: High

## 5. 模块三: Mempool 默认占用收紧 / Module 3: Tighten Default Mempool Footprint

### 5.1 目标 / Objective

中文:

通过默认参数下调，降低普通节点的实际内存占用。

English:

Reduce real-world memory usage for ordinary nodes by tightening default settings.

### 5.2 最小修改点 / Minimal Change Scope

中文:

仅调整默认值，不改算法:

- `DEFAULT_MAX_MEMPOOL_SIZE_MB`
- `DEFAULT_MAX_ORPHAN_TRANSACTIONS`
- `DEFAULT_BLOCK_RECONSTRUCTION_EXTRA_TXN`

English:

Adjust defaults only, without changing algorithms:

- `DEFAULT_MAX_MEMPOOL_SIZE_MB`
- `DEFAULT_MAX_ORPHAN_TRANSACTIONS`
- `DEFAULT_BLOCK_RECONSTRUCTION_EXTRA_TXN`

### 5.3 实施步骤 / Steps

1. 先记录当前默认值和启动参数说明。
2. 选择保守下调幅度，避免一次改太大。
3. 保持用户显式传参优先级不变。
4. 检查帮助文本和文档同步更新。

1. Record current defaults and startup help text.
2. Use conservative reductions instead of a large one-shot cut.
3. Preserve user-specified arguments exactly as they are today.
4. Update help text and docs consistently.

### 5.4 风险 / Risk

中文:

- `R3`
- 低
- 这是行为层取舍，不是正确性风险

English:

- `R3`
- Low
- This is a policy tradeoff, not a correctness risk

### 5.5 备注 / Note

中文:

这一项能最快降低实际占用，但它属于“配置优化”，不是结构性优化。

English:

This is the fastest way to reduce actual usage, but it is a configuration optimization rather than a structural optimization.

## 6. 模块四: Mempool 元数据瘦身 / Module 4: Reduce Mempool Metadata Overhead

### 6.1 目标 / Objective

中文:

减少每笔交易在 mempool 中的附加元数据成本，但不改变 mempool 行为模型。

English:

Reduce per-transaction metadata overhead inside the mempool without changing mempool behavior.

### 6.2 最小修改点 / Minimal Change Scope

中文:

优先看这些方向:

- 是否有可以延迟填充的辅助向量
- 是否有仅调试或边缘路径使用的缓存可下沉
- 是否有明显重复的映射维护

暂不做:

- `boost::multi_index_container` 替换
- 祖先/后代关系算法重写

English:

Focus on:

- Helper vectors that could be filled lazily
- Caches used only by debug or edge paths that can be moved out of the hot path
- Clearly duplicated mapping maintenance

Do not do yet:

- Replace `boost::multi_index_container`
- Rewrite ancestor/descendant logic

### 6.3 实施步骤 / Steps

1. 先测 `CTxMemPool::DynamicMemoryUsage()` 的组成。
2. 找出最重的非交易本体部分。
3. 只对最重的 1 到 2 个附加结构下手。
4. 保持 `add/remove/reorg/trim` 行为一致。

1. Break down `CTxMemPool::DynamicMemoryUsage()`.
2. Identify the heaviest non-transaction components.
3. Touch only the top one or two metadata structures first.
4. Keep `add/remove/reorg/trim` behavior identical.

### 6.4 风险 / Risk

中文:

- `R4`
- 中
- mempool 是热路径，不能为了省内存把维护成本和复杂度拉得过高

English:

- `R4`
- Medium
- The mempool is a hot path, so memory savings must not significantly raise maintenance cost or complexity

### 6.5 推荐优先级 / Priority

中文: 中  
English: Medium

## 7. 模块五: 大容器重构暂缓 / Module 5: Defer Large Container Rewrites

### 7.1 目标 / Objective

中文:

明确哪些高风险优化现在不做，避免路线跑偏。

English:

Explicitly mark which high-risk optimizations should not be done now.

### 7.2 暂缓项 / Deferred Items

中文:

- `CCoinsMap` 从 `std::unordered_map` 改成自定义紧凑容器
- 大规模重写网络缓冲管理
- 重写 mempool 多索引结构

English:

- Replace `CCoinsMap` with a custom compact container
- Large rewrite of network buffer management
- Rewrite mempool multi-index structures

### 7.3 暂缓原因 / Why Deferred

中文:

- `R5`
- 收益可能很高
- 但它们都不是“最小修改”
- 验证成本远高于当前阶段可接受水平

English:

- `R5`
- They may provide large savings
- But they are not minimal changes
- Their validation cost is far above what is acceptable at this stage

## 8. 建议执行顺序 / Recommended Execution Order

中文:

建议按以下顺序推进:

1. `Coin / CCoinsCacheEntry` 瘦身
2. Sugar 附加索引按需化
3. 收紧 mempool 默认参数
4. Mempool 元数据局部瘦身
5. 重新评估是否值得做 `CCoinsMap` 容器替换

English:

Proceed in this order:

1. Shrink `Coin / CCoinsCacheEntry`
2. Make Sugar extra indexes on-demand
3. Tighten mempool defaults
4. Reduce selected mempool metadata overhead
5. Re-evaluate whether `CCoinsMap` container replacement is worth doing

## 9. 每一步的回归要求 / Regression Checklist For Every Step

中文:

- 能正常编译 `sugarchain-cli`
- 能正常编译 `sugarchaind`
- 能正常编译 `sugarchain-qt`
- `-version` 输出正常
- 索引关闭时行为不变
- 索引开启时结果不变
- 不修改 P2P 消息格式
- 不修改区块和交易序列化格式

English:

- `sugarchain-cli` must still build
- `sugarchaind` must still build
- `sugarchain-qt` must still build
- `-version` output must remain valid
- Behavior must stay unchanged when indexes are disabled
- Results must stay unchanged when indexes are enabled
- No P2P message format changes
- No block or transaction serialization changes

## 10. 最终结论 / Final Conclusion

中文:

如果坚持“最小修改、影响最小”，下一步最应该做的不是再换一个大容器，而是先处理两个低风险高收益点:

- UTXO 条目本体瘦身
- Sugar 附加索引按需化

这两项做完之后，再决定是否需要进入更激进的容器级重构。

English:

If the priority is truly minimal change and minimal impact, the next step should not be another large container rewrite. The best next targets are:

- Shrinking the UTXO entry footprint
- Making Sugar extra indexes truly on-demand

Only after these two are done should we decide whether a more aggressive container-level rewrite is justified.
