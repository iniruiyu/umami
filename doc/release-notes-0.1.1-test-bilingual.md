# sugarcoin Umami Lite 0.1.1 Test Release Notes
# sugarcoin Umami Lite 0.1.1 测试版发布说明

日期 / Date: 2026-03-12

## Release Type / 发布类型

English:

This is a test release for `sugarcoin Umami Lite 0.1.1`.

中文:

这是 `sugarcoin Umami Lite 0.1.1` 的测试版本发布。

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

### 1. Branding and version update / 品牌和版本更新

English:

- Updated the product branding to `sugarcoin Umami Lite`
- Bumped the version to `0.1.1`
- Aligned Windows-generated package metadata with the new branding

中文:

- 产品标题统一更新为 `sugarcoin Umami Lite`
- 版本号提升到 `0.1.1`
- Windows 生成链中的包信息已同步到新的品牌名称

### 2. Windows build support / Windows 构建支持

English:

- Kept the MSVC build flow aligned with the current project name and version
- Preserved Qt static build support in the existing Windows environment

中文:

- 保持 MSVC 构建链与当前项目名称和版本一致
- 保留现有 Windows 环境下的 Qt 静态编译支持

### 3. Optional mempool side-index optimization / 可选 mempool 附加索引优化

English:

- Added explicit mempool options for optional address and spent side indexes
- When `-addressindex` or `-spentindex` is disabled, the mempool now skips maintaining those side-index paths
- This reduces unnecessary work and memory overhead on nodes that do not use those indexes

中文:

- 为 mempool 增加了可选地址索引和花费索引的显式开关
- 当 `-addressindex` 或 `-spentindex` 关闭时，mempool 不再维护对应附加索引路径
- 对不使用这些索引的节点，可以减少不必要的维护开销和内存占用

## Compatibility / 兼容性

English:

- No P2P message format changes
- No block or transaction serialization changes
- No consensus rule changes

中文:

- 没有修改 P2P 消息格式
- 没有修改区块或交易序列化格式
- 没有修改共识规则

## Notes / 备注

English:

- This is a test release intended for verification and further optimization work
- The release focuses on minimal-change updates and low-risk memory improvements

中文:

- 这是一个用于验证和继续优化的测试版本
- 这次发布重点是最小修改和低风险内存优化
