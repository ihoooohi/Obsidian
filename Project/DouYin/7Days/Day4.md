好！进入 **Day 4：Feed流 + 分页策略** —— 这是面试最体现技术深度的部分！

---

## 📱 Day 4：Feed流 + 分页策略

### 4.1 4种Feed流一览

| 接口 | 排序方式 | 分页方式 | 登录要求 | 缓存 |
|------|----------|----------|----------|------|
| `/feed/listLatest` | 时间倒序 | 时间游标 | SoftJWTAuth | ✅ |
| `/feed/listLikesCount` | 点赞数倒序 | 复合游标 | SoftJWTAuth | ❌ |
| `/feed/listByPopularity` | 热度排序 | 热榜快照 | SoftJWTAuth | ✅ |
| `/feed/listByFollowing` | 关注关系 | 关注列表聚合 | JWTAuth | ❌ |

---

### 4.2 Feed流整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Feed流服务                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ listLatest │  │listLikesCount│ │listByPopularity          │
│  │ (最新)      │  │ (点赞排序)   │  │ (热榜)        │            │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│    ┌────┴────┐     ┌─────┴─────┐    ┌─────┴─────┐              │
│    │ Redis   │     │   MySQL   │    │  Redis    │              │
│    │ 缓存    │     │  查询     │    │  ZSET热榜 │              │
│    └─────────┘     └───────────┘    └───────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 4.3 核心代码：最新Feed流（ListLatest）

        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/feed/service.go
          


        
            
toolName: search_by_regex
            
status: success
          
query: func.*ListByPopularity
          


        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/feed/service.go
          


        
            
toolName: search_by_regex
            
status: success
          
query: hot:video
          


        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/popularity_cache.go
          
完美！现在我有了完整信息，整理 **Day 4** 内容：

---

## 📱 Day 4：Feed流 + 分页策略

### 4.1 4种Feed流一览

| 接口 | 排序方式 | 分页方式 | 登录要求 | 缓存 |
|------|----------|----------|----------|------|
| `/feed/listLatest` | 时间倒序 | 时间游标 | SoftJWTAuth | ✅ |
| `/feed/listLikesCount` | 点赞数倒序 | 复合游标 | SoftJWTAuth | ❌ |
| `/feed/listByPopularity` | 热度排序 | 热榜快照 | SoftJWTAuth | ✅ |
| `/feed/listByFollowing` | 关注关系 | 关注列表聚合 | JWTAuth | ❌ |

---

### 4.2 Feed 1：最新Feed流（ListLatest）

**特点**：按时间倒序，支持时间游标分页，匿名用户也能看

**流程图**：
```
请求 /feed/listLatest
       │
       ▼
┌────────────────────────────────────────────┐
│  1. 构建缓存Key                             │
│  feed:listLatest:limit=10:before=1706xxx   │
└──────────────────┬─────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
   命中缓存？             未命中缓存
        │                     │
        ▼                     ▼
   直接返回            ┌─────┴─────┐
   (5秒TTL)           │  加互斥锁 │ (SETNX)
                      └─────┬─────┘
                            ▼
                     查MySQL → 返回 → 写入缓存
```

