# Redis 使用说明

本文档基于当前仓库代码实现梳理，目标是回答两个问题：

1. 这个项目里 Redis **实际缓存了什么**。
2. Redis 在这个项目里 **怎么接入、怎么工作、怎么排查和怎么使用**。

结论先说：**当前代码里，Redis 主要承担 4 类职责：JWT Token 缓存、Feed 列表缓存、视频详情缓存、热视频榜单计算/分页；另外还承担缓存击穿保护用的分布式锁。**

## 1. Redis 在项目里的定位

Redis 是**可选依赖**，不是核心单点：

- API 进程启动时会尝试连接 Redis。
- Worker 进程启动时会尝试连接 Redis。
- 如果 Redis 不可用，系统会降级：
  - 鉴权回退 MySQL；
  - Feed/视频详情缓存失效后回退 DB；
  - 热榜查询回退 MySQL 的 `popularity` 排序逻辑；
  - 热度增量异步更新不可用时，不影响核心 MySQL 数据落库。

这意味着 Redis 主要负责**性能优化和热点查询加速**，不是业务正确性的唯一来源。

## 2. 连接方式与配置

当前配置文件：

- `D:\MyProject\Douyin\feedsystem_video_go\backend\configs\config.yaml`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\configs\config.docker.yaml`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\configs\config.bench.yaml`

Redis 配置项只有 4 个：

```yaml
redis:
  host: localhost
  port: 6379
  password: 123456
  db: 0
```

代码入口：

- `backend/cmd/main.go`：API 进程连接 Redis。
- `backend/cmd/worker/main.go`：Worker 进程连接 Redis。
- `backend/internal/middleware/redis/redis.go`：Redis 客户端封装。

当前使用的是 `github.com/redis/go-redis/v9`。

## 3. 实际用了哪些 Redis Key

当前实现中，Redis Key 可以分成 5 类。

### 3.1 账号 Token 缓存

**用途**：加速 JWT 鉴权，减少每次请求都查 MySQL。

- Key 模式：`account:<accountID>`
- 类型：`STRING`
- Value：当前账号最新有效 JWT token
- TTL：`24h`

典型例子：

- `account:12` -> `eyJhbGciOiJIUzI1NiIs...`

写入时机：

- 登录成功后写入
- 改名后生成新 token，并覆盖写入
- Redis 未命中但 MySQL 校验成功时，会自动回填

删除时机：

- 登出时删除
- 改密码时会调用登出，因此也会删除

读取时机：

- `JWTAuth` / `SoftJWTAuth` 中间件优先读 Redis
- 如果 Redis 不可用或未命中，则回退 MySQL 的 `account.token`

这部分的特点是：**Redis 是鉴权加速层，MySQL 才是兜底真相源。**

---

### 3.2 Feed 最新列表缓存

**用途**：缓存匿名用户最新视频流，减少首页热点查询压力。

- Key 模式：`feed:listLatest:limit=<n>:before=<unix>`
- 类型：`STRING`
- Value：`ListLatestResponse` 的 JSON
- TTL：`5s`

典型例子：

- `feed:listLatest:limit=30:before=0`
- `feed:listLatest:limit=30:before=1719932000`

说明：

- 只有 `viewerAccountID == 0` 时才使用这个缓存，也就是**匿名流缓存**。
- 已登录用户请求 `listLatest` 时不会走这个 key。
- 缓存非常短，只保留 5 秒，属于典型的短时热点缓存。

---

### 3.3 Feed 关注列表缓存

**用途**：缓存已登录用户的关注流。

- Key 模式：`feed:listByFollowing:limit=<n>:accountID=<uid>:before=<unix>`
- 类型：`STRING`
- Value：`ListByFollowingResponse` 的 JSON
- TTL：`5s`

典型例子：

- `feed:listByFollowing:limit=20:accountID=12:before=0`

说明：

