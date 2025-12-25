
---

### 1. 路由方法（定义“句子”的谓语）

Gin 支持所有的标准 HTTP 动作，这些方法决定了接口要做什么。

```go
r := gin.Default() // 创建并返回一个“已经配置好常用功能的 Gin 路由引擎（HTTP 服务器核心对象）”

r.GET("/user", handleGet)      // 查询
r.POST("/user", handlePost)    // 新增
r.PUT("/user", handlePut)      // 全量更新
r.PATCH("/user", handlePatch)  // 部分更新
r.DELETE("/user", handleDelete)// 删除

// 路由组：把相同前缀的路径分在一起，方便管理
v1 := r.Group("/api/v1")
{
    v1.GET("/login", login)
    v1.GET("/submit", submit)
}
```

---

### 2. 获取参数（解析用户的请求）

用户传递数据的方式通常有三种：**路径参数、查询参数、表单/JSON 参数**。

#### **① 路径参数 (Param)**

对应路径：`/user/123`

```go
r.GET("/user/:id", func(c *gin.Context) {
    id := c.Param("id") // 获取路径中的 123
})
```

#### **② 查询参数 (Query)**

对应路径：`/search?name=jack&age=20`

```go
r.GET("/search", func(c *gin.Context) {
    name := c.Query("name")              // 获取 jack
    age := c.DefaultQuery("age", "18")   // 获取参数，若没有则默认为 18
})
```

#### **③ Post 参数 (PostForm)**

对应 HTML 表单提交的数据：

```go
r.POST("/submit", func(c *gin.Context) {
    message := c.PostForm("message")
})
```

---

### 3. 数据绑定 (Binding) —— **最核心的功能**

这是 Gin 最强大的地方。它利用你学过的 **结构体（Struct）** 和 **反射**，自动把请求内容塞进变量里，并进行校验。

```go
type LoginRequest struct {
    // binding:"required" 表示这个字段不能为空
    User     string `json:"user" binding:"required"`
    Password string `json:"password" binding:"required"`
}

r.POST("/login", func(c *gin.Context) {
    var val LoginRequest
    // ShouldBindJSON 会自动判断：1.这是不是JSON？ 2.字段对不对？
    if err := c.ShouldBindJSON(&val); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // 此时 val 已经被填满了数据
    fmt.Println(val.User, val.Password)
})
```

---

### 4. 响应请求 (Response)

告诉客户端结果是什么。


```go
// 1. 返回 JSON（最常用，gin.H 是 map[string]any 的简写）
c.JSON(200, gin.H{"message": "success"})

// 2. 返回 HTML 页面
c.HTML(200, "index.tmpl", gin.H{"title": "首页"})

// 3. 返回纯文本
c.String(200, "Hello World")

// 4. 重定向
c.Redirect(http.StatusMovedPermanently, "https://google.com")
```

---

### 5. 中间件 (Middleware) —— “安检口”

中间件是在处理业务逻辑**之前**或**之后**执行的代码，比如日志记录、身份验证。


```go
// 定义一个中间件
func MyMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        fmt.Println("开始安检...")
        c.Next() // 放行，去执行后面的 Handler
        fmt.Println("业务做完了，打扫战场...")
    }
}

func main() {
    r := gin.Default()
    r.Use(MyMiddleware()) // 全局注册：所有接口都要过一遍安检
}
```

---

### 6. 实战建议：如何组织你的代码？

结合你之前的 **依赖注入** 代码，一个典型的 Gin Handler 使用方式是这样的：


```go
// Handler 结构体
type UserHandler struct {
    service *Service // 依赖注入过来的 Service
}

// Handler 的方法
func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")
    
    // 调用 Service 层处理逻辑
    user, err := h.service.FindUser(id)
    
    if err != nil {
        c.JSON(404, gin.H{"msg": "找不到用户"})
        return
    }
    
    // 返回结果
    c.JSON(200, user)
}
```

### 总结

1. **`c.Param` / `c.Query`**：零散拿数据。
    
2. **`c.ShouldBindJSON`**：批量拿数据（最推荐）。
    
3. **`c.JSON`**：返回数据。
    
4. **`r.Group`**：分类接口。
    

## 八、绝对不能混用（新手常见误区）

❌ 错误理解：

> “能不能在 RouterGroup 里拿参数？”

❌ 不行

`v1.Param("id") // ❌ 根本不存在`

---

❌ 错误理解：

> “能不能在 Context 里注册路由？”

❌ 不行

`c.GET(...) // ❌`

---

## 九、核心关系图（脑内模型）

```
Engine
 └── RouterGroup   （定义规则）
       └── 路由 → HandlerFunc
                         ↑
                    gin.Context（请求时传入）

```