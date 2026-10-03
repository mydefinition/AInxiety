# 事件与渠道契约草案

接口版本：v1 草案，尚未实现。服务间使用 UTC 时间戳，报告周期另带 IANA 时区。

## 核心实体

| 实体 | 核心字段 |
| --- | --- |
| Source | id、type、entrypoint、topics、poll_policy、health |
| SourceItem | id、source_id、external_id、canonical_url、published_at、discovered_at |
| SourceRevision | id、item_id、content_hash、title、body、fetched_at、extraction_status |
| Event | id、canonical_event_id、entities、action、product_version、occurred_at、status |
| EventRevision | event_id、revision、title、facts、evidence、importance、change_kind |
| ItemEvent | source_revision_id、event_id、relation、rationale |
| Publication | id、kind、period、timezone、event_revisions、title、summary、links、content_version |
| Subscription | id、consumer_id、destination_id、kinds、topics、mode、webhook_url、secret_ref、recovery_policy、enabled |
| Delivery | id、publication_id、subscription_id、destination_id、idempotency_key、state、expires_at、lease_until、lease_generation、remote_ids、attempts |
| Feedback | id、actor_id、event_id/publication_id、item_ref、type、value、created_at |
| Job | id、kind、dedupe_key、state、lease_until、attempts、next_attempt_at |

`facts` 保存可独立比较的陈述及事实 ID；`evidence` 保存每项事实的来源修订、链接、支持片段、归属/引用关系。官方声明也是声明，核验状态不能由来源类型直接推导。

一个来源文章可以对应多个事件，一个事件可以对应多篇文章。正文变化形成来源修订，事实变化形成事件修订，发布内容保留当时的修订引用。

事件修订关系：`duplicate`、`material_update`、`new_event`、`uncertain`；核验状态与重要性分开记录。

## 投递方式

采用两种传输方式，共用同一组投递记录。第一阶段实现订阅式 HTTP 推送，第二阶段实现领取式拉取。QQ Bot、博客及邮件适配器各自只有所需目标渠道的权限；不向它们暴露数据库。邮件适配器领取或接收任务后调用邮件服务，不向邮箱地址发送 Webhook。

### 订阅配置

第一阶段通过受控配置管理订阅，暂不要求实现管理界面。每项订阅包含：

- 消费者身份与目标 ID；目标可为 QQ 群、博客站点或邮件收件目标。
- 内容类型 `kinds`、主题过滤 `topics`；频率限制归属于目标，不能通过多订阅绕过。
- `mode`：`push`、`pull`、`push_with_pull_recovery`。默认目标模式为后者，第一阶段只启用其推送路径；拉取能力上线前不宣称已经支持恢复领取。
- Webhook 接收地址、签名密钥引用、超时与重试策略（推送模式必需）。内置邮件适配器可直接接收持久化任务，无需额外 HTTP 服务。
- `recovery_policy`：快讯有效期、报告补投范围、最大补投速率；部署时明确配置。
- 启停状态。暂停订阅停止新任务和新领取；已接收的任务需要另行确认取消，不能假定撤销成功。

多个订阅命中同一内容版本和目标时，合并为一个 Delivery，并记录匹配来源；数据库唯一约束与幂等键基于 `publication_id + content_version + destination_id`，不包含传输方式或重试次数。修改订阅默认只影响未来发布，不自动回填历史。

### 推送任务（第一阶段）

AInxiety worker 原子领取待投递记录后，向配置的下游地址发送 `POST`（推荐路径 `/integrations/ainxiety/publications`）。请求体使用下方统一消息结构，推送与拉取共享 `delivery_id` 和 `idempotency_key`。

下游先在持久化 inbox 中按投递 ID 去重并存储任务，再返回 `202 Accepted` 及 `delivery_id`、`status=accepted`；重复请求返回已存状态。仅当任务安全落库后才能确认接收。已同步完成的处理可以返回 `200` 及 `status=sent`、远端 ID。

接收成功后 AInxiety 将任务记为 `accepted`，停止常规推送重试；下游独立执行并提交最终回执。网络错误或临时失败按退避和抖动重试，明确不可重试的错误进入 `failed`。超时可能意味着下游已经接收，先以相同 ID 查询/核对；只有在下游持久化去重机制有效的前提下才能安全重复提交同一任务。

推送采用 HTTPS（私有网络部署也须有认证），请求头携带投递 ID、时间戳及原始请求体的 HMAC 签名。各订阅独立密钥，校验时间窗口；重试保持投递 ID，更新签名时间。Webhook 地址只能由有权限的配置管理者指定。

### 领取任务（第二阶段）

`POST /v1/deliveries/claim`

请求：`consumer_id`、`destination_id`、`limit`。服务端根据认证身份校验可领取的目标，不能只信任请求中的 consumer_id。

接口返回任务数组，原子领取 `pending` 或明确可重试且到期的任务；按优先级、时效和配额过滤。纯拉取订阅不由推送 worker 领取。推送附带拉取恢复的订阅共享领取锁，不能并行发出两个执行授权。轮询频率及最大批量可配置。

