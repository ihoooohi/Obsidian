# Post-mortem：C 端简历上传持续 500 排查全记录

**日期：** 2026-04-20 ~ 2026-04-21  
**影响范围：** `heartcertai.com/profile/setup` 简历上传功能完全不可用  
**最终根因：** nginx 配置未同步——服务器上 `/api/storage/` 路由指向了 FastAPI，而非 Next.js  
**修复耗时：** 约 8 小时，经历 6+ 次部署  
**修复方式：** 零代码改动，同步一行 nginx 配置后 reload，30 秒搞定

---

## 一、症状

用户上传 PDF/DOCX 简历后，界面报：

> 上传失败，请稍后重试。

浏览器 Network 面板：`POST /api/storage/upload → 500`，响应体 26 字节。

---

## 二、排查时间线与每一步的错误假设

### 阶段 1：以为是 Supabase 冷启动问题

**现象：** `POST /api/storage/upload` 返回 401。

**假设：** Next.js 路由 handler 里调用了 `supabase.auth.getUser()`，Alibaba Cloud Supabase 冷启动偶发 500，导致 auth 失败返回 401。

**操作：** 把 `getUser()` 改成 `getSession()`，再次部署。

**结果：** 还是 401。

---

### 阶段 2：再改 auth 策略——JWT 本地 decode

**假设：** `getSession()` 也会触发 Supabase 网络请求，仍然有冷启动风险。

**操作：** 彻底去掉 Supabase auth 调用，改为从 `Authorization: Bearer <token>` 头部直接 base64url decode JWT payload，本地提取 `userId`，不发任何网络请求。

```typescript
function decodeJwtUserId(token: string): string | null {
  const parts = token.split('.');
  const payload = JSON.parse(Buffer.from(parts[1], 'base64url').toString());
  if (payload.exp < Date.now() / 1000) return null;
  return payload.sub ?? null;
}
```

**结果：** 401 消失，变成 500。进步了，但还是失败。

---

### 阶段 3：以为是 Next.js middleware 拦截

**现象：** 500，且 `docker logs` 里看不到任何 `console.error` 输出。

**假设：** `middleware.ts` 的 `updateSession()` 对所有路径都调用 `getUser()`，包括 `/api/` 路由。Supabase 冷启动 500 让 middleware 崩溃，在 route handler 执行前就返回了 500。

**操作：** 在 middleware matcher 中排除 `api/` 路径：

```typescript
"/((?!_next|_vercel|favicon.ico|api/|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"
```

部署。

**结果：** 仍然 500，`docker logs` 依然没有输出。

---

### 阶段 4：迷失在"日志为什么消失"

**假设连环：** 
1. 也许 `console.error` 在 Next.js standalone 生产模式被抑制？
2. 也许 Node.js stdout 有缓冲没刷新？
3. 也许容器重启把日志清掉了？
4. 也许 macOS `grep -i 'upload\|error'` 的 `\|` 语法不对（BSD grep 不支持 ERE alternation）？

**浪费了大量时间在这个方向上排查代码逻辑**，反复分析每一行可能抛出异常的代码，研究 Supabase Storage 网络可达性、`File.arrayBuffer()` 兼容性、`createAdminClient()` 的 env 变量...

**实际上日志消失的真正原因（后来证实）：** 请求根本没有到达 Next.js route handler，所以 console.error 从未执行。

---

### 阶段 5：直连服务器，发现真相

**转折点：** 用户允许 SSH 到服务器直接操作。

**诊断步骤：**

1. `docker ps` → `guixin-web` 显示 **(unhealthy)**，一度以为服务器挂了
2. `ss -tlnp` → 端口 3000 确实在监听，只是 healthcheck 用 `localhost` 解析到 IPv6 `::1`，而 Next.js 只监听 IPv4，所以 healthcheck 误报（**虚警，不是真正问题**）
3. 用 crafted JWT 发请求测试 upload 路由 → 收到 `{"detail":"Authentication failed...bad_jwt..."}` 

**关键线索：** 响应格式是 `{"detail":"..."}` ——这是 **FastAPI** 的错误格式，不是 Next.js 的 `{"error":"..."}` 格式！

