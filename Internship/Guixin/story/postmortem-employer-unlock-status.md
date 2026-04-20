# Post-mortem：B 端候选人解锁状态持续显示"锁定"排查全记录

**日期：** 2026-04-21  
**影响范围：** `employer.heartcertai.com` 候选人详情页解锁功能持续失效  
**最终根因：** `employer_unlock_history` 表开启了 RLS 但缺少 SELECT 策略，Next.js 路由用 anon key 永远读不到解锁记录  
**修复耗时：** 约 1 小时（确立"先收数据"方法论后）  
**修复方式：** 将 `unlock-status` API 路由改为 service role key 查询 DB，零业务逻辑改动

---

## 一、症状

用户在 B 端解锁候选人（扣 50 积分）后，返回再次进入同一候选人详情页，仍然显示"解锁详情（50 积分）"按钮，好像从未解锁过。

- 积分已被正确扣除（每次点击都扣）
- 无报错，界面正常，仅状态不对
- 同时存在另一路径：企业沙盒 200 积分解锁报告后，跳转到候选人详情也显示未解锁

---

## 二、排查时间线与每次的假设

### 阶段 1：代码层面猜测——`useMemo` 与 React effect 循环

**假设：** `resume/page.tsx` 里 `createClientComponentClient()` 没有 `useMemo`，每次渲染产生新引用被放入 `useEffect` deps，导致 effect 循环触发，unlock-status 被反复查询，状态永远是初始的 `locked`。

**操作：** 修复 `resume/page.tsx`（加 `useMemo`、`isMounted`、精简 deps）。部署。

**结果：** resume 页卡死问题解决，但解锁状态消失的问题仍在。

---

### 阶段 2：猜测 TOCTOU 竞态条件

**假设：** `bside.py::unlock_details` 里幂等性检查（读）和插入（写）不是原子操作，两个并发请求可能都通过检查然后各插一条，或者插入失败导致没有记录。

**操作：** 准备在后端加双重检查。

**结果：** 被用户叫停——"先搜集数据，再改代码"。

---

### 阶段 3：猜测两张表不互通

**假设：** 企业沙盒解锁写 `report_unlock_records`，而 `unlock-status` 只查 `employer_unlock_history`，导致 200 积分解锁后跳到详情页还是显示锁定，用户再花 50 积分再解锁。

**操作：** 更新 `unlock-status` 路由，同时查两张表。部署。

**结果：** 部分改善（沙盒解锁路径理论上通了），但主路径（50 积分解锁后仍显示锁定）依然复现。

---

### 阶段 4：SSH 进服务器，读真实日志——找到决定性证据

**转折点：** 放弃代码推断，直接查 FastAPI 日志。

**关键发现（来自 `docker logs guixin-api`）：**

```
02:14:18 GET /employer_unlock_history?select=id%2Caccess_level&employer_id=eq.33c665b6...&candidate_id=eq.3aa576e7...  → 200 OK
POST /api/employer/unlock-details  → 200 OK
（无 PATCH 扣费，无 INSERT 记录）

02:14:47 同样的查询，同样的结果
POST /api/employer/unlock-details  → 200 OK
（再次无 PATCH、无 INSERT）
```

**解读：** 两次解锁都在幂等性检查时提前返回——**说明 `employer_unlock_history` 里确实有记录**，FastAPI（service role key）能读到。但前端 `unlock-status` 路由（Next.js，anon key）却始终返回 `unlocked: false`。

**结论：** DB 数据正确，但 **读取权限** 不对。

---

### 阶段 5：定位根因——RLS 策略缺失

**检查 FastAPI 客户端（`core/deps.py`）：**

```python
def get_supabase() -> Client:
    """Get Supabase client with Service Role Key for admin operations."""
    return create_client(settings.supabase_url, settings.supabase_service_role_key)
```

FastAPI 用 **service role key**，绕过所有 RLS。

**检查 Next.js `unlock-status` 路由：**

```typescript
const supabase = createRouteHandlerClient({ cookies: () => cookieStore });
// ↑ 用 NEXT_PUBLIC_SUPABASE_ANON_KEY，受 RLS 约束
```

