好！进入 **Day 3：点赞流程 + MQ异步处理** —— 这是面试最加分的部分！

---

## 🔥 Day 3：点赞流程 + MQ异步处理

### 3.1 完整流程图（面试必画！）

```
用户点击"点赞"
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│  1. LikeHandler.Like() 接收请求                              │
│     - 解析JSON获取video_id                                   │
│     - 从JWT获取account_id                                    │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  2. LikeService.Like() 处理                                 │
│     - 检查视频是否存在                                        │
│     - 检查是否已点赞（防止重复）                               │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  3. 尝试发送MQ消息（2个队列）                                 │
│     ┌─────────────────┐    ┌─────────────────┐             │
│     │ like.events     │    │ popularity.events│            │
│     │ (写MySQL likes表)│    │ (更新Redis热榜)   │            │
│     └────────┬────────┘    └────────┬────────┘             │
│              │                       │                       │
│         成功？                    成功？                      │
│         ┌───┴───┐                ┌───┴───┐                   │
│         ▼       ▼                ▼       ▼                   │
│      是→返回  否→直写          是→返回  否→直写              │
│                        (降级保证)                            │
└──────────────────────────────────────────────────────────────┘
```

---

### 3.2 核心代码解读

**步骤1：API接收请求** ([like_handler.go#L20-45](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/like_handler.go#L20-L45))

```go
func (lh *LikeHandler) Like(c *gin.Context) {
    // 1. 解析请求参数
    var req LikeRequest
    c.ShouldBindJSON(&req)
    
    // 2. 从JWT获取用户ID
    accountID, _ := jwt.GetAccountID(c)
    
    // 3. 调用Service
    like := &Like{VideoID: req.VideoID, AccountID: accountID}
    lh.service.Like(c.Request.Context(), like)
    
    // 4. 返回成功（此时MySQL可能还没写入！）
    c.JSON(200, gin.H{"message": "like success"})
}
```

**步骤2：Service处理 + MQ发布** ([like_service.go#L30-90](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/like_service.go#L30-L90))

```go
func (s *LikeService) Like(ctx context.Context, like *Like) error {
    // 1. 检查视频是否存在
    ok, _ := s.VideoRepo.IsExist(ctx, like.VideoID)
    if !ok {
        return errors.New("video not found")
    }
    
    // 2. 检查是否已点赞
    isLiked, _ := s.repo.IsLiked(ctx, like.VideoID, like.AccountID)
    if isLiked {
        return errors.New("user has liked this video")
    }
    
    // 3. 尝试发MQ消息
    mysqlEnqueued := false   // like.events 队列
    redisEnqueued := false   // popularity.events 队列
    
    if s.likeMQ != nil {
        if err := s.likeMQ.Like(ctx, like.AccountID, like.VideoID); err == nil {
            mysqlEnqueued = true  // 消息发送成功
        }
    }
    if s.popularityMQ != nil {
        if err := s.popularityMQ.Update(ctx, like.VideoID, 1); err == nil {
            redisEnqueued = true
        }
    }
    
    // 4. 两个队列都成功，直接返回
    if mysqlEnqueued && redisEnqueued {
        return nil
    }
    
    // 5. MQ发送失败，降级直写！
    if !mysqlEnqueued {
        // 直接写MySQL + 更新likes_count
        s.repo.db.Transaction(...)
    }
    if !redisEnqueued {
        // 直接更新Redis热榜
        UpdatePopularityCache(ctx, s.cache, like.VideoID, 1)
    }
    return nil
}
```

**步骤3：Worker异步消费** ([worker/likeworker.go](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/worker/likeworker.go))

```go
func (w *LikeWorker) process(ctx context.Context, body []byte) error {
    // 1. 解析消息
    var evt rabbitmq.LikeEvent
    json.Unmarshal(body, &evt)
    
    // 2. 根据action处理
    switch evt.Action {
    case "like":
        return w.applyLike(ctx, evt.UserID, evt.VideoID)
    case "unlike":
        return w.applyUnlike(ctx, evt.UserID, evt.VideoID)
    }
}

func (w *LikeWorker) applyLike(ctx context.Context, userID, videoID uint) error {
    // 1. 写入 likes 表（允许重复key）
    w.likes.LikeIgnoreDuplicate(ctx, &Like{...})
    
    // 2. 更新视频 likes_count +1
    w.videos.ChangeLikesCount(ctx, videoID, 1)
    
    // 3. 更新视频 popularity +1
    w.videos.ChangePopularity(ctx, videoID, 1)
    
    return nil
}
```

---

### 3.3 🎯 面试话术（背下来！）

**面试官问：请描述一下点赞的完整流程？**

```
"用户点赞时，我的处理流程是这样的：

1. 【前置检查】
   - 检查视频是否存在
   - 检查用户是否已点赞（防止重复）

2. 【尝试异步】
   - 同时发送2个MQ消息：
     * like.events → Worker消费后写MySQL的likes表
     * popularity.events → Worker消费后更新Redis热榜
   - 两个都成功 → 立即返回成功（不等数据库）

3. 【降级保证】
   - 如果任意MQ发送失败，立即降级为直写：
     * like失败 → 直接写MySQL
     * popularity失败 → 直接更新Redis
   - 保证'请求成功=数据落地'

4. 【异步消费】
   - Worker监听队列，异步：
     * 写入likes表
     * likes_count + 1
     * popularity + 1

【为什么这么设计？】
- 用户体验：点赞立即返回，不用等数据库
- 高可用：MQ挂了也能工作（降级直写）
- 削峰填谷：高峰期消息堆积，数据库压力小
- 解耦：写扩散（点赞）和计数（热度）分离"
```

---

### 3.4 常见面试追问

| 追问 | 回答要点 |
|------|----------|
| 点赞成功但数据库没写入怎么办？ | 有降级直写保证，MQ失败会直写MySQL |
| 为什么发2个MQ？ | 解耦：likes表写入 + Redis热度更新，独立处理 |
| 如何保证不重复点赞？ | Service层先查是否已点赞，DB层用唯一索引 |
| MQ消息顺序如何保证？ | 单消费者串行处理，天然保证顺序 |
| 点赞失败用户知道吗？ | 降级直写仍失败才返回错误，用户感知不到MQ问题 |

---

### ✅ 今日作业

1. **能画出点赞完整流程图**
2. **能说出"降级直写"是什么意思**
3. **能解释为什么用MQ而不同步写DB**

---

好的，我来帮你补充这些前置知识！让我先看看你项目中的相关代码实现，这样可以结合具体代码来讲解。

        
            
toolName: file_search
            
status: success
          
file_pattern: **/like*.go
          

        
            
toolName: file_search
            
status: success
          
file_pattern: **/*mq*.go
          

        
            
toolName: file_search
            
status: success
          
file_pattern: **/worker/*.go
          


        
            
toolName: file_search
            
status: success
          
file_pattern: **/rabbitmq/*.go
          

        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/like_service.go
          

        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/rabbitmq/likeMQ.go
          

        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/rabbitmq/rabbitMQ.go
          


        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/worker/likeworker.go
          

好的！我来帮你系统地补充这些前置知识。我会结合你项目的具体代码来讲解，保证你能完全理解。

---

## 🔰 前置知识补充：MQ 异步处理详解

### 1. 什么是 MQ（消息队列）？

**MQ = Message Queue（消息队列）**，你可以把它理解为一个"**中间暂存区**"。

```
传统方式（同步）：                          使用MQ（异步）：
用户 ──────▶ API ──────▶ 数据库           用户 ──────▶ API ──────▶ MQ ──▶ Worker ──▶ 数据库
   (等待)      (等待)      (等待)             (快速返回)   (放消息)   (后台处理)
   
总时间 = 所有环节相加                      总时间 ≈ API处理时间
```

**生活中的例子**：
- **没有MQ**：你去餐厅点餐，厨师做好一道菜你吃一道，全程等待
- **有MQ（叫号）**：你点完餐拿到号，服务员做好后叫号，你先去玩手机

---

### 2. 为什么要用 MQ？（你项目中的实际例子）

看你的 [like_service.go#L49-61](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/like_service.go#L49-L61)：

```go
// 尝试发MQ消息
mysqlEnqueued := false
redisEnqueued := false
if s.likeMQ != nil {
    if err := s.likeMQ.Like(ctx, like.AccountID, like.VideoID); err == nil {
        mysqlEnqueued = true  // ← 消息发送到 MQ 成功
    }
}
if s.popularityMQ != nil {
    if err := s.popularityMQ.Update(ctx, like.VideoID, 1); err == nil {
        redisEnqueued = true  // ← 消息发送到 MQ 成功
    }
}
```

**为什么这么做？** 有3个核心原因：

| 原因 | 解释 | 你项目的体现 |
|------|------|-------------|
| **🚀 加快响应速度** | 用户点赞后不用等数据库写完，直接返回成功 | `c.JSON(200, gin.H{"message": "like success"})` 立即返回 |
| **🔄 削峰填谷** | 高峰期请求太多，MQ帮忙排队慢慢处理 | 1000人同时点赞，数据库压力小很多 |
| **🔌 解耦** | API不需要知道后续怎么存，交给Worker处理 | API只管发消息，不关心谁处理 |

---

### 3. RabbitMQ 核心概念（你项目用的）

看 [rabbitMQ.go#L47-90](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/rabbitmq/rabbitMQ.go#L47-L90)：

```
┌─────────────────────────────────────────────────────────────┐
│                      RabbitMQ 工作原理                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Producer (生产者)     Exchange (交换机)    Queue (队列)   │
│   ────────────────▶    ───────────────▶  ──────────────    │
│   你的 API服务              路由分发            暂存消息     │
│                                                             │
│                              │                              │
│                              ▼                              │
│                       Consumer (消费者)                     │
│                       LikeWorker 处理消息                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**你项目中的配置**（[likeMQ.go#L10-20](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/rabbitmq/likeMQ.go#L10-L20)）：

```go
const (
    likeExchange   = "like.events"      // 交换机：接收API发来的消息
    likeQueue      = "like.events"      // 队列：暂时存放消息
    likeBindingKey = "like.*"          // 绑定键：交换机用这个决定发给哪个队列
    
    // 路由键（Routing Key）：决定消息类型的标识
    likeLikeRK   = "like.like"         // 点赞
    likeUnlikeRK = "like.unlike"       // 取消点赞
)
```

---

### 4. 消息是怎么发送的？（你项目代码解读）

看 [likeMQ.go#L36-50](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/rabbitmq/likeMQ.go#L36-L50)：

```go
func (l *LikeMQ) Like(ctx context.Context, userID, videoID uint) error {
    // 1. 构建消息体（LikeEvent）
    event := LikeEvent{
        EventID:    id,           // 唯一ID，用于追踪
        Action:     "like",       // 操作类型：点赞
        UserID:     userID,       // 用户ID
        VideoID:    videoID,      // 视频ID
        OccurredAt: time.Now(),   // 时间
    }
    // 2. 发送到交换机
    return l.PublishJSON(ctx, likeExchange, likeLikeRK, event)
}
```

**消息内容示例**：
```json
{
    "event_id": "a1b2c3d4e5f6",
    "action": "like",
    "user_id": 123,
    "video_id": 456,
    "occurred_at": "2024-01-15T10:30:00Z"
}
```

---

### 5. Worker（消费者）是怎么处理的？

看 [likeworker.go#L62-100](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/worker/likeworker.go#L62-L100)：

```go
func (w *LikeWorker) process(ctx context.Context, body []byte) error {
    // 1. 解析消息
    var evt rabbitmq.LikeEvent
    json.Unmarshal(body, &evt)
    
    // 2. 根据action处理
    switch evt.Action {
    case "like":
        return w.applyLike(ctx, evt.UserID, evt.VideoID)
    case "unlike":
        return w.applyUnlike(ctx, evt.UserID, evt.VideoID)
    }
}

// 实际处理：写数据库
func (w *LikeWorker) applyLike(ctx context.Context, userID, videoID uint) error {
    // 1. 写入 likes 表
    w.likes.LikeIgnoreDuplicate(ctx, &video.Like{...})
    
    // 2. 视频 likes_count +1
    w.videos.ChangeLikesCount(ctx, videoID, 1)
    
    // 3. 视频 popularity +1
    w.videos.ChangePopularity(ctx, videoID, 1)
    
    return nil
}
```

---

### 6. 完整流程图（面试必画！）

```
用户点击点赞
      │
      ▼
┌─────────────────────────────────────┐
│  API: LikeHandler.Like()            │
│  - 解析 JSON 获取 video_id          │
│  - 从 JWT 获取 account_id           │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Service: LikeService.Like()        │
│  - 检查视频是否存在                  │
│  - 检查是否已点赞                    │
└──────────────┬──────────────────────┘
               │
               ▼
        ┌──────┴──────┐
        │ 发送MQ消息  │
        └──────┬──────┘
               │
     ┌─────────┴─────────┐
     ▼                   ▼
┌─────────┐         ┌──────────┐
│ like.   │         │popularity│
│ events  │         │.events   │
│ (写MySQL)│         │(更新Redis)│
└────┬────┘         └────┬─────┘
     │                   │
     ▼                   ▼
┌─────────────────────────────────┐
│  Worker (后台异步处理)           │
│  - LikeWorker 消费 like.events  │
│  - 写入 likes 表                │
│  - likes_count + 1              │
│  - popularity + 1               │
└─────────────────────────────────┘
```

---

### 7. 降级机制（MQ挂了怎么办？）

看 [like_service.go#L61-85](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/like_service.go#L61-L85)：

```go
// 两个队列都成功，直接返回
if mysqlEnqueued && redisEnqueued {
    return nil
}

// ⚠️ MQ发送失败，降级直写！
if !mysqlEnqueued {
    // 直接写MySQL（同步等待）
    s.repo.db.Transaction(func(tx *gorm.DB) error {
        tx.Create(like)                                    // 写入 likes 表
        tx.Model(&Video{}).UpdateColumn("likes_count", ...)   // 点赞数+1
        tx.Model(&Video{}).UpdateColumn("popularity", ...)    // 热度+1
    })
}

if !redisEnqueued {
    // 直接更新Redis（同步等待）
    UpdatePopularityCache(ctx, s.cache, like.VideoID, 1)
}
```

**这就是面试亮点**：MQ挂了也能正常工作！

---

### 8. 总结：面试话术

> **面试官问：请说一下点赞的异步处理流程？**

```
我的处理流程是：

1. 【API层】接收点赞请求，解析出 video_id 和 account_id

2. 【Service层】先做前置检查（视频是否存在、是否已点赞）

3. 【发送MQ消息】尝试发送两条消息：
   - like.events → 异步写入MySQL的likes表
   - popularity.events → 异步更新Redis热榜

1. 【降级保证】如果MQ发送失败，立刻降级为同步直写，保证数据不丢失  高可用

2. 【Worker消费】后台Worker监听队列，异步：
   - 写入likes表
   - 视频likes_count + 1
   - 视频popularity + 1

【为什么这样设计？】
- 加快响应：用户点赞立即返回，不用等数据库
- 削峰填谷：高峰期消息堆积，数据库压力小
- 高可用：MQ挂了也能工作（降级直写）
- 解耦：写入和计数分离
```
