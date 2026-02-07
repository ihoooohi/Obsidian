
## Gin 框架与 Web 开发篇

中间件就类似于保安

请求--中间件--处理函数（控制器/controller/handler）--响应

### 常见鉴权方式速览

| 方式      | 用途     |
| ------- | ------ |
| Session | 传统 Web |
| Token   | 前后端分离  |
| JWT     | 无状态鉴权  |
| OAuth2  | 第三方登录  |
| API Key | 系统对系统  |
### JWT(Json Web Token)

jwt通常在每个请求的Header里
#### 一、一句话先记住（先别纠结细节）

> **JWT 鉴权 = 用一段“服务器签过名的字符串”来证明你是谁**

它解决的是一句话：

> **“我怎么知道你已经登录过，而且这次请求还是你？”**

---

#### 二、为什么需要 JWT？（没有 JWT 会怎样）

##### 先看「最原始的登录」

###### 1️⃣ 你登录一次

```
用户名 + 密码 → 登录接口
```

###### 2️⃣ 登录成功后

问题来了：

❓ 下一次请求：

```
GET /profile
```

服务器怎么知道：

- 你是谁？
    
- 你登录过？
    

---

 最蠢的办法（不能用）

👉 **每次请求都传用户名+密码**

❌ 不安全  
❌ 太麻烦

---

#### 三、JWT 出现前：Session（简单带过）

传统方案是：

- 登录成功 → 服务器存一份 Session
    
- 浏览器存一个 SessionID
    
- 每次请求带 SessionID
    

问题：

- 服务器要存状态
    
- 分布式、微服务很麻烦
    

👉 **JWT 就是为了解决这个问题的**

---

#### 四、JWT 到底是啥？（别被名字吓到）

 JWT 全名

**JSON Web Token**

你可以把它理解成：

> **一张“服务器盖了章的身份证复印件”**

---

 JWT 是一串字符串，看起来像这样：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJ1c2VySWQiOjEwMDEsInVzZXJOYW1lIjoiemhhbmdzYW4ifQ
.abc123xxx
```

三段，用 `.` 分开：

```
头.内容.签名
```

---

#### 五、JWT 三部分用“人话”讲

 1️⃣ Header（头）

> “我用什么算法签的”

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

---

 2️⃣ Payload（内容，最重要）

> “这个人是谁”

```json
{
  "userId": 1001,
  "userName": "zhangsan",
  "exp": 1710000000
}
```

⚠️ 注意：

- **不是加密的**
    
- 只是 Base64 编码
    
- 不要放密码！！！
    

---

 3️⃣ Signature（签名）

> “服务器用密钥盖的章”

```
HMACSHA256(
  base64(header) + "." + base64(payload),
  secretKey
)
```

👉 **防伪用的**

---

#### 六、JWT 鉴权完整流程（重点）

我们一步一步来。

---

 ① 登录阶段（只发生一次）

```
POST /login
用户名 + 密码
```

服务器：

- 校验账号密码
    
- 生成 JWT
    
- 返回给客户端
    

```json
{
  "token": "xxxxx.yyyyy.zzzzz"
}
```

客户端：

- 保存 JWT（浏览器 / App）
    

---

 ② 访问接口阶段（重点）

客户端每次请求都带上：

```
Authorization: Bearer xxxxx.yyyyy.zzzzz
```

---

 ③ 服务端鉴权（中间件里做）

服务器收到请求后：

1. 从 Header 里拿 Token
    
2. 校验签名（有没有被篡改）
    
3. 校验是否过期
    
4. 解析出用户信息
    
5. 放进 `gin.Context`
    

```go
c.Set("userId", 1001)
c.Set("userName", "zhangsan")
```

---

 ④ 业务 Handler 直接用

```go
userId, _ := c.Get("userId")
```

---

#### 七、JWT 鉴权 + gin.Context = 天作之合

现在你应该能看懂这一句了：

```go
c.Set("userName", "zhangsan")
```

它的来源是：

```
JWT → 解析 → Context
```

---

#### 八、JWT 和 Session 的本质区别（不背也能懂）

|对比|JWT|Session|
|---|---|---|
|状态|无状态|有状态|
|存储|客户端|服务端|
|扩展性|好|一般|
|服务器压力|小|大|

---

#### 九、你现在真正要记住的 3 句话

你可以直接抄走👇

1️⃣ **JWT 是登录成功后的“身份证”**  
2️⃣ **每个请求都要带 JWT，服务器不存登录状态**  
3️⃣ **JWT 的信息最终会被放进 gin.Context 给业务用**

---


### Gin框架

#### gin.context

**同一次请求，用同一个 Context**

 1️⃣ `context` 里有什么？

`gin.Context` 基本涵盖了一次请求的**所有东西**：

- **请求信息**
    
    - `c.Request`（`*http.Request`）
        
    - URL、Query、Path 参数
        
    - Header、Cookie
        
    - Body（JSON / Form / XML）
        
- **响应相关**
    
    - `c.Writer`
        
    - 状态码
        
    - 返回的数据（JSON / String / File）
        
- **中间件之间共享的数据**
    
    - `c.Set(key, value)`
        
    - `c.Get(key)`
        
- **控制请求流程**
    
    - `c.Next()` → 继续执行后面的中间件 / handler
        
    - `c.Abort()` → 直接中断后续处理
        
    - `c.AbortWithStatusJSON(...)`