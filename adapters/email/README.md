# Email Adapter

将统一发布内容转换为邮件主题、HTML 和纯文本正文，通过邮件服务 API 或 SMTP 发送。

初期作为投递 worker 的模块；未来可以独立服务化。负责凭据引用、服务商消息 ID、发送结果与退信状态映射。服务商接收不等于进入收件箱，SMTP Message-ID 不保证去重。

服务商和实现尚未选择。参见 [系统设计](../../docs/design.md)。