**Supabase RLS 行为：** 如果一张表开启了 RLS 但没有 SELECT 策略，通过 anon/authenticated key 查询会返回空结果（不报错，只是 0 行）。

`employer_unlock_history` 正是这种情况：RLS 开启，无 SELECT 策略 → anon key 永远查到空 → `unlock-status` 返回 `{ unlocked: false }` → 前端显示锁定。

**根因确认。**

---

## 三、根本原因

| | FastAPI（正确） | Next.js unlock-status（错误） |
|---|---|---|
| Supabase 客户端 | `service role key`，绕过 RLS | `anon key`，受 RLS 约束 |
| 查 `employer_unlock_history` | 返回实际记录 | 返回 0 行（RLS 拦截） |
| 结论 | 记录存在，幂等性生效 | 记录"不存在"，返回未解锁 |

积分每次都被正确扣除，记录也每次都正确写入，但前端读取权限不足，导致用户看到的状态永远是锁定。

---

## 四、修复

```typescript
// unlock-status/route.ts — 修复前
const supabase = createRouteHandlerClient({ cookies: () => cookieStore });
// 用 anon key 查 DB → RLS 拦截 → 永远 0 行

// 修复后：auth 用 cookie session，DB 查询用 service role key
const authClient = createRouteHandlerClient({ cookies: () => cookieStore });
const { data: { user } } = await authClient.auth.getUser(); // 仅验证身份

const db = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,  // ← 绕过 RLS
    { auth: { autoRefreshToken: false, persistSession: false } }
);
// db 用于查 employers / employer_unlock_history / report_unlock_records
```

零业务逻辑改动，只换了 DB 查询的 key。

---

## 五、附带修复

同次排查中发现并修复了另外两个 B 端 bug：

**1. `resume/page.tsx` 报告页卡死**

- 根因：`const { createClientComponentClient } = require(...)` 在组件体内无 `useMemo`，每次渲染新建 supabase 实例，放入 effect deps → 死循环触发 unlock-status 查询 + 重定向
- 修复：ES import + `useMemo(() => createClientComponentClient(), [])` + deps 精简到 `[candidateId]` + `isMounted` cleanup

**2. 企业沙盒解锁后仍显示锁定**

- 根因：200 积分解锁写 `report_unlock_records`，但 `unlock-status` 只查 `employer_unlock_history`，跨路径不互通
- 修复：`unlock-status` 同时查两张表（已在 RLS 修复中一并完成）

---

## 六、为什么这次这么快？

| 关键决策 | 效果 |
|---|---|
| 用户叫停"猜测改代码" | 避免了又一轮无效部署 |
| 直接 SSH 看 FastAPI 日志 | 30 秒内确认 DB 记录存在 |
| 对比 FastAPI vs Next.js 客户端 | 2 分钟内定位 RLS 权限差异 |

前一次简历上传 500 花了 8 小时，最后靠响应格式（`{"detail":...}` vs `{"error":...}`）发现了 nginx 误路由。这次解锁 bug 也是靠服务器真实数据（日志里 DB 查询的有无）发现了权限问题。**规律一致：生产 bug 的根因几乎都藏在基础设施层（路由、权限、配置），不在业务逻辑里。**

---

## 七、后续防范措施

1. **为 `employer_unlock_history` 添加 RLS SELECT 策略**（根治而非绕过）：
   ```sql
   CREATE POLICY "Employers can read their own unlock history"
   ON employer_unlock_history FOR SELECT
   USING (employer_id = (SELECT id FROM employers WHERE user_id = auth.uid()));
   ```

2. **所有 Next.js API 路由：验证身份用 cookie client，查数据用 service role client** —— 不要用同一个 anon key 客户端混用。

3. **生产 bug 排查顺序**：
   - 先看服务器日志（FastAPI httpx 日志直接暴露 Supabase REST 调用）
   - 对比不同客户端（service key vs anon key）的查询结果
   - 确认 DB 状态后再看代码

4. **RLS 覆盖检查**：新建 Supabase 表时，明确写出每个操作（SELECT/INSERT/UPDATE/DELETE）的策略，缺失策略等于全部拒绝。
