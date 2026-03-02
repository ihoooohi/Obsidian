好！今天进入 **Day 2：JWT鉴权 + 缓存自愈机制**

---

## 🔐 Day 2：JWT鉴权 + 缓存自愈机制

### 2.1 整体流程图（必须能画出来）

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户登录                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              1. 验证用户名密码 (bcrypt)                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              2. 生成JWT Token                                    │
│              (AccountID + Username + 24h过期)                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            ▼                              ▼
     ┌─────────────┐                ┌─────────────┐
     │   MySQL    │                │   Redis    │
     │ account表  │                │account:{id}│
     │ Token字段  │                │  TTL:24h   │
     └─────────────┘                └─────────────┘
```

---

### 2.2 核心代码解读

**代码1：登录时生成Token** ([service.go#L112-140](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/account/service.go#L112-L140))

```go
func (as *AccountService) Login(ctx context.Context, username, password string) (string, error) {
    // 1. 验证用户名密码
    account, err := as.FindByUsername(ctx, username)
    if err != nil {
        return "", err
    }
    if err := bcrypt.CompareHashAndPassword([]byte(account.Password), []byte(password)); err != nil {
        return "", err
    }
    
    // 2. 生成JWT Token
    token, err := auth.GenerateToken(account.ID, account.Username)
    
    // 3. 存MySQL + Redis
    if err := as.accountRepository.Login(ctx, account.ID, token); err != nil { // MySQL
        return "", err
    }
    if as.cache != nil {
        as.cache.SetBytes(..., fmt.Sprintf("account:%d", account.ID), token, 24*time.Hour) // Redis
    }
    return token, nil
}
```

**代码2：JWT验证中间件** ([middleware/jwt/jwt.go#L63-100](file:///home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/jwt/jwt.go#L63-L100))

```go
func check(...) {
    key := fmt.Sprintf("account:%d", claims.AccountID)

    // 步骤1：先查 Redis（高性能）
    if cache != nil {
        b, err := cache.GetBytes(ctx, key)
        if err == nil {
            // Redis命中！直接验证Token
            if string(b) != tokenString {
                return error("token已失效")
            }
            c.Set("accountID", claims.AccountID)
            c.Next()
            return
        }
    }

    // 步骤2：Redis没有/失败，回退查 MySQL
    accountInfo, err := accountRepo.FindByID(ctx, claims.AccountID)
    if accountInfo.Token != tokenString {
        return error("token已失效")
    }

    // 步骤3：自愈！回填Redis
    if cache != nil {
        cache.SetBytes(ctx, key, tokenString, 24*time.Hour)
    }
    c.Next()
}
```

---

### 2.3 两种鉴权模式（必须分清）

| 模式 | 使用场景 | 区别 |
|------|----------|------|
| `JWTAuth` | 点赞、评论、关注 | **必须**带Token，否则401 |
| `SoftJWTAuth` | Feed流 | 可选带Token，带了则验证，不带也能访问 |

```go
// Feed流路由 - SoftJWTAuth
feedGroup.Use(jwt.SoftJWTAuth(...))  // 可选登录
{
    feedGroup.POST("/listLatest", ...)
}

// 点赞路由 - JWTAuth  
protectedLikeGroup.Use(jwt.JWTAuth(...))  // 必须登录
{
    protectedLikeGroup.POST("/like", ...)
}
```

---

### 2.4 🎯 面试话术（背下来！）

**面试官问：请解释一下你的JWT鉴权流程？**

```
"我的项目使用JWT Token鉴权，并实现了'缓存自愈机制'：

【登录流程】
1. 用户登录 → 验证用户名密码（bcrypt）
2. 生成JWT Token（包含AccountID、Username、24小时过期）
3. Token存两份：
   - MySQL的account表（持久化，保证不丢）
   - Redis的account:{id}（高性能，TTL 24小时）

【验证流程】
1. 从Header取Token，解析JWT（验证签名和过期）
2. 先查Redis（key=account:{id}）
   - 命中：直接比对Token，一致则通过
   - 未命中/失败：回退到MySQL查询
3. 如果从MySQL验证通过，回填Redis（自愈）

【两种鉴权】
- JWTAuth：强制登录（点赞、关注）
- SoftJWTAuth：可选登录（刷Feed，匿名也能看）

【设计好处】
- 高性能：大部分请求走Redis
- 高可用：Redis挂了不影响登录（回退MySQL）
- 自愈：Redis恢复后自动回填，不用人工干预"
```

---





## 📌 核心代码架构

### 1. 两种鉴权模式（见 [jwt.go](file:///Users/ihoo/GolandProjects/feedsystem_video_go/backend/internal/middleware/jwt/jwt.go)）

| 模式 | 用途 | 区别 |
|------|------|------|
| **JWTAuth** | 强制登录（点赞、评论等） | 无Token直接返回401 |
| **SoftJWTAuth** | 可选登录（刷Feed流） | 无Token也放行，继续处理 |

---

### 2. 缓存自愈机制流程图

```
请求进来 ──▶ 解析 JWT Token ──▶ 提取 accountID
                     │
                     ▼
              先查 Redis (account:{id})
                     │
           ┌────────┴────────┐
           │                 │
        命中                未命中/失败
           │                 │
           ▼                 ▼
     比较 Token 是否一致   查 MySQL 兜底
           │                 │
           │           ┌────┴────┐
           │           │         │
           ▼           ▼         ▼
        验证通过    查到Token   未找到/不匹配
           │           │         │
           │           ▼         │
           │     回填 Redis      │
           │     (自愈机制)      │
           │           │         │
           └───────────┴────────┘
                     │
                     ▼
              设置 context.accountID
                     │
                     ▼
               放行进入业务
