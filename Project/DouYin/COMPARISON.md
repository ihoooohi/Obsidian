# 性能对比：Baseline vs Optimized vs Refactor

## 版本说明

| 版本 | 说明 | 优化内容 |
|------|------|----------|
| **Baseline** | 基线版本 | 禁用Redis缓存、禁用RabbitMQ、Feed回退到DB、Like/Comment同步写DB |
| **Optimized** | 优化版本 | 启用Redis缓存、启用RabbitMQ异步写、Worker单进程单channel单goroutine |
| **Refactor** | 重构版本 | 在Optimized基础上，Worker并发优化（预期） |

## 核心指标对比

### S1 纯读场景（1200 VUs）

| 版本 | QPS | Feed P95 | QPS vs Baseline | P95 vs Baseline |
|------|-----|----------|-----------------|-----------------|
| Baseline | 3,209 | 1,127ms | - | - |
| Optimized | 13,696 | 132ms | +327% | -88% |
| Refactor | 15,137 | 113ms | +372% | -90% |

**关键发现**：
- Optimized 相比 Baseline，QPS 提升 **327%**，P95 降低 **88%**
- Refactor 相比 Optimized，QPS 提升 **11%**，P95 降低 **14%**

### S2 混合场景（1500 VUs, 80%读/20%写）

| 版本 | 总 QPS | Feed P95 | Like P95 | Comment P95 | QPS vs Baseline | Feed P95 vs Baseline |
|------|--------|----------|----------|-------------|-----------------|----------------------|
| Baseline | 2,954 | 2,092ms | - | - | - | - |
| Optimized | 6,710 | 569ms | - | - | +127% | -73% |
| Refactor | 7,715 | 439ms | 3,595ms | 3,120ms | +161% | -79% |

**关键发现**：
- Optimized 相比 Baseline，QPS 提升 **127%**，Feed P95 降低 **73%**
- Refactor 相比 Optimized，QPS 提升 **15%**，Feed P95 降低 **23%**
- Refactor 写延迟仍然很高（Like P95=3.6s，Comment P95=3.1s）

### S3 写峰值场景（1800 VUs, 50%读/50%写）

| 版本 | 总 QPS | Feed P95 | 总 P95 | QPS vs Baseline | Feed P95 vs Baseline |
|------|--------|----------|---------|-----------------|----------------------|
| Baseline | 3,304 | 2,148ms | - | - | - |
| Optimized | 4,746 | 437ms | - | +44% | -80% |
| Refactor | 5,068 | 1,171ms | 1,492ms | +53% | -46% |

**关键发现**：
- Optimized 相比 Baseline，QPS 提升 **44%**，Feed P95 降低 **80%**
- Refactor 相比 Optimized，QPS 提升 **7%**，但 Feed P95 反而上升 **168%**
- Refactor 在写峰值场景下性能下降明显

## 版本间对比分析

### Baseline → Optimized

| 场景 | QPS 提升 | Feed P95 降低 | 主要优化点 |
|------|----------|---------------|------------|
| S1 纯读 | +327% | -88% | Redis缓存、异步写 |
| S2 混合 | +127% | -73% | Redis缓存、RabbitMQ异步 |
| S3 写峰值 | +44% | -80% | Redis缓存、RabbitMQ异步 |

**结论**：Optimized 版本通过引入 Redis 缓存和 RabbitMQ 异步写入，性能提升显著，尤其是在纯读场景。

### Optimized → Refactor

| 场景 | QPS 提升 | Feed P95 变化 | 问题分析 |
|------|----------|---------------|----------|
| S1 纯读 | +11% | -14% | 轻微提升 |
| S2 混合 | +15% | -23% | 轻微提升 |
| S3 写峰值 | +7% | +168% | **性能退化** |

**结论**：Refactor 版本在 S1/S2 场景有轻微提升，但在 S3 写峰值场景下性能严重退化，Feed P95 从 437ms 恶化到 1,171ms。

## 核心瓶颈识别

### 当前 Refactor 版本的主要问题

1. **写延迟极高**
   - Like P95: 3,595ms
   - Comment P95: 3,120ms
   - 这是单 goroutine 串行消费导致的瓶颈

2. **写峰值场景性能退化**
   - S3 场景 Feed P95 从 437ms 恶化到 1,171ms
   - 可能是队列积压导致读请求被阻塞

3. **Worker 并发不足**
   - 当前：单进程、单 connection、单 channel、单 goroutine
   - 建议：每类队列独立 channel，多 goroutine 并发消费

## 优化建议

### P0 止血项：Worker 并发扩容

| Worker 类型 | 当前并发 | 建议并发 | 预期效果 |
|-------------|----------|----------|----------|
| Like Worker | 1 | 8 | 写延迟降低 70-80% |
| Comment Worker | 1 | 4 | 写延迟降低 70-80% |
| Popularity Worker | 1 | 8 | 更新延迟降低 70-80% |

### P1 优化项：队列配置优化

- prefetch 与并发数匹配
- 每类队列使用独立 channel
- 避免多个 goroutine 共享消费 channel

## 预期优化效果（基于 Worker 并发扩容）

| 场景 | 当前 QPS | 目标 QPS | 当前 P95 | 目标 P95 |
|------|----------|----------|----------|----------|
| S1 纯读 | 15,137 | 18,000 | 113ms | 100ms |
| S2 混合 | 7,715 | 10,000 | 439ms | 200ms |
| S2 Like | - | 1,600 | 3,595ms | 800ms |
| S2 Comment | - | 400 | 3,120ms | 800ms |
| S3 写峰值 | 5,068 | 7,000 | 1,171ms | 300ms |

**预期提升**：
- S2 写延迟降低 **78%**（Like P95: 3,595ms → 800ms）
- S3 写峰值场景性能恢复并超越 Optimized 版本
- 整体 QPS 提升 **30-40%**