- 只有登录用户调用 `/feed/listByFollowing` 时才会使用。
- 这是**按用户维度隔离**的缓存，不同用户的关注流不会混用。
- TTL 同样很短，更多是防止短时间内重复回源 DB。

---

### 3.4 视频详情缓存

**用途**：缓存单个视频详情，减少详情页重复回源数据库。

- Key 模式：`video:detail:id=<videoID>`
- 类型：`STRING`
- Value：`Video` 的 JSON
- TTL：`5m`

典型例子：

- `video:detail:id=101`

写入时机：

- `GetDetail` 缓存未命中后，从 MySQL 读取并回填 Redis

删除时机：

- 删除视频时主动删除
- 视频热度更新时主动删除
- 点赞/取消点赞/评论导致热度变化时，也会删这个 key
- Worker 消费热度消息时也会删这个 key

说明：

- 这是当前项目里生命周期最长的缓存项。
- 它的失效策略是**主动删除 + 读时回填**。

---

### 3.5 热榜分钟窗口与热榜快照

这部分不是传统“对象缓存”，而是 Redis 作为**排行榜计算存储**使用。

#### 3.5.1 分钟窗口热度桶

- Key 模式：`hot:video:1m:<yyyyMMddHHmm>`
- 类型：`ZSET`
- Member：`videoID`
- Score：该分钟窗口内的热度增量累计值
- TTL：`2h`

典型例子：

- `hot:video:1m:202604011230`

热度增量来源：

- 点赞：`+1`
- 取消点赞：`-1`
- 评论：`+1`
- 显式调用 `UpdatePopularity`：按传入增量写入

这类 key 的作用是：**把热度变化按分钟切片累积起来，而不是直接维护一个长期单榜单。**

#### 3.5.2 热榜聚合快照

- Key 模式：`hot:video:merge:1m:<yyyyMMddHHmm>`
- 类型：`ZSET`
- 含义：把最近 60 个 `hot:video:1m:*` 窗口做 `ZUNIONSTORE SUM` 聚合后的快照
- TTL：`2m`

典型例子：

- `hot:video:merge:1m:202604011230`

说明：

- `/feed/listByPopularity` 查询时，会以 `as_of` 所在分钟为基准，回看最近 60 分钟。
- 如果该时刻快照不存在，就临时聚合生成一个快照 key。
- 然后用 `ZREVRANGE` 按 offset 分页读成员。
- 读到视频 ID 后，再回 MySQL 查询视频详情，最后按 Redis 返回顺序组装响应。

这意味着：

- Redis 负责**排行榜计算、稳定分页顺序和高频排序读**；
- MySQL 负责**视频实体信息和兜底排序**。

---

### 3.6 防击穿锁 Key

**用途**：避免缓存刚过期时，大量并发同时打到 MySQL。

- Key 模式：`lock:<业务缓存Key>`
- 类型：`STRING`
- Value：随机 token
- TTL：
  - Feed 列表：`500ms`
  - 视频详情：`2s`

典型例子：

- `lock:feed:listLatest:limit=30:before=0`
- `lock:feed:listByFollowing:limit=20:accountID=12:before=0`
- `lock:video:detail:id=101`

实现方式：

- 加锁：`SETNX + TTL`
- 解锁：Lua 脚本比对 token 后删除，避免误删别人的锁

说明：

- 这些 key 不是业务数据，不用于持久化展示。
- 它们属于**并发控制辅助 key**。

## 4. 当前代码里 Redis 的完整职责清单

按“实际代码已落地”的口径，当前 Redis 用途如下：

| 类别 | Key | 类型 | TTL | 当前状态 |
|---|---|---:|---:|---|
| Token 缓存 | `account:<id>` | STRING | 24h | 已实现 |
| 匿名最新 Feed 缓存 | `feed:listLatest:...` | STRING | 5s | 已实现 |
| 关注 Feed 缓存 | `feed:listByFollowing:...` | STRING | 5s | 已实现 |
| 视频详情缓存 | `video:detail:id=<id>` | STRING | 5m | 已实现 |
| 分钟热度桶 | `hot:video:1m:<minute>` | ZSET | 2h | 已实现 |
| 热榜聚合快照 | `hot:video:merge:1m:<minute>` | ZSET | 2m | 已实现 |
| 防击穿锁 | `lock:<cacheKey>` | STRING | 500ms/2s | 已实现 |

