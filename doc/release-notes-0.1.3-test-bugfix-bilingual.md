# sugarcoin Umami Lite 0.1.3 Test Bugfix Release Notes
# sugarcoin Umami Lite 0.1.3 测试修复版发布说明

## English

This is the `0.1.3` test bugfix release for `sugarcoin Umami Lite`.

### Fixed

- Fixed a wallet crash in Qt after receiving a transaction.
- Fixed the case where wallet status lookup could throw `NonFatalCheckError` when the wallet tip hash was not found in the current in-memory block index.
- Changed the wallet UI path to tolerate a missing tip block and return an unknown block time instead of letting an exception escape through the Qt event loop.

### Scope

- No protocol changes.
- No consensus changes.
- No block or transaction serialization changes.
- Existing nodes remain wire-compatible.

## 中文

这是 `sugarcoin Umami Lite 0.1.3` 的测试修复版本。

### 已修复问题

- 修复了 Qt 钱包在收到交易后可能发生的崩溃问题。
- 修复了钱包状态查询在“钱包记录的 tip 区块哈希不在当前内存区块索引中”时触发 `NonFatalCheckError` 的问题。
- 现在钱包界面路径会容错处理缺失的 tip 区块，并返回未知区块时间，而不是让异常穿过 Qt 事件循环导致程序报错。

### 影响范围

- 不修改网络协议。
- 不修改共识规则。
- 不修改区块或交易序列化格式。
- 与现有节点保持网络兼容。
