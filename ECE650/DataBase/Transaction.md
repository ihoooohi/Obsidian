```text
                    多个用户同时访问数据库
                             │
                             ▼
                       Concurrent Transactions
                     (T1, T2, T3 同时执行)
                             │
                             ▼
                   可能产生并发问题
        ┌─────────────────────────────────┐
        │ 1. Lost Update                  │
        │ 2. Dirty Read                   │
        │ 3. Incorrect Summary            │
        └─────────────────────────────────┘
                             │
                             ▼
                    DBMS 并发控制机制
                  (Concurrency Control)
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   Locking               Timestamp            MVCC
 (锁机制)                Ordering         (多版本并发控制)
        │                    │                    │
        └───────────────┬────┴────┬───────────────┘
                        ▼         ▼
                  控制事务执行顺序
                      (Schedule)
                        │
                        ▼
              产生正确的事务调度
            ┌─────────────────────┐
            │  Serializable       │
            │  Recoverable        │
            │  Strict Schedule    │
            └─────────────────────┘
                        │
                        ▼
               满足数据库 ACID
      ┌──────────────┬──────────────┬──────────────┬──────────────┐
      │ Atomicity    │ Consistency  │ Isolation    │ Durability   │
      └──────────────┴──────────────┴──────────────┴──────────────┘
                        │
                        ▼
                  最终结果
        ┌────────────────────────────────┐
        │ 高并发 + 正确结果 + 可恢复系统 │
        └────────────────────────────────┘

```
# 1. 为什么需要事务（Transaction Processing）

现实数据库系统：

- 用户很多
    
- 操作同时发生
    
- 数据共享
    

例如：

- 银行系统
    
- 机票订票系统
    
- 股票交易系统
    
- 电商系统
    

这些系统需要：

- **高并发**
    
- **高可靠性**
    
- **数据正确性**
    

---

# 2. Transaction（事务）

## 定义

事务是：

> **数据库处理的逻辑操作单位**。

一个事务包含：

- 查询
    
- 插入
    
- 删除
    
- 更新
    

---

## 事务结构

```sql
BEGIN TRANSACTION

SQL operations

COMMIT
```

如果失败：

```sql
ABORT / ROLLBACK
```

---

## 示例（银行转账）

```text
T1:

read(A)
A = A - 100
write(A)

read(B)
B = B + 100
write(B)
```

---

# 3. 数据库操作模型

数据库被抽象为：

> 一组 **data items**。

例如：

- 字段
    
- 记录
    
- 表
    

---

## 基本操作

### read_item(X)

读取数据

步骤：

1. 找到磁盘地址
    
2. 读取到 buffer
    
3. 复制到变量
    

---

### write_item(X)

写入数据

步骤：

1. 找到磁盘
    
2. 读取到 buffer
    
3. 修改
    
4. 写回磁盘
    

---

# 4. 并发事务（Concurrent Transactions）

数据库允许：

```text
T1
T2
T3
```

同时执行。

执行方式：

**interleaving**

例如：

```text
T1 step1
T2 step1
T1 step2
T2 step2
```

这样可以提高：

- CPU 利用率
    
- 系统吞吐量
    

---

# 5. 并发问题（Concurrency Problems）

三个经典问题：

---

## 5.1 Lost Update

两个事务同时更新同一个数据。

例如：

```text
X = 100
```

事务：

```text
T1: X = X - 10
T2: X = X + 20
```

执行顺序：

```text
T1 read 100
T2 read 100
T1 write 90
T2 write 120
```

最终：

```text
X = 120
```

正确答案：

```text
110
```

T1 的更新被覆盖。

---

## 5.2 Dirty Read

事务读取了：

```text
未提交的数据
```

例子：

```text
T1 write X = 900
T2 read X
```

如果：

```text
T1 abort
```

T2 读取的数据就错了。

---

## 5.3 Incorrect Summary

一个事务在 **统计数据**：

```text
SUM(A, X, Y)
```

另一个事务在 **更新数据**。

可能出现：

```text
X 已经减
Y 还没加
```

