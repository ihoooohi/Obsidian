## Relational Algebra vs. SQL Queries

关系代数是过程式的，需要明确指定查询的执行顺序；而 SQL 是声明式的，只需说明要什么结果，执行方式由数据库系统自动优化决定。

当你写 SQL 时：

1. 解析 SQL
2. 转换成关系代数表达式
3. 生成多个执行计划
4. 计算成本（Cost）
5. 选择最优执行方案

这个过程叫：
👉 Query Optimization（查询优化）

## 外键违反参照完整性时怎么办（Referential Integrity Actions）

### 一、什么是参照完整性？

当一个表有外键（Foreign Key）时：

- 子表的外键值
    
- 必须在父表中存在

==**外键引用的主键必须存在**==

否则就叫：

👉 违反参照完整性

这种情况可能发生在：

- 插入（INSERT）
    
- 删除（DELETE）
    
- 更新（UPDATE）
    

---

### 二、如果违反了怎么办？

SQL 允许你在定义外键时指定：

> 触发动作（referential triggered action）

---

### 三、可选动作（Options）

#### 1️⃣ SET NULL

把子表的外键设为 NULL

意思：  
父表记录被删或更新时，  
子表不再引用它。

---

#### 2️⃣ CASCADE

级联更新或删除

意思：  
父表删除 → 子表也删除  
父表更新 → 子表也跟着更新

这是最常见的。

---

#### 3️⃣ SET DEFAULT

把外键设为默认值

前提是：  
该列必须定义 DEFAULT 值。

---

### 四、这些动作必须配合使用

必须指定触发时机：

#### ON DELETE

当父表记录被删除时

#### ON UPDATE

当父表记录被更新时

---

### 五、举个例子

```sql
FOREIGN KEY (dept_id)
REFERENCES Department(id)
ON DELETE CASCADE
ON UPDATE CASCADE;
```

意思：

- 删除部门 → 自动删除该部门所有员工
    
- 修改部门 id → 员工表也自动更新
    

---

### 六、一句话总结

当外键引用的父表数据被删除或更新时，可以通过 ON DELETE / ON UPDATE 指定 SET NULL、CASCADE 或 SET DEFAULT 等动作来维护参照完整性。