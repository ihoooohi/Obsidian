## 一、什么是http

http是一个 “客户端（client）发起请求，服务端（server）回响应“ 的模型

## 二、一次http请求包含什么
### 1. Method（方法）

GET、POST、PUT、PATCH、DELETE

### 2.URL（定位资源）

```
scheme://host:port/path?query#fragment
```

### 3.Header（元数据，重点）

Header = **控制和描述请求的元信息**

#### 通用

- `Host`（HTTP/1.1 必须）
    
- `User-Agent`
    
- `Accept`
    
- `Content-Type`
    
- `Content-Length

👉 **Header 决定行为，比 Body 还重要**

### 4.Body（数据本体）

- GET：通常没有
    
- POST / PUT / PATCH：常有

## 一次http响应有什么

### 1.Status Code（状态码）

400 / 401 / 403 / 404

### 2.Response Header

和请求一样重要：

- `Content-Type`
    
- `Content-Length`
    
- `Set-Cookie`
    
- `Location`
    
- `Cache-Control`
    

⚠️ 面试常问：

- Cookie 是通过什么传的？  
    👉 **Header，不是 Body**
### 3. Response Body

- 真正的数据
    
- JSON / HTML / 文件流
    
- 是否有 Body 由 **Status Code 决定**
    
    - 204 / 304：**没有 Body**