**核心代码** ([feed/service.go#L28-95](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/feed/service.go#L28-L95))：

```go
func (f *FeedService) ListLatest(ctx context.Context, limit int, latestBefore time.Time, viewerAccountID uint) {
    // 1. 构建缓存Key
    cacheKey := fmt.Sprintf("feed:listLatest:limit=%d:before=%d", limit, before)
    
    // 2. 先查缓存
    if cache != nil {
        b, err := cache.GetBytes(ctx, cacheKey)
        if err == nil {
            // 命中！直接返回
            return json.Unmarshal(b, &cached)
        }
    }
    
    // 3. 未命中，加互斥锁（防击穿）
    token, locked, _ := cache.Lock(ctx, "lock:"+cacheKey, 500*time.Millisecond)
    if locked {
        defer cache.Unlock(ctx, "lock:"+cacheKey, token)
        // 查MySQL
        videos, _ := f.repo.ListLatest(ctx, limit, latestBefore)
        // 写入缓存（TTL 5秒）
        cache.SetBytes(ctx, cacheKey, data, 5*time.Second)
        return data
    }
}
```

**面试话术**：
```
"最新Feed使用时间游标分页：
- 用'最后一条视频的创建时间'作为游标
- 下次请求带上这个时间，查更早的视频
- 只对匿名用户（viewerAccountID=0）缓存，5秒TTL
- 实现了'防击穿'：缓存未命中时用SETNX加锁，只一个请求回源"
```

---

### 4.3 Feed 2：热榜Feed流（ListByPopularity）- 重点！

**特点**：滑动窗口热榜 + 快照分页，保证分页稳定性

**核心设计**：
```
┌─────────────────────────────────────────────────────────────────┐
│  热度写入：按分钟分桶                                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│  │ 1m窗口  │ │ 1m窗口  │ │ 1m窗口  │ │ 1m窗口  │  ...         │
│  │ 12:00   │ │ 12:01   │ │ 12:02   │ │ 12:03   │               │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘               │
│       │           │           │           │                      │
│       └───────────┴───────────┴───────────┘                     │
│                     ↓ ZUNIONSTORE                               │
│              合并成"热榜快照"                                     │
│              (as_of: 快照生成时间)                                │
└─────────────────────────────────────────────────────────────────┘
```

**热度更新** ([popularity_cache.go](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/video/popularity_cache.go))：

```go
func UpdatePopularityCache(ctx context.Context, cache *rediscache.Client, id uint, change int64) {
    // 按分钟分桶：hot:video:1m:202601021200
    now := time.Now().UTC().Truncate(time.Minute)
    windowKey := "hot:video:1m:" + now.Format("200601021504")
    
    // ZINCRBY 增加热度
    cache.ZincrBy(ctx, windowKey, videoID, float64(change))
}
```

**热榜查询** ([feed/service.go#L233-290](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/feed/service.go#L233-L290))：

```go
func (f *FeedService) ListByPopularity(...) {
    // 1. 生成快照版本号
    asOf := time.Now().UTC().Truncate(time.Minute)  // 当前分钟
    if reqAsOf > 0 {
        asOf = time.Unix(reqAsOf, 0).UTC().Truncate(time.Minute)
    }
    
    // 2. 收集最近60分钟的桶
    keys := []string{}
    for i := 0; i < 60; i++ {
        keys = append(keys, "hot:video:1m:"+asOf.Add(-i*time.Minute).Format("200601021504"))
    }
    
    // 3. 合并成快照（ZUNIONSTORE）
    dest := "hot:video:merge:1m:" + asOf.Format("200601021504")
    cache.ZUnionStore(ctx, dest, keys, "SUM")  // 聚合60分钟的热度
    
    // 4. 快照分页（ZREVRANGE）
    start := offset
    stop := start + limit - 1
    members := cache.ZRevRange(ctx, dest, start, stop)
    
    // 5. 根据video_id查MySQL获取详情
    videos := f.repo.GetByIDs(ctx, members)
    return videos
}
```

---

### 4.4 🎯 面试话术

**面试官问：热榜如何实现稳定分页？**

```
"热榜我使用了'滑动窗口+快照分页'方案：

【写入】
- 按分钟分桶：hot:video:1m:202601011200
- 每次点赞/评论，用ZINCRBY更新对应分钟桶
- 好处：减少单个Key的写竞争

【查询】
- 首次请求生成快照：ZUNIONSTORE合并最近60个桶
- 快照有个版本号as_of（分钟级时间戳）
- 后续分页带上相同的as_of

【为什么这么做】
- 实时热榜不稳定：上一页第10条，下一页可能变成第5条
- 快照分页：同一as_of内分页稳定，不会跳页/重复
- 2分钟快照过期，保证数据不太旧"
```

---

### 4.5 Feed 3：点赞数排序（ListLikesCount）

**复合游标分页**：
```go
// SQL查询
WHERE likes_count < ? OR (likes_count = ? AND id < ?)
ORDER BY likes_count DESC, id DESC
```

**面试话术**：
```
"点赞数排序时，可能100个视频都是10个赞，
单纯按点赞数分页会重复。

我的方案：'点赞数+ID'复合游标
WHERE likes_count < 10 OR (likes_count = 10 AND id < 100)

保证下一页的第一条 < 上一页的最后一条
实现：不重复、不遗漏"
```

---

### 4.6 常见面试追问

| 追问 | 回答要点 |
|------|----------|
| 分页时数据被删除怎么办？ | 快照分页保证单次分页稳定，全局有HasMore兜底 |
| 热榜数据多久更新一次？ | 实时更新（每次点赞ZINCRBY），快照每分钟生成 |
| 为什么热榜用Redis不用MySQL？ | ZSET操作原子，高性能，支持排序 |
| Feed缓存的TTL为什么这么短？ | 5秒，保证新视频能及时展示 |

---

### ✅ 今日作业

1. **能画出热榜滑动窗口+快照分页的流程**
2. **能解释复合游标分页的SQL怎么写**
3. **能说出"防击穿"是什么意思**

# Feed流基础知识补充

让我用通俗的方式帮你补充这些前置知识：

## 1. 什么是Feed流？

**Feed流** = "动态信息流"，你刷的短视频、朋友圈、微博首页都是Feed流。

```
┌─────────────────────────────────────┐
│           手机屏幕                   │
├─────────────────────────────────────┤
│  视频1  ← 最新发布的                  │
│  视频2  ← 稍早一点                    │
│  视频3  ← 再早一点                    │
│  ...                                │
└─────────────────────────────────────┘
     ↑ 不断往下刷，内容源源不断"流"出来
```

**本质**：一个**按某种规则排序的内容列表**，用户不断往下翻页。

---

## 2. 你提到的4种Feed流类型

| 类型 | 含义 | 举例 |
|------|------|------|
| **ListLatest** | 按发布时间排序 | 抖音"最新" tab |
| **ListByPopularity** | 按热度排序 | 微博热搜、抖音热榜 |
| **ListLikesCount** | 按点赞数排序 | 按点赞数倒序 |
| **ListByFollowing** | 关注的人发的内容 | 朋友圈 |

---

## 3. 分页方式（你不懂的核心概念）

### 传统分页 vs 游标分页

**传统分页**（Page 1, Page 2...）：
```sql
SELECT * FROM videos ORDER BY created_at DESC LIMIT 10 OFFSET 20
```
- 问题：页码越大，数据库扫描越多（OFFSET 10000 要扫1万行）

**游标分页**（Cursor-based）：
```sql
-- 记住上一页最后一条的时间/ID
SELECT * FROM videos 
WHERE created_at < '2024-01-01 12:00:00'  -- 游标
ORDER BY created_at DESC 
LIMIT 10
```
- 优势：不管翻到第几页，性能一样快

---

## 4. 缓存相关术语

| 术语 | 含义 |
|------|------|
| **TTL (Time To Live)** | 缓存存活时间，如5秒后过期 |
| **缓存击穿** | 热点key过期瞬间，大量请求打穿到数据库 |
| **互斥锁/SETNX** | 防止缓存击穿的常见方案：只有一个请求去查数据库 |
| **缓存命中率** | 命中缓存的比例，越高越好 |

---

## 5. 你提到的Redis操作

```
ZINCRBY  - 有序集合中增加分数（用于热度排序）
ZREVRANGE - 按分数倒序取值（获取热榜前N个）
ZUNIONSTORE - 合并多个有序集合（聚合60分钟的热度桶）
```

---

## 6. 你需要理解的项目架构

```
用户请求 /feed/listLatest
        │
        ▼
   API层 (handler)
        │
        ▼
   Service层 (业务逻辑：查缓存/加锁)
        │
        ▼
   Repository层 (操作数据库)
        │
        ▼
   MySQL / Redis
```

---

需要我更详细解释哪个部分吗？比如：
- 游标分页的具体实现？
- 热度计算公式？
- 缓存如何防击穿？