统计结果错误。

---

# 6. Transaction States（事务状态）

事务生命周期：

```
BEGIN
 ↓
ACTIVE
 ↓
PARTIALLY COMMITTED
 ↓
COMMITTED
 ↓
TERMINATED
```

如果失败：

```
ACTIVE
 ↓
FAILED
 ↓
ABORTED
```

---

# 7. Recovery（恢复机制）

数据库必须保证：

事务要么：

```
全部成功
```

要么：

```
全部失败
```

不能：

```
只执行一半
```

---

## 事务失败原因

- 逻辑错误
    
- 程序错误
    
- 系统 crash
    
- 磁盘损坏
    
- 断电
    
- 并发冲突
    

---

# 8. Transaction Log

数据库会记录 **日志**。

日志类型：

```
[start_transaction, T]
[write_item, T, X, old, new]
[commit, T]
[abort, T]
```

---

## 恢复方式

系统 crash 后：

### committed transactions

```
redo
```

重新执行。

---

### uncommitted transactions

```
undo
```

回滚。

---

# 9. ACID Properties

数据库事务必须满足 **ACID**。

---

## 9.1 Atomicity

事务：

```
all or nothing
```

全部执行或全部撤销。

---

## 9.2 Consistency

事务执行前后：

数据库必须保持 **合法状态**。

例如：

- 外键约束
    
- 主键约束
    
- 数据约束
    

---

## 9.3 Isolation

事务执行时：

```
看起来像独立执行
```

事务之间不会互相影响。

通过：

- Locks
    
- Timestamp
    
- MVCC
    

实现。

---

## 9.4 Durability

事务 **commit 后**：

数据必须永久保存。

即使系统崩溃。

通过：

- transaction log
    
- backup
    
- redo
    

实现。

---

# 10. Schedule（事务调度）

定义：

> 事务操作的执行顺序。

表示方式：

```
r1(X)
w1(X)
r2(X)
```

含义：

```
T1 read X
T1 write X
T2 read X
```

---

# 11. Serial Schedule

最简单的执行方式：

```
T1 完全执行
T2 执行
```

例如：

```
r1(X)
w1(X)
c1
r2(X)
w2(X)
c2
```

优点：

```
一定正确
```

缺点：

```
没有并发
性能差
```

---

# 12. Recoverable Schedule

如果：

```
T2 读取 T1 的数据
```

那么：

```
T2 commit 必须在 T1 commit 之后
```

否则：

如果 T1 abort：

数据库无法恢复。

---

# 13. Cascading Rollback

事务之间依赖：

```
T1 → T2 → T3
```

如果：

```
T1 abort
```

那么：

```
T2 abort
T3 abort
```

这叫：

```
cascading rollback
```

---

# 14. Cascadeless Schedule

规则：

```
事务只能读取
已经 commit 的数据
```

这样可以避免级联回滚。

---

# 15. Strict Schedule

更严格的规则：

如果：

```
T1 写 X
```

那么：

```
其他事务不能 read/write X
直到 T1 commit
```

Strict schedule：

- 最安全
    
- 最容易恢复
    

---

# 16. Serializability（最重要概念）

数据库希望：

```
并发执行
```

但结果必须：

```
等价于某个串行执行
```

如果满足：

这个调度就是：

```
Serializable schedule
```

---

## 串行执行

```
T1 → T2
```

或者

```
T2 → T1
```

---

## 可串行化

例如：

```
T1 read
T2 read
T1 write
T2 write
```

如果最终结果和某个 serial schedule 一样：

就是 serializable。

---

# 17. 事务调度关系

调度安全级别：

```
Strict
   ↓
Cascadeless
   ↓
Recoverable
```

含义：

```
Strict 一定是 Cascadeless
Cascadeless 一定是 Recoverable
```

---

# 18. 整章核心目标

数据库系统必须实现：

```
高并发
高性能
数据正确
系统可恢复
```

但这些之间存在：

```
trade-off
```

例如：

```
更严格控制 → 更安全
但 → 性能下降
```

