好！今天进入 **Day 2：JWT鉴权 + 缓存自愈机制**

这是面试必问的内容，必须完全搞懂！

---

## 1. 先看整体鉴权流程

        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/auth/jwt.go
          


        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/middleware/jwt/jwt.go
          
现在看登录时Token如何存储：

        
            
toolName: search_by_regex
            
status: success
          
query: func.*Login
          


        
            
toolName: view_files
            
status: success
          
            
filePath: /home/ihoo/Mypoject/feedsystem_video_go/backend/internal/account/service.go
          
好的！我继续为你深入讲解 **JWT鉴权 + 缓存自愈机制**，这部分是面试必问的重点！

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

### 2.5 常见面试追问

| 追问 | 回答要点 |
|------|----------|
| Token存在Redis被劫持怎么办？ | Token存MySQL做最终验证，Redis只是缓存加速 |
| 为什么要两份存储？ | Redis快但可能丢数据，MySQL慢但可靠 |
| Token过期了怎么办？ | 前端引导重新登录 |
| 修改密码后Token还有效吗？ | 会清空Token，强制下线 |
| Redis和MySQL如何保证一致性？ | 先写MySQL，再删Redis缓存 |

---

### ✅ 今日作业

1. **能画出登录 + 验证的完整流程图**
2. **能说出"缓存自愈"是什么意思**
3. **能解释JWTAuth和SoftJWTAuth的区别**

---

### 📅 明天预告：Day 3 - 点赞流程 + MQ异步处理

这是项目最核心的部分，也是面试最加分的亮点！

准备好了喊我，我们继续！ 🚀