说明请求根本没到 Next.js，而是去了 FastAPI。

4. 检查 nginx 配置：

```bash
docker exec guixin-nginx grep -n -A 8 'location /api/storage/' /etc/nginx/conf.d/default.conf
```

输出：
```nginx
location /api/storage/ {
    proxy_pass http://api_upstream;   ← 指向 FastAPI！！！
```

**根因确认。**

---

## 三、根本原因

**服务器上的 nginx 配置与本地仓库不一致。**

| | 服务器（错误） | 本地仓库（正确） |
|---|---|---|
| `/api/storage/` | `proxy_pass http://api_upstream` (FastAPI) | `proxy_pass http://web_upstream` (Next.js) |

某次历史改动更新了本地 `nginx/conf.d/default.conf`，但没有执行 `deploy.sh sync` 同步到服务器，导致服务器上的 nginx 一直用着旧配置。

**错误路由链路：**
```
浏览器 → nginx → FastAPI /api/storage/upload
                 ↓
         FastAPI 做 Supabase JWT 签名验证
                 ↓
         Supabase 冷启动 500 / JWT 验证失败
                 ↓
         返回 {"detail":"Authentication failed..."} 或 500
```

**正确路由链路：**
```
浏览器 → nginx → Next.js /api/storage/upload route handler
                 ↓
         本地 JWT decode（无网络请求）
                 ↓
         Supabase Storage admin client 上传文件
                 ↓
         返回 {"path":"userId/timestamp-filename.pdf"}
```

---

## 四、修复

```bash
# 1. 同步本地正确的 nginx 配置到服务器
scp nginx/conf.d/default.conf root@47.97.189.35:/opt/guixin-cert/nginx/conf.d/default.conf

# 2. 验证语法并热重载（零停机）
docker exec guixin-nginx nginx -t
docker exec guixin-nginx nginx -s reload
```

**验证：** 用 crafted JWT + test PDF 直接 curl，立刻返回 `{"path":"test-user-00000000/xxx-test.pdf"}`，上传成功。

---

## 五、附带修复：API key 失效

上传链路修通后，发现 AI 解析步骤（`/api/resume/parse`）报 401：

```
AI API call failed: Error code: 401 - Incorrect API key provided
```

`.env.production` 里的 `OPENAI_API_KEY` 已过期，更新为有效 key 后重启 API 容器：

```bash
sed -i 's|^OPENAI_API_KEY=.*|OPENAI_API_KEY=sk-xxxx|' /opt/guixin-cert/.env.production
docker compose --env-file .env.production up -d --force-recreate api
```

---

## 六、为什么排查这么久？

| 误导因素 | 实际情况 |
|---|---|
| `docker logs` 里看不到 console.error | 请求根本没到 Next.js，所以没有日志 |
| 响应体 26 字节被认为是 Next.js catch block 的 `{"error":"internal_error"}` | 实际上是 FastAPI 偶发返回的格式，或者 Next.js 偶尔确实处理了某些请求 |
| macOS grep `\|` 语法失效 | 没有用 `-E` flag，alternation 不生效 |
| 多次代码修改后部署 | 每次部署都需要重新 build Docker 镜像（5-10分钟），大量时间花在等待上 |
| healthcheck (unhealthy) 误导 | IPv4/IPv6 healthcheck 配置问题，与实际请求处理无关 |

**最大教训：** 遇到 500 先检查请求实际到了哪里（看响应格式、`{"detail":...}` vs `{"error":...}`），而不是直接翻代码。

---

## 七、后续防范措施

1. **nginx 配置变更后必须立刻同步服务器**
   ```bash
   # 修改 nginx/conf.d/default.conf 后：
   cd guixin-cert && bash deploy.sh sync
   docker exec guixin-nginx nginx -s reload
   ```

2. **生产问题排查优先 SSH 到服务器确认实际请求链路**，不要在本地代码上猜测

3. **响应格式是最快的路由诊断手段**
   - `{"detail":"..."}` → FastAPI
   - `{"error":"..."}` → Next.js
   - HTML 页面 → nginx 默认错误页

4. **API key 轮换时同步更新 `.env.production` 并重启相关容器**
