# 项目中的 RabbitMQ 使用说明

本文基于当前仓库实现说明这个项目里 RabbitMQ 的用途、接入方式、消息拓扑、运行方式，以及业务开发时应该怎么使用。

## 1. RabbitMQ 在这个项目里解决什么问题

这个项目把 RabbitMQ 用作业务事件总线，主要承担两类职责：

1. 将接口请求和真正的数据落库/热度更新解耦。
2. 让点赞、评论、关注、热度更新这些高频写操作可以异步消费，降低接口阻塞时间。

当前实现中，RabbitMQ 不是全局强依赖，而是“可选启用”：

- API 进程启动时，如果 RabbitMQ 连不上，服务仍然可以启动。
- Worker 进程依赖 RabbitMQ；它负责真正消费消息并执行异步逻辑。
- 点赞、评论、热度更新都实现了“MQ 失败时本地直写”的降级。
- 关注模块目前是“发布 MQ + 直接写库”并存，不是纯异步模式。

## 2. 整体架构

项目里有两个和 RabbitMQ 直接相关的进程：

- API 进程：`backend/cmd/main.go`
- Worker 进程：`backend/cmd/worker/main.go`

工作方式如下：

1. 用户调用 HTTP 接口。
2. API 进程根据业务类型发布 RabbitMQ 消息。
3. Worker 进程消费对应队列。
4. Worker 把消息落到 MySQL，或者更新 Redis 热度缓存。
5. 如果 API 发布失败，部分业务会直接回退到同步写库/写缓存。

## 3. RabbitMQ 连接配置

本地默认配置在 `backend/configs/config.yaml`：

```yaml
rabbitmq:
  host: localhost
  port: 5672
  username: admin
  password: password123
```

Docker Compose 默认服务：

- AMQP 端口：`5672`
- 管理台端口：`15672`
- 默认账号：`admin`
- 默认密码：`password123`

Compose 里的服务定义在根目录 `docker-compose.yml`，镜像是：

```yaml
rabbitmq:
  image: rabbitmq:3-management
```

其中 `3-management` 版本自带 Web 管理界面，便于查看 exchange、queue、消息堆积和消费者状态。

## 4. 当前定义的消息拓扑

项目统一使用 `topic exchange`。声明逻辑在：

- API 侧公共封装：`backend/internal/middleware/rabbitmq/rabbitMQ.go`
- Worker 侧拓扑声明：`backend/cmd/worker/main.go`

### 4.1 点赞事件

- Exchange: `like.events`
- Queue: `like.events`
- Binding Key: `like.*`
- Routing Key:
  - `like.like`
  - `like.unlike`

消息结构：

```json
{
  "event_id": "随机事件ID",
  "action": "like|unlike",
  "user_id": 1,
  "video_id": 2,
  "occurred_at": "2026-04-01T12:00:00Z"
}
```

### 4.2 评论事件

- Exchange: `comment.events`
- Queue: `comment.events`
- Binding Key: `comment.*`
- Routing Key:
  - `comment.publish`
  - `comment.delete`

消息结构：

```json
{
  "event_id": "随机事件ID",
  "action": "publish|delete",
  "comment_id": 10,
  "username": "alice",
  "video_id": 2,
  "author_id": 1,
  "content": "hello",
  "occurred_at": "2026-04-01T12:00:00Z"
}
```

### 4.3 关注事件

- Exchange: `social.events`
- Queue: `social.events`
- Binding Key: `social.*`
- Routing Key:
  - `social.follow`
  - `social.unfollow`

消息结构：

```json
{
  "event_id": "随机事件ID",
  "action": "follow|unfollow",
  "follower_id": 1,
  "vlogger_id": 2,
  "occurred_at": "2026-04-01T12:00:00Z"
}
```

### 4.4 视频热度事件

- Exchange: `video.popularity.events`
- Queue: `video.popularity.events`
- Binding Key: `video.popularity.*`
- Routing Key:
  - `video.popularity.update`

消息结构：

```json
{
  "event_id": "随机事件ID",
  "video_id": 2,
  "change": 1,
  "occurred_at": "2026-04-01T12:00:00Z"
}
```

## 5. 公共 RabbitMQ 封装是怎么做的

核心封装在 `backend/internal/middleware/rabbitmq/rabbitMQ.go`。

它提供了几个基础能力：

- `NewRabbitMQ`：建立 AMQP 连接和 channel。
- `DeclareTopic`：声明 `topic exchange`、队列，并完成绑定。
- `PublishJSON`：把结构体序列化成 JSON 后发布。

当前消息发布有几个固定特征：

- exchange 类型统一为 `topic`
- exchange 为 durable
- queue 为 durable
- 消息 `DeliveryMode` 为 `Persistent`
- 每条消息都有 JSON body
- 每条业务事件都会生成 `event_id`

这意味着：

- RabbitMQ 重启后，交换机/队列定义仍会保留。
- 已持久化的消息在 broker 正常持久化后不会因重启直接丢失。
- 消费端可以按事件类型拆分路由键，后续扩展统计、风控、审计消费者会比较容易。

