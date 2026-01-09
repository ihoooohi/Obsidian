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

`https://www.example.com:443/path/page.html?key=value#section
`

各部分含义是：
1. **协议（Scheme）**  
    `https`：表示使用什么方式访问  
    常见的有：
    
    - `http`
        
    - `https`
        
    - `ftp`
        
    - `file`
        
2. **域名（Host）**  
    `www.example.com`：服务器的地址
    
3. **端口（Port，可选）**  
    `443`：服务器开放的端口
    
    - http 默认是 80
        
    - https 默认是 443
        
4. **路径（Path）**  
    `/path/page.html`：服务器上的具体资源路径
    
5. **查询参数（Query，可选）**  
    `?key=value`：向服务器传递的参数
    
6. **锚点（Fragment，可选）**  
    `#section`：页面内的位置
    

### 一句话总结

👉 **URL 就是告诉浏览器“用什么方式，到哪里，去拿什么东西”**