**当前没有看到的内容**：

- 没有把评论列表放进 Redis
- 没有把点赞关系放进 Redis
- 没有把关注关系放进 Redis
- 没有使用 Hash / List / Set 保存业务实体
- 没有做 Redis Stream / PubSub

## 5. Redis 是怎么被业务使用的

### 5.1 鉴权流程

1. 客户端带 `Authorization: Bearer <token>` 请求。
2. 中间件先解析 JWT。
3. 优先读取 `account:<id>`。
4. 如果 Redis 命中且 token 一致，直接通过。
5. 如果 Redis 未命中或异常，则回退 MySQL 查询 `account.token`。
6. MySQL 校验成功后，再把 token 回填到 Redis。

这是一种典型的：**缓存优先，DB 自愈兜底**。

### 5.2 视频详情缓存流程

1. 读取 `video:detail:id=<id>`。
2. 命中则直接返回。
3. 未命中时先尝试抢 `lock:video:detail:id=<id>`。
4. 抢到锁的请求回源 MySQL，并把结果写回 Redis。
5. 没抢到锁的请求短暂等待，重试读缓存。
6. 仍未拿到时，再走 DB。

这部分实现了**缓存击穿保护**。

### 5.3 Feed 缓存流程

`/feed/listLatest` 与 `/feed/listByFollowing` 的逻辑类似：

1. 根据分页参数拼接缓存 key。
2. 命中直接返回 JSON。
3. 未命中则尝试抢 `lock:<cacheKey>`。
4. 抢到锁后查 DB，构建响应，写缓存。
5. 没抢到锁则短暂等待其他请求回填。
6. 等待失败后仍会回退 DB。

### 5.4 热榜流程

1. 点赞/评论/取消点赞等事件产生热度增量。
2. 增量写入当前分钟 ZSET：`hot:video:1m:<minute>`。
3. 查询 `/feed/listByPopularity` 时，取最近 60 个分钟窗口。
4. 用 `ZUNIONSTORE` 聚合为 `hot:video:merge:1m:<as_of>`。
5. 用 `ZREVRANGE` 分页读取视频 ID。
6. 再回 MySQL 取视频详情，并按 Redis 顺序返回。

这个设计的重点不是“缓存整页结果”，而是**把排序问题交给 Redis 做**。

## 6. Redis 与 RabbitMQ / Worker 的关系

Redis 的热榜更新有两种路径。

### 6.1 正常路径：走 MQ 异步更新 Redis

- 点赞/评论接口先发布 `video.popularity.update` 事件。
- Worker 消费该事件。
- Worker 调用 `UpdatePopularityCache`：
  - 删除 `video:detail:id=<id>`
  - 更新 `hot:video:1m:<minute>` ZSET
  - 设置 2 小时过期

### 6.2 降级路径：MQ 失败时同步更新 Redis

如果发布 popularity MQ 失败，则业务线程直接调用 `UpdatePopularityCache`。

因此即使 RabbitMQ 有问题，只要 Redis 正常，热榜缓存仍然会被更新。

## 7. 当前实现里的几个重要细节

### 7.1 Redis 不可用时，系统不会整体不可用

当前代码大量写了 `if cache != nil` 分支，说明 Redis 被当作增强能力而不是强依赖。

### 7.2 Feed 缓存与详情缓存都做了防击穿

这是当前 Redis 使用里最有价值的部分之一。否则首页和热门详情在高并发下会直接把 MySQL 打穿。

### 7.3 热榜分页是“快照分页”