## 6. API 进程里怎么使用 RabbitMQ

API 启动入口在 `backend/cmd/main.go`。

启动时逻辑是：

1. 加载配置。
2. 连接 MySQL。
3. 尝试连接 Redis。
4. 尝试连接 RabbitMQ。
5. 构造 Router 和业务 Service。

如果 RabbitMQ 连接失败：

- 不会让 API 直接退出。
- 会打印 `RabbitMQ config error (disabled)`。
- 后续路由初始化 `LikeMQ`、`CommentMQ`、`SocialMQ`、`PopularityMQ` 时会失败并置空。
- Service 进入降级路径，直接写 MySQL 或 Redis。

对应装配位置在 `backend/internal/http/router.go`。

## 7. Worker 进程里怎么使用 RabbitMQ

Worker 启动入口在 `backend/cmd/worker/main.go`。

它和 API 的区别是：

- Worker 会主动声明 RabbitMQ 拓扑。
- Worker 为每类业务启动消费者组。
- 每个消费者使用独立 channel。
- 每个 channel 设置了 `Qos(prefetch, 0, false)`。

当前默认并发配置：

```yaml
worker:
  prefetch: 32
  social_concurrency: 1
  like_concurrency: 8
  comment_concurrency: 4
  popularity_concurrency: 8
```

这表示：

- social worker 启动 1 个消费者
- like worker 启动 8 个消费者
- comment worker 启动 4 个消费者
- popularity worker 启动 8 个消费者
- 每个消费者单独消费，对应 channel 最多预取 32 条未确认消息

## 8. 各业务具体怎么用 RabbitMQ

### 8.1 点赞

入口：

- 生产者：`backend/internal/video/like_service.go`
- 消费者：`backend/internal/worker/likeworker.go`

#### Like 流程

1. API 校验 `video_id`、`account_id`。
2. 调用 `likeMQ.Like(...)` 发布 `like.like`。
3. 调用 `popularityMQ.Update(...)` 发布热度变化。
4. 如果两个消息都成功发布，接口直接返回成功。
5. Worker 异步消费 `like.events`：
   - 插入点赞记录
   - 更新 `likes_count`
   - 更新 MySQL 里的 `popularity`
6. popularity worker 异步消费 `video.popularity.events`，更新 Redis 热榜缓存。

#### Unlike 流程

和 Like 类似，只是动作反过来：

- 删除点赞记录
- `likes_count - 1`
- `popularity - 1`
- Redis 热度缓存减 1

#### 点赞的降级策略

如果 `like.events` 发布失败：

- API 会直接走 MySQL 事务写入：
  - 创建/删除点赞记录
  - 修改 `likes_count`
  - 修改 MySQL 的 `popularity`

如果 `video.popularity.events` 发布失败：

- API 会直接更新 Redis 热度缓存。

这部分是当前项目里 RabbitMQ 使用最完整的一块。

### 8.2 评论

入口：

- 生产者：`backend/internal/video/comment_service.go`
- 消费者：`backend/internal/worker/commentworker.go`

#### 发布评论流程

1. API 校验 `video_id`、`author_id`、`content`。
2. 发布 `comment.publish`。
3. 发布 `video.popularity.update`。
4. 两个消息都成功时，接口直接返回。
5. comment worker 异步处理：
   - 写入评论表
   - MySQL `popularity + 1`
6. popularity worker 异步更新 Redis 热度缓存。

#### 删除评论流程

1. API 先查评论是否存在，以及操作者是否为评论作者。
2. 如果 `comment.delete` 发布成功，则接口直接返回。
3. comment worker 消费后执行删除。
4. 如果 MQ 不可用，则 API 直接删库。

#### 评论的降级策略

发布评论时：

- 评论消息发送失败，会直接写 MySQL。
- 热度消息发送失败，会直接更新 Redis。

需要注意的当前实现细节：

- `comment.publish` 会增加 MySQL 热度。
- `comment.delete` 当前只删除评论，没有同步扣减 MySQL 热度，也没有单独发热度回滚事件。

也就是说，文档里如果描述“删除评论会同步降低热度”，那与当前代码不一致；当前实现并没有做这一步。

### 8.3 关注

入口：

- 生产者：`backend/internal/social/service.go`
- 消费者：`backend/internal/worker/socialworker.go`

#### 当前行为

关注模块和点赞/评论不一样，它不是“消息成功就完全交给 worker”，而是：

1. API 校验用户存在、不能关注自己、不能重复关注。
2. 如果配置了 `socialMQ`，会尝试发 `social.follow` 或 `social.unfollow`。
3. 无论消息发布是否成功，API 都会继续直接操作 MySQL。
4. social worker 也会消费同一条事件并再次尝试执行关注/取关。

因此当前 social 模块更接近：

- 同步直写为主
- MQ 作为附加异步事件流

为了避免副作用，worker 对重复 follow 做了幂等处理：

- follow 如果命中 MySQL 唯一键冲突，会直接忽略
- unfollow 对不存在关系的删除通常也不会报业务错误

如果后续希望 social 模块和 like/comment 一样彻底异步化，需要把 service 里的直接写库逻辑改成“仅在 MQ 发布失败时回退”。

