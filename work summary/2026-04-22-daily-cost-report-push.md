# Push 式日报：从 usage_events 到飞书卡片

> 源：PR #2355 · 2026-04-21 · Amy 舰队每日 API 成本推送到 Lark webhook

## 我之前可能不知道的事

- "Dashboard" 是 pull，"日报"是 push——**重要的事要来找人**，建 Grafana 看板不等于有了观测性，因为没人每天主动打开。
- `$1::int * INTERVAL '1 hour'` 是 asyncpg 里参数化 SQL 时间窗口的正确姿势——**参数只能绑值不能绑语法元素**，所以 `INTERVAL '$1 hour'` 不工作。
- 算"本窗口 vs 上一窗口"的 %Δ 不需要状态变量或快照表，**一条 SQL 查两个相邻时间段** 就够了：`BETWEEN now() - 2N AND now() - N`。
- 零依赖约束是**主动的设计选择**——stdlib 的 `urllib.request` 虽然丑但永远不会 pip 安装失败；运维脚本能用 stdlib 就不要 requests/httpx。
- `docker exec <existing-container> python3 script.py` 是一种"寄生设计"——借用已有容器的 Python + 依赖 + 网络 + env，**不新增任何运行时**。
- cron 脚本的 stdout 重定向必须带 `2>&1`，否则 stderr 会变成邮件或被吞掉，丢 error 的 debug 极其痛苦。

## 核心模型

一个标准的"日报式观测"有四步：**聚合（SQL 窗口）→ 封装（dataclass）→ 渲染（card）→ 推送（webhook POST）**。
核心是让"昨天发生了什么"从躺在表里变成**每天同一时间撞进相关人视野**的一张卡，而不是依赖人主动去查。
设计上追求**零新依赖 / 零新容器 / 零新端点**——寄生在已有容器、用 stdlib 通信、用 host cron 调度。失败只影响自己、爆炸半径最小。
关键实现技巧是把"当前窗口"和"上一窗口"的对比塞进 SQL 的 `BETWEEN now()-2N AND now()-N`，让趋势计算无需应用层状态。

## 可迁移的 3 个通用知识

1. **Push vs Pull 观测性** —— 变成"不看也会撞进视野"的信号。适用场景：SLA 违规告警、周报、成本异常、新用户激活 digest。
2. **寄生设计** —— 新功能借用已有容器/进程的环境跑，不新增运行时。Trade-off：耦合 vs 简洁。非关键 side-task 首选寄生。
3. **参数化 SQL 时间窗口 + 相邻窗口对比** —— 任何带 `created_at` 的事件表，用 `$1::int * INTERVAL '1 hour'` 做动态窗口，用 `BETWEEN now()-2N AND now()-N` 拿前一窗口。环比同比通用模板。

## 最容易踩的坑

- 「SQL INTERVAL 参数化」—— 写成 `INTERVAL '$1 hour'` 会语法错或参数不替换 —— 因为参数绑定只替换值不替换语法 —— 写成 `$1::int * INTERVAL '1 hour'`，参数是数值倍数。
- 「除零 / 基数为 0 的百分比」—— `cache_read / total_input_tokens` 在空舰队日抛 ZeroDivisionError —— 因为 Python 不返回 NaN —— 用 `max(1, denominator)` 兜底；"无前值"场景用 sentinel（`"new"` / `"–"`）而不是 0%。
- 「固定 30 天预测跨窗口展示」—— 7 天窗口 × 30 = 胡说八道 —— 因为指标的有效性依赖窗口类型 —— 根据 `since_hours` 条件抑制不适用的派生指标。
- 「cron 脚本缺 `2>&1`」—— 成功日志写文件、失败 traceback 丢或发邮件 —— 因为默认 stderr 不跟 stdout 一起重定向 —— 永远写 `>> log 2>&1`；忘了就只剩"今天没收到卡"一个症状，没法定位。

## 下次遇到类似问题我会先想

- 写运维/观测脚本 → 先问"能不能 docker exec 进已有容器跑？" 避免新起 runtime。
- 需要算趋势 %Δ → 先想能不能用 SQL 的相邻时间窗口对比，而不是建状态变量/快照表。
- 任何有外部副作用（发消息/写库/打钱）的脚本 → 第一个参数加 `--dry-run`，打印不发送。没 `--dry-run` 的脚本不上生产。
- 展示类指标 → 先想"当数据为空或基数为 0 时这张卡长啥样？"，空状态要有独立文案，不能让 `↑inf%` 漏出去。
- 配置 → "env override → secondary env → hardcoded default" 三级回退，比强制环境变量更易用也更 robust。

---
*相关 PR*: https://github.com/Vispie-AI/VisPie_backend/pull/2355
*首次接触日期*: 2026-04-22
