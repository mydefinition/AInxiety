# Shared Contracts

这里用于维护版本化的 HTTP 协议、JSON Schema 和消息示例。当前协议草案位于 [docs/contracts.md](../docs/contracts.md)，尚未生成可执行 Schema 或客户端。

服务只共享公开协议，不共享内部业务实现。推送和拉取使用相同消息结构、投递 ID、幂等键和状态语义。

后续实现约束：协议变更应同时更新提供方、消费者及契约验证；不兼容变更升级版本，支持独立部署期间的兼容过渡。
