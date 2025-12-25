

> **pgx** 是 Go 语言中性能最好、功能最完整的 PostgreSQL 驱动之一，既可以当 **database/sql driver** 用，也可以直接用 **原生 API（推荐）**。

---

## 1. pgx 是什么

- PostgreSQL 官方推荐的 Go 驱动之一
    
- 比 `lib/pq` **更快**、**功能更全**
    
- 支持：
    
    - 连接池
        
    - 事务
        
    - 批量操作
        
    - Copy
        
    - Context（超时/取消）
        

🔑 **核心模块**：

- `pgx/v5`
    
- `pgxpool`
    

---

## 2. 安装

```bash
go get github.com/jackc/pgx/v5
```

如果用连接池（强烈推荐）：

```bash
go get github.com/jackc/pgx/v5/pgxpool
```

---

## 3. 最小可用示例（连接 + 查询）

```go
conn, err := pgx.Connect(context.Background(), "postgres://user:pass@localhost:5432/dbname")
if err != nil {
    panic(err)
}
defer conn.Close(context.Background())

var now time.Time
err = conn.QueryRow(context.Background(), "select now()").Scan(&now)
```

---

## 4. 推荐用法：pgxpool（连接池）

### 4.1 创建连接池

```go
pool, err := pgxpool.New(context.Background(), os.Getenv("DATABASE_URL"))
if err != nil {
    panic(err)
}
defer pool.Close()
```

### 4.2 查询单行

```go
var name string
err := pool.QueryRow(ctx,
    "select name from users where id=$1",
    userID,
).Scan(&name)
```

---

## 5. SQL 占位符规则（非常重要）

PostgreSQL **只支持 $1 $2 ...**，不支持 `?`

```sql
select * from users where id = $1 and status = $2
```

❌ 错误：

```sql
select * from users where id = ?
```

---

## 6. 查询多行（Query）

```go
rows, err := pool.Query(ctx, "select id, name from users")
if err != nil {
    return err
}
defer rows.Close()

for rows.Next() {
    var id int
    var name string
    err := rows.Scan(&id, &name)
}
```

---

## 7. Exec（不关心返回行）

```go
cmdTag, err := pool.Exec(ctx,
    "update users set name=$1 where id=$2",
    name, id,
)

rows := cmdTag.RowsAffected()
```

---

## 8. 事务（Transaction）

```go
tx, err := pool.Begin(ctx)
if err != nil {
    return err
}

defer tx.Rollback(ctx) // 安全写法

_, err = tx.Exec(ctx, "insert into users(name) values($1)", name)
if err != nil {
    return err
}

return tx.Commit(ctx)
```

---

## 9. 批量操作（Batch）

```go
batch := &pgx.Batch{}
batch.Queue("insert into logs(msg) values($1)", "a")
batch.Queue("insert into logs(msg) values($1)", "b")

br := pool.SendBatch(ctx, batch)
br.Close()
```

---

## 10. 扫描到结构体（常见模式）

```go
type User struct {
    ID   int
    Name string
}

rows, _ := pool.Query(ctx, "select id, name from users")
for rows.Next() {
    var u User
    rows.Scan(&u.ID, &u.Name)
}
```

📌 pgx **不自动 ORM 映射**（这是优点）

---

## 11. NULL 处理

### 方式 1：sql.NullXXX

```go
var name sql.NullString
```

### 方式 2：指针（推荐）

```go
var name *string
```

---

## 12. Context 是必须的

- 所有 pgx API 都需要 `context.Context`
    
- 用来控制：
    
    - 超时
        
    - 取消
        

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
```

---

## 13. 常见错误处理

### 查不到数据

```go
err := pool.QueryRow(ctx, sql).Scan(&v)
if err == pgx.ErrNoRows {
    // 没数据
}
```

---

## 14. pgx vs database/sql

|对比|pgx|database/sql|
|---|---|---|
|性能|⭐⭐⭐⭐|⭐⭐|
|PostgreSQL 特性|全支持|部分|
|API|更现代|通用|

👉 **推荐：直接用 pgx / pgxpool**

---

## 15. 项目中推荐的结构

```text
/db
  ├── pool.go      // 初始化 pgxpool
  ├── user_repo.go // SQL 封装
```

---

## 16. 一句话总结

- pgx 是 **Go + PostgreSQL 的最佳实践**
    
- 永远用 `$1` 占位符
    
- 永远用 `context`
    
- 生产环境用 `pgxpool`
    

---

## 17. 速查代码片段

```go
pool.QueryRow(ctx, sql, args...).Scan(&v)
pool.Query(ctx, sql)
pool.Exec(ctx, sql)
pool.Begin(ctx)
```