数组中的单条任务示例（也是推送请求体）：

```json
{
  "delivery_id": "del_123",
  "claim_token": "opaque-lease-token",
  "lease_expires_at": "2026-10-03T10:02:00Z",
  "idempotency_key": "pub_123:qq_group_demo:v1",
  "destination_id": "qq_group_demo",
  "publication": {
    "id": "pub_123",
    "kind": "breaking",
    "content_version": 1,
    "event_refs": [{"event_id": "evt_123", "revision": 1}],
    "title": "示例：某项目发布新版本",
    "summary": "示例内容，仅展示接口结构。",
    "links": [{"label": "原始公告", "url": "https://example.org/announcement"}]
  }
}
```

支持 `breaking`、`daily`、`weekly`、`correction`。日报、周报可增加结构化 `items`，每项包含事件引用、摘要和链接；共享字段不依赖 QQ 消息格式。

### 回执与续租

- `POST /v1/deliveries/{id}/heartbeat`：携带领取令牌续租。
- `POST /v1/deliveries/{id}/result`：提交领取令牌、状态、远端消息 ID 列表、完成时间、错误码及是否可重试。
- 状态：`sent`、`failed`、`unknown`。`unknown` 表示可能发送成功但未收到确认。

`claim_token` 对应投递租约和递增的 `lease_generation`，推送也携带该令牌。接收或执行阶段允许续租，服务端拒绝失效令牌推进状态。重复回执按投递及尝试去重；迟到回执保留为核对证据，不能覆盖新执行状态。

核心状态流为 `pending → leased → accepted → sent`；明确可重试失败进入 `retry_wait`，不可重试进入 `failed`，不确定结果进入 `unknown`，未开始且超过有效期进入 `expired`。拉取消费者持久化接收后，通过 `POST /v1/deliveries/{id}/accepted` 携带令牌确认；推送响应承担同样作用。租约到期不能直接从 `accepted/unknown` 回到待领取。

### 离线恢复与状态核对

1. 下游启动先读取自身 inbox，向 `GET /v1/deliveries/{id}` 查询授权范围内的状态，并补交已完成任务的回执。
2. 已接收但未完成的任务先核对下游/平台执行记录；确认未执行且需续办时，通过 `POST /v1/deliveries/{id}/reconcile` 提交证据，请求原子授予新租约。该接口不能仅凭超时自动认定未发送。
3. 对结果未知、旧 worker 可能仍执行的任务，先隔离或停止旧执行者并核对；租约本身不能阻止已发往平台的请求产生副作用。不盲目接管或重发。
4. 再调用领取接口补齐仍有效的未接收任务；不按“上次拉取时间”简单扫描所有新闻，避免把已经推送成功的内容重新发布。
5. Webhook 下游提供按投递 ID 查询 inbox 状态的能力，供 AInxiety 对丢失确认进行核对。只出站的下游通过恢复接口主动报告本地状态。

过期快讯不逐条补发，保留记录并进入报告候选；日报、周报按配置的恢复范围补投。跨日恢复重新申请当天配额，重试同一次投递不重复计数；结果未知继续占用相应额度直至核对。过期规则仅能直接终止未执行任务，已开始任务需要协调停止或核对结果。

历史内容查询如果后续提供，应为独立只读接口，支持游标分页；读取历史不创建或确认 Delivery。投递记录保留期覆盖允许的离线恢复窗口，超出可恢复范围时明确返回状态，不静默重建任务。

回执需幂等，重复成功回执不能触发二次发送。过期领取令牌不得覆盖新状态。租约到期后必须考虑消费者是否已发送；不对可能发生过外部副作用的任务盲目重新投递。

消费者在发送前持久化投递尝试记录，并按 `idempotency_key` 去重。若下游支持幂等请求或查询，优先利用；若不支持，则无法承诺严格 exactly-once。发送结果未知时进入核对状态，采用渠道特定恢复或人工处理。

待投递记录与发布决策在同一个数据库事务中创建。网络投递在事务外进行，成功后存回执。博客端拟用外部发布 ID 唯一约束，做到重试返回原文章而非重复创建。

## 反馈入口

`POST /v1/feedback`

字段：`external_feedback_id`、`publication_id`、可选 `event_id/item_ref`、`type`、`value`、`occurred_at`。调用者和反馈者身份由可信认证及 Bot 映射校验；只有已配置的操作者可以修改全局偏好。

反馈类型：`useful`、`low_value`、`duplicate`、`more_topic`、`less_topic`、`factual_error`。重复请求使用外部反馈 ID 去重。报告级反馈不默认扩散到全部条目。

## 凭据与边界

服务间采用独立受限凭据，限定消费者与目标渠道；凭据可撤销、轮换。公网传输使用 TLS。模型节点不持有 QQ 或博客发布凭据，来源正文不能改变目的地或权限。