### 8.4 热度缓存

入口：

- 生产者：
  - `backend/internal/video/like_service.go`
  - `backend/internal/video/comment_service.go`
  - `backend/internal/video/video_service.go`
- 消费者：`backend/internal/worker/popularityworker.go`

这个队列只负责 Redis 热度缓存，不负责 MySQL 主数据。

消息被消费后会调用：

```go
video.UpdatePopularityCache(ctx, w.cache, evt.VideoID, evt.Change)
```

也就是说，`video.popularity.events` 的目标是维护 Redis 里的热榜增量，而不是直接维护视频表。

## 9. 消费确认与失败重试

所有 worker 都使用手动确认：

- 处理成功：`Ack`
- 处理失败：`Nack(false, true)`

这表示失败消息会重新入队。

当前行为总结：

- JSON 解析失败：直接忽略并 `Ack`
- 参数非法：直接忽略并 `Ack`
- 数据库/Redis 等真正处理失败：`Nack` 并重新入队

这样设计的好处：

- 坏格式消息不会无限重试
- 暂时性故障有机会恢复

需要注意：

- 当前没有死信队列
- 当前没有最大重试次数
- 如果消息因为逻辑错误一直失败，可能出现反复重投

## 10. 本地怎么启动和使用 RabbitMQ

### 10.1 只启动依赖

在项目根目录执行：

```bash
docker compose up -d mysql redis rabbitmq
```

然后分别启动：

```bash
cd backend
go run ./cmd
```

```bash
cd backend
go run ./cmd/worker
```

访问 RabbitMQ 管理台：

- [http://localhost:15672](http://localhost:15672)

登录账号：

- 用户名：`admin`
- 密码：`password123`

### 10.2 直接用 Compose 启整个系统

```bash
docker compose up -d --build
```

这会启动：

- mysql
- redis
- rabbitmq
- backend
- worker
- frontend

## 11. 在管理台里应该看到什么

启动 API 或 Worker 并触发过相关路由后，管理台里通常能看到这些 exchange：

- `like.events`
- `comment.events`
- `social.events`
- `video.popularity.events`

通常能看到这些 queue：

- `like.events`
- `comment.events`
- `social.events`
- `video.popularity.events`

如果 worker 正常运行，对应队列会看到消费者数量大于 0。

排查时重点看：

- Queues 页面是否有消息堆积
- Consumers 数量是否符合配置
- Ready / Unacked 是否异常升高
- Connections / Channels 是否持续存在

## 12. 业务开发时怎么继续使用 RabbitMQ

如果要新增一个新的异步事件，建议沿用当前模式：

1. 在 `backend/internal/middleware/rabbitmq/` 下新增一个业务 MQ 封装。
2. 定义：
   - exchange
   - queue
   - binding key
   - routing key
   - event struct
3. 在 Router 中初始化对应 MQ 对象。
4. 在 Service 中优先发布消息。
5. 视业务需要决定是否做“发布失败回退直写”。
6. 在 `backend/internal/worker/` 下新增 worker。
7. 在 `backend/cmd/worker/main.go` 里声明拓扑并启动消费者组。

建议遵守以下规则：

- 一个业务域用一个 exchange，便于扩展消费者
- 每条消息带 `event_id`
- 消费逻辑尽量幂等
- 明确哪些失败应该重试，哪些应该直接丢弃
- 需要长期稳定运行时，补 dead-letter queue

## 13. 当前实现的几个重要注意点

### 13.1 RabbitMQ 对 API 不是强依赖

这是当前项目的设计选择。优点是本地开发方便，缺点是：

- “启用 MQ”和“禁用 MQ”两种模式都要保证逻辑一致
- 降级路径需要持续维护，否则容易出现行为漂移

### 13.2 social 模块不是纯异步

当前 social service 会：

- 发 MQ
- 同时直接写 MySQL

这不是标准的“生产者-消费者完全解耦”模型。文档和面试描述里最好明确说“social 目前是混合模式”。

### 13.3 评论删除没有回滚热度

当前 `comment.delete` 只删除评论，不会：

- 扣减 MySQL `popularity`
- 发布热度回滚事件
- 更新 Redis 热榜

如果你后续要把“评论数/热度”做严格一致，这里需要补齐。

### 13.4 没有死信和重试上限

目前适合课程设计、演示和中小规模场景；如果要继续工程化，建议补：

- dead-letter exchange
- retry queue
- poison message 隔离
- 消费监控和告警

## 14. 一句话总结怎么使用

这个项目里 RabbitMQ 的正确使用方式是：

- API 作为生产者发布业务事件
- Worker 作为消费者异步落库或更新 Redis
- 点赞、评论、热度更新走“优先 MQ，失败降级”
- 关注目前是“同步直写 + MQ 事件并存”

如果你只是要把项目跑起来：

1. 启动 `rabbitmq`
2. 启动 `backend`
3. 启动 `worker`
4. 调用点赞/评论/关注相关接口
5. 在 RabbitMQ 管理台观察 exchange、queue、consumer 和消息堆积情况
