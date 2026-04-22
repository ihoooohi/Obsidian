# 部署验证：从「sleep + grep "Up"」到「polling health status」

> 源：PR #2350 · 2026-04-21 · 修复 Unified Deployment 40% 概率红叉的"伪 one-api-v2 噪声"

## 我之前可能不知道的事

- `Up (health: starting)` 也被 grep `"Up"` 匹配到——**子串匹配是错的判据**，字符串包含并不等价于"运行中"。
- Docker 的 `start_period: 60s` **不是** "等 60 秒再开始 healthcheck"，而是"前 60 秒的 healthcheck 失败不算数"。check 一直在跑。
- `docker ps | grep <name>` 会同时命中同名前缀容器，而 `--force-recreate` 期间新老容器并存的那几秒内就会踩这个坑。改用 `docker inspect <name>` 是精确匹配。
- CI 红叉里堆满某个无关容器的 stderr——**常常不是凶手**，只是脚本 `docker-compose logs`（全量）把相邻容器的日志一起喷出来让人追错方向。信噪比比日志量重要。
- Docker 已经自己在跑 healthcheck 了，结果缓存在 `.State.Health.Status`——不用自己再造一份 `curl` 验证逻辑，直接读这个字段。

## 核心模型

部署验证的正确姿势是"**读权威事实，且 polling 直到条件成立**"，不是"**sleep 一个猜的秒数，然后 grep 一下**"。
Docker / K8s / systemd 都把"进程在跑（liveness）"和"服务能接请求（readiness）"分成两件事——验证脚本该看后者。
配合 `start_period` 作为启动宽限期、polling 循环作为自适应等待、`tail -N` 作为错误时的噪声隔离——这三件事一起，让"失败"变成一个明确的信号而不是一锅烂粥。

## 可迁移的 3 个通用知识

1. **Liveness vs Readiness vs Startup** —— 进程活着 ≠ 服务就绪 ≠ 首次完成初始化。K8s Pod、ALB target、Cloud Run startup probe 里都有对应物。设计长启动服务时先分清是哪一层。
2. **TOCTOU / 时序竞争** —— `sleep N` 后 `check` 是靠巧合过关的。修复模板：(a) 消除 gap 改原子操作，或 (b) 把"一次性 check"改成"持续 polling 直到条件成立"。数据库 SELECT→UPDATE、文件 stat→open、分布式锁 check→acquire 都适用。
3. **用已有事实，不造新事实** —— 系统里已经有一份权威数据时（Docker healthcheck、DB sequence、K8s pod status），不要在外面再造一份验证逻辑。双源数据必然出现不一致。

## 最容易踩的坑

- 「`grep "Up"` 作为健康判据」—— `Up (health: starting)` 也匹配 → 容器还在热身就被判"已就绪" —— 因为 shell 的子串匹配语义和 Docker 状态字符串设计相撞 —— 改用结构化字段 `.State.Health.Status == "healthy"`。
- 「`sleep N` 作为等待策略」—— 要么不够用频繁失败、要么过量拖慢 CI —— 因为 sleep 是对系统行为的猜测，没有反馈 —— 改成 polling + 早退 + 超时兜底（`for i in 1..N; break if ready`）。
- 「错误分支 dump 全量日志」—— 相邻无关服务的噪声淹没真正的故障信号，人被误导去追错的锅 —— 因为 `docker-compose logs` 默认是所有服务全量 —— 限定服务 + `--tail N`，只给诊断最近的一小段。
- 「顺手重构没坏的地方」—— Django 检查没问题偏要一起改成 polling，但 Django 没配 healthcheck，反而 timeout —— 因为把"优雅"看得比"稳定"重 —— Scope kept minimal，**没坏的东西不要碰**。

## 下次遇到类似问题我会先想

- 部署/启动验证 → 先问"这个系统已经有哪个组件在判断健康？" 直接读它，别自己 curl。
- 看到 CI 日志刷屏 → 先问"这些日志的来源是不是脚本错误分支的 `logs` dump？" 别急着追日志里最醒目的错误。
- 任何 `sleep N` 或"先 check 后 use" → 立刻警觉是不是 race condition，用 polling 或原子操作重写。
- 健康检查相关的修复 → 先想清楚是 liveness / readiness / startup 哪一层的问题，再选方案。

---
*相关 PR*: https://github.com/Vispie-AI/VisPie_backend/pull/2350
*首次接触日期*: 2026-04-22
