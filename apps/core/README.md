# AInxiety Core

独立服务，负责多源采集、事件记忆、去重、核验、摘要生成、订阅管理、发布策略和投递记录。

技术基线：Python、FastAPI、LangGraph、PostgreSQL/pgvector。先实现订阅推送，再增加领取式拉取；暂未创建应用代码或运行入口。

核心不连接 QQ 协议，不直接操作博客数据库。渠道凭据由投递组件持有，不传入模型节点。

参见 [系统设计](../../docs/design.md) 和 [渠道契约](../../docs/contracts.md)。