`listByPopularity` 不直接对实时变动中的分钟桶分页，而是先聚合出 `hot:video:merge:1m:<as_of>`，再按 offset 取数据。这样分页顺序更稳定。

### 7.4 当前 Feed 缓存失效策略偏保守，更多依赖短 TTL

当前代码里：

- 视频发布/删除后，没有看到对 `feed:listLatest:*` 或 `feed:listByFollowing:*` 的批量主动失效；
- 关注/取关后，也没有看到对关注流缓存的主动失效；
- 这意味着 Feed 缓存一致性主要依赖 **5 秒短 TTL**。

这不是错误，但要明确：**当前实现更偏向“短缓存容忍轻微陈旧”，而不是严格实时失效”。**

## 8. 如何在本地查看 Redis 实际内容

如果你是 Docker Compose 启动：

```powershell
docker compose exec redis redis-cli -a 123456
```

进入后可以这样看：

```redis
KEYS account:*
KEYS feed:listLatest:*
KEYS feed:listByFollowing:*
KEYS video:detail:*
KEYS hot:video:1m:*
KEYS hot:video:merge:1m:*
KEYS lock:*
```

查看具体值：

```redis
GET account:12
GET video:detail:id=101
GET feed:listLatest:limit=30:before=0
TTL video:detail:id=101
ZRANGE hot:video:1m:202604011230 0 -1 WITHSCORES
ZREVRANGE hot:video:merge:1m:202604011230 0 9 WITHSCORES
```

如果你本机直接跑 Redis：

```powershell
redis-cli -h 127.0.0.1 -p 6379 -a 123456
```

## 9. 如何正确使用这个项目里的 Redis

### 9.1 启动方式

最简单方式是用 Compose：

```powershell
docker compose up -d mysql redis rabbitmq
```

然后启动后端和 worker。

如果不用 Docker，也至少要保证：

- Redis 地址与 `backend/configs/config.yaml` 一致
- 密码与配置一致
- API 和 worker 都能连到同一个 Redis 实例

### 9.2 新增缓存时建议遵守的模式

如果后续你要继续在这个项目里加 Redis，建议保持当前风格：

- **对象缓存**：用 `STRING + JSON`
- **短列表缓存**：用 `STRING + JSON + 短 TTL`
- **热点保护**：统一配 `lock:<cacheKey>`
- **排行榜**：继续用 `ZSET`
- **一致性策略**：优先“主动删缓存”，不要做复杂更新
- **降级策略**：Redis 异常时必须能回退 MySQL

### 9.3 不建议直接依赖 Redis 作为唯一真相源

当前工程并不是 Redis-first 架构：

- Token 最终还是以 MySQL 为兜底；
- 视频详情最终还是 MySQL；
- 热榜只是 Redis 算排序，视频实体仍来自 MySQL；
- MQ/Redis 异常时，都有同步降级路径。

所以如果你后续扩展功能，最好继续保持这个原则。

## 10. 代码定位索引

如果你要继续看代码，最关键的文件是：

- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\middleware\redis\redis.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\middleware\redis\cache.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\middleware\redis\zset.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\middleware\jwt\jwt.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\account\service.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\feed\service.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\video\video_service.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\video\like_service.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\video\comment_service.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\video\popularity_cache.go`
- `D:\MyProject\Douyin\feedsystem_video_go\backend\internal\worker\popularityworker.go`

## 11. 一句话总结

当前项目里 Redis **不是“全量缓存数据库”**，而是一个**面向热点读和榜单计算的性能层**：

- `account:<id>`：JWT token 缓存
- `feed:listLatest:*`：匿名最新流缓存
- `feed:listByFollowing:*`：关注流缓存
- `video:detail:id=*`：视频详情缓存
- `hot:video:1m:*` / `hot:video:merge:1m:*`：热视频榜单计算与分页
- `lock:*`：防击穿分布式锁

这就是当前代码里 Redis **实际**承担的全部核心职责。
