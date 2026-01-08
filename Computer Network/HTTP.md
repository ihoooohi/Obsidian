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

## 三、URL是什么

url就是一个定位资源的**地址**