```

---

### 3. 关键代码实现

**登录时双写**（[service.go#L104-117](file:///Users/ihoo/GolandProjects/feedsystem_video_go/backend/internal/account/service.go#L104-L117)）：

```go
func (as *AccountService) Login(ctx context.Context, username, password string) (string, error) {
    // 1. 验证密码
    // 2. 生成 JWT Token
    token, _ := auth.GenerateToken(account.ID, account.Username)
    
    // 3. 存 MySQL（持久化）
    as.accountRepository.Login(ctx, account.ID, token)
    
    // 4. 存 Redis（高性能）
    as.cache.SetBytes(cacheCtx, fmt.Sprintf("account:%d", account.ID), []byte(token), 24*time.Hour)
    
    return token, nil
}
```

**验证时的自愈逻辑**（[jwt.go#L79-117](file:///Users/ihoo/GolandProjects/feedsystem_video_go/backend/internal/middleware/jwt/jwt.go#L79-L117)）：

```go
func check(c *gin.Context, claims *auth.Claims, tokenString string, ...) {
    key := fmt.Sprintf("account:%d", claims.AccountID)
    
    // 先查 Redis
    b, err := cache.GetBytes(cacheCtx, key)
    if err == nil {
        // 命中，直接用
        if string(b) != tokenString {
            c.AbortWithStatusJSON(http.StatusUnauthorized, ...)
            return
        }
        c.Set("accountID", claims.AccountID)
        c.Next()
        return
    }
    
    // Redis 没有/失败 → 查 MySQL 兜底
    accountInfo, err := accountRepo.FindByID(c.Request.Context(), claims.AccountID)
    if accountInfo.Token != tokenString {
        c.AbortWithStatusJSON(http.StatusUnauthorized, ...)
        return
    }
    
    // 查到后回填 Redis（自愈）
    cache.SetBytes(cacheCtx, key, []byte(tokenString), 24*time.Hour)
    
    c.Set("accountID", claims.AccountID)
    c.Next()
}
```

---

## 🎯 面试话术（直接背）

面试官让你介绍项目鉴权时，直接说：

> "我的项目使用 JWT Token 鉴权，实现了**缓存自愈机制**：
>
> **1. 登录时双写 Token**：
> - MySQL 持久化存储（用于兜底）
> - Redis 高性能缓存（key 为 `account:{id}`，24小时过期）
>
> **2. 验证时优先读 Redis**：
> - 先查 Redis，命中则直接使用
> - Redis 未命中/故障 → 回退查 MySQL
> - MySQL 查到后**回填 Redis**，实现自愈
>
> **3. 两种鉴权模式**：
> - `JWTAuth`：强制登录场景（点赞、评论、发社交动态）
> - `SoftJWTAuth`：可选登录（浏览 Feed 流），无 Token 也能继续
>
> **4. 这样做的好处**：
> - 99% 请求走 Redis，性能高
> - Redis 故障不影响服务，自动降级 MySQL
> - 每次验证都会自愈缓存，Redis 里的数据始终是最新的"

---

## 💡 面试官可能追问的问题

| 问题 | 回答要点 |
|------|----------|
| **Token 过期了怎么办？** | JWT 本身有过期时间(24h)，前端收到401后引导用户重新登录 |
| **如何处理 Token 失效/注销？** | 登出时删除 Redis 中的 Token，MySQL 中 Token 设为空，下次验证就会失败 |
| **为什么不用 Redis 存用户信息？** | 只存 Token 用于验证状态，用户信息太大且不需要每次都完整获取 |
| **Redis 和 MySQL 数据一致性？** | 登录/登出/改名时同时写两边，验证时以 MySQL 为准并回填 Redis |
| **如何防止 Token 盗用？** | Token 存在 MySQL，每次验证都比对；登出立即删 Redis |

| 追问                  | 回答要点                         |
| ------------------- | ---------------------------- |
| Token存在Redis被劫持怎么办？ | Token存MySQL做最终验证，Redis只是缓存加速 |
| 为什么要两份存储？           | Redis快但可能丢数据，MySQL慢但可靠       |
| Token过期了怎么办？        | 前端引导重新登录                     |
| 修改密码后Token还有效吗？     | 会清空Token，强制下线                |
| Redis和MySQL如何保证一致性？ | 先写MySQL，再删Redis缓存            |
