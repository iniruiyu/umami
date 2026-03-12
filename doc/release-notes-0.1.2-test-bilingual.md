# sugarcoin Umami Lite 0.1.2 Test Release Notes
# sugarcoin Umami Lite 0.1.2 测试版发布说明

日期 / Date: 2026-03-12

## Release Type / 发布类型

English:

This is a test release for `sugarcoin Umami Lite 0.1.2`.

中文:

这是 `sugarcoin Umami Lite 0.1.2` 的测试版本发布。

## Binaries / 二进制程序

English:

- `CLI`
- `sugarcoind`
- `sugarcoin-qt`

中文:

- `CLI`
- `sugarcoind`
- `sugarcoin-qt`

## Changes / 改动说明

### 1. Version update / 版本更新

English:

- Bumped the test version from `0.1.1` to `0.1.2`

中文:

- 测试版本从 `0.1.1` 提升到 `0.1.2`

### 2. Optional mempool side-index behavior retained / 保留可选 mempool 附加索引优化

English:

- Kept the explicit mempool switches for optional address and spent side indexes
- Nodes that do not enable those indexes continue to avoid maintaining the extra mempool side-index paths

中文:

- 保留可选地址索引和花费索引的 mempool 显式开关
- 未启用这些索引的节点，继续避免维护对应附加索引路径

### 3. Default memory policy tightening / 默认内存策略收紧

English:

- Reduced the default mempool size from `300 MB` to `256 MB`
- Reduced the default orphan transaction limit from `100` to `64`
- Reduced the default compact-block reconstruction extra transaction cache from `100` to `64`

中文:

- 默认 mempool 上限从 `300 MB` 下调到 `256 MB`
- 默认孤儿交易上限从 `100` 下调到 `64`
- 默认紧凑区块重建附加交易缓存从 `100` 下调到 `64`

## Compatibility / 兼容性

English:

- No P2P message format changes
- No block or transaction serialization changes
- No consensus rule changes
- User-specified runtime arguments still override defaults

中文:

- 没有修改 P2P 消息格式
- 没有修改区块或交易序列化格式
- 没有修改共识规则
- 用户显式传入的运行参数仍然优先于默认值

## Notes / 备注

English:

- This release focuses on low-risk memory reduction by tightening defaults instead of changing core data structures

中文:

- 这次发布通过收紧默认值来降低内存占用，属于低风险优化，没有改动核心数据结构
