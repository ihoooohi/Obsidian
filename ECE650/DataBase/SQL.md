## Overview

### 1. 关系模型基础 (Relational Model Basics)

- **关系/表 (Relation/Table)**：例如一张名为 `Employee Table`（员工表）的表格。
- **元组/行 (Tuple/Row)**：代表一组相关联的数据值，例如员工表中的一行数据 `<Reynolds, Manager, Sales, 555-555-5444>`。
- **属性/列 (Attribute/Column)**：列的标头，用于说明值的含义，例如 `Name`（姓名）、`Position`（职位）、`Department`（部门）、`Phone #`（电话号码）。
- **域 (Domain)**：某列允许的取值范围和格式，例如 `USA_phone_numbers`（格式为 `(ddd)ddd-dddd` 的10位电话号码字符集合），或 `Grade_point_averages`（0到4之间的实数集合）。
- **关系模式 (Relation Schema)**：表的定义结构，例如 `STUDENT(Name, SSN, HomePhone, Address, OfficePhone, Age, GPA)`。

### 2. 关系约束 (Relational Constraints)

- **超键 (Superkey) 与 键 (Key)**：在上述 `STUDENT` 关系中，`{SSN, Name, Age}` 是超键（保证每行唯一），而极简的 `{SSN}` 是键（从中移除任何属性就不再是超键）。习惯上主键会被带有下划线标出，如汽车表中的 `LicenseNum`。
- **实体完整性约束 (Entity Integrity Constraint)**：主键绝不能为NULL。例如：尝试往 `EMPLOYEE` 表中插入一条主键 `SSN` 为 `null` 的记录 `<‘Cecilia’, ‘F’, ‘Kolonsky’, null, ...>` 会因为违反实体完整性而被拒绝。
- **NULL约束 (NULL Constraints)**：普通属性是否允许为空。例如，规定 `Name` 属性必须有有效值，则它会被约束为 `NOT NULL`。
- **参照完整性约束 (Referential Integrity Constraint)**：外键必须引用真实存在的值。例如 `EMPLOYEE` 表中的外键 `DNO` 必须引用 `DEPARTMENT` 表中存在的 `DNUMBER`。
- **语义与状态约束 (Semantic & Transition Constraints)**：语义约束例如“员工工资不能超过其主管工资”；状态约束例如“员工的任职时间只能随时间增加”。

### 3. 关系代数操作 (Relational Algebra)

- **选择 (SELECT, $\sigma$)**：按行过滤，例如 $\sigma_{(DNO=4 \text{ AND } SALARY > 50000)}(EMPLOYEE)$（选出4号部门且工资大于5万的员工）。
- **投影 (PROJECT, $\pi$)**：按列筛选，例如 $\pi_{LNAME, FNAME, SALARY}(EMPLOYEE)$（仅保留员工的姓、名、工资这三列）。
- **重命名 (RENAME)**：保存中间结果，例如将上述投影结果重命名为 $R(LASTNM, FIRSTNM, SALARY) = \pi_{LNAME, FNAME, SALARY}(TMP)$。
- **集合操作 (Set Operations)**：例如寻找在5号部门工作的员工集合 ($RESULT1$) **与** 管理5号部门的员工集合 ($RESULT2$) 的并集：$RESULT1 \cup RESULT2$。
- **笛卡尔积 (Cartesian Product, $\times$)**：例如将女员工名表和家属表直接组合，`EMPNAMES \times DEPENDENT`。
- **连接 (JOIN, $\bowtie$)**：例如找出各部门对应的经理，`DEPARTMENT \bowtie_{MGRSSN=SSN} EMPLOYEE`。

### 4. SQL 数据定义与表管理 (DDL)

- **数据类型 (Data Types)**：如 `VARCHAR(15)`（变长字符串）、`CHAR(9)`（定长字符串）、`DECIMAL(10,2)`（数字精度10，小数点后2位）、`DATE`（日期）等。
- **默认值 (Default Values)**：如果不填入数据则使用预设值，例如 `DNO INT NOT NULL DEFAULT 1`。
- **创建表 (CREATE TABLE)**：例如 `CREATE TABLE Employee (FNAME VARCHAR(15) NOT NULL, SSN CHAR(9) NOT NULL, PRIMARY KEY (SSN));`。
- **修改表 (ALTER TABLE)**：添加列如 `ALTER TABLE Employee ADD JOB VARCHAR(12);`；删除列如 `ALTER TABLE Employee DROP ADDRESS CASCADE;`。
- **参照完整性触发动作 (Referential Integrity Actions)**：
    - 如果引用的主键记录被删除了，把当前外键设为NULL：`ON DELETE SET NULL`。
    - 如果引用的主键更新了，当前外键也跟着更新：`ON UPDATE CASCADE`。

### 5. SQL 查询与数据操作 (DML & DQL)

- **插入数据 (INSERT)**：例如 `INSERT INTO Employee (SSN, FNAME, LNAME) VALUES ('123456789', 'Paul', 'Weng');`。
- **更新数据 (UPDATE)**：例如将所有女性员工的工资翻倍 `UPDATE Employee SET salary = salary*2 WHERE sex = 'F';`。
- **基本查询 (SELECT-WHERE)**：例如查找John B Smith的生日和地址 `SELECT BDATE, ADDRESS FROM EMPLOYEE WHERE FNAME='John' AND MINIT='B' AND LNAME='Smith';`。
- **表别名 (Aliasing)**：常用于表自连接，比如为了查询员工姓名及其对应的主管姓名，把同一张表当作 `E` 和 `S` 两张用：`SELECT E.FNAME, E.LNAME, S.FNAME, S.LNAME FROM EMPLOYEE AS E, EMPLOYEE AS S WHERE E.SUPERSSN=S.SSN;`。
- **模糊匹配 (LIKE)**：例如查找地址中包含Houston, TX的员工 `WHERE ADDRESS LIKE '%Houston%TX%';`。
- **算术运算 (Arithmetic)**：例如在查询时模拟加薪10% `SELECT FNAME, 1.1*SALARY FROM EMPLOYEE;`。
- **排序 (ORDER BY)**：例如按照部门倒序、姓氏正序排列 `ORDER BY DNAME DESC, LNAME ASC, FNAME ASC;`。
- **嵌套查询 - IN/ALL**：例如找出比5号部门所有员工工资都高的人 `WHERE SALARY > ALL (SELECT SALARY FROM EMPLOYEE WHERE DNO=5);`。
- **存在性检查 (EXISTS)**：例如找出没有任何家属的员工 `WHERE NOT EXISTS (SELECT * FROM DEPENDENT WHERE SSN=ESSN);`。
- **NULL 值检查**：例如找出没有上级主管的员工必须使用 `IS NULL`：`WHERE SUPERSSN IS NULL;`。
- **聚合函数与分组 (GROUP BY)**：例如统计每个部门的部门编号、总人数和平均工资 `SELECT DNO, COUNT(*), AVG(SALARY) FROM EMPLOYEE GROUP BY DNO;`。

### 6. 高级特性与安全性

- **视图 (SQL Views)**：用于保存复杂查询的“虚拟表”。例如经常需要看员工负责的项目时长，可以创建一个视图：`CREATE VIEW WORKS_ON1 AS SELECT FNAME, LNAME, PNAME, HOURS FROM EMPLOYEE, PROJECT, WORKS_ON ...;`。
- **SQL 注入危险 (SQL Injection)**：如果直接拼接用户输入，恶意用户可能会输入 `'; INSERT INTO USERS VALUES ('hacker', 'mypassword', True);` 从而直接在数据库里植入后门。
- **对象关系映射 (ORM)**：最佳的防注入/数据库交互方式，不用写原生SQL语句，而是操作代码对象，例如 `Person p = repository.GetPerson(10); String name = p.getFirstName();`。
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

# **模糊匹配 (LIKE)**

这张图片展示的是 SQL 语言中的 **LIKE 子句**，它主要用于在 `WHERE` 语句中进行**模糊查询**（字符串匹配）。

比起直接用 `=` 要求完全一致，`LIKE` 允许你利用“通配符”来查找符合某种模式的文本。图中提到了两个最核心的通配符：

### 1. 核心通配符

- **百分号 `%`**：代表**任意数量**（零个、一个或多个）的字符。
    
    - _例子_：`'%Houston%'` 表示只要中间包含 "Houston" 这个词，前后是什么、有多少字符都无所谓。
        
- **下划线 `_`**：代表**有且仅有一个**字符。
    
    - _例子_：`'195_'` 要求 195 后面必须跟一个字符。
        

---

### 2. 图片中的案例解析

#### 案例 A：查找地址

SQL

```
SELECT FNAME, LNAME
FROM EMPLOYEE
WHERE ADDRESS LIKE '%Houston%TX%';
```

- **逻辑**：系统会筛选出所有地址中包含 "Houston" 且之后包含 "TX" 的员工。
    
- **匹配示例**：
    
    - "123 Houston St, Dallas, TX" (匹配)
        
    - "Houston, TX" (匹配)
        
    - "TX, Houston" (**不匹配**，因为顺序不对)
        

#### 案例 B：查找出生日期（1950年代）

SQL

```
SELECT FNAME, LNAME
FROM EMPLOYEE
WHERE BDATE LIKE '195_-_ _-_ _';
```

- **逻辑**：利用下划线精确限制位置。
    
- **格式**：日期格式为 `YYYY-MM-DD`。
    
- **解析**：`195_` 锁定了年份必须是 1950 到 1959 年之间。后面的 `_ _-_ _` 则是为月份和日期占位，确保整个字符串的长度和格式符合日期标准。
    

---

### 3. 使用小贴士

- **大小写敏感**：在某些数据库（如 PostgreSQL）中，`LIKE` 是大小写敏感的；而在另一些（如 MySQL 或 SQL Server 的默认配置）中可能不敏感。如果想强制忽略大小写，通常使用 `ILIKE`。
    
- **性能考量**：以 `%` 开头的查询（如 `LIKE '%abc'`）通常无法利用数据库索引，查询速度会比较慢。如果数据量很大，要谨慎使用。

# GROUP BY 查询

这一行 SQL 语句是数据库查询中非常核心的部分，它展示了如何将数据**分门别类**并进行**统计计算**。

简单来说，它的逻辑是：**“先分组，再计算”**。

我们可以把这句 SQL 拆解成三个部分来理解：

## 1. 核心动作：`GROUP BY DNO` (分组)

这是整个语句的灵魂。

- **逻辑**：数据库会扫描 `EMPLOYEE`（员工）表，看 `DNO`（部门编号）这一列。所有 `DNO` 相同的员工会被“打包”在一起。
    
- **比喻**：想象你有一堆各色的小球（员工），每个小球上写着所属的颜色（部门）。`GROUP BY` 就是把红色球放一堆，蓝色球放一堆。
    

## 2. 统计工具：聚合函数 (`COUNT`, `AVG`)

当数据被分成一个个小组后，我们就不能再看单个员工的信息了，而是要看**小组的整体特征**。

- **`COUNT(*)`**：统计每个小组里有多少个“球”（即每个部门的总人数）。
    
- **`AVG(SALARY)`**：计算每个小组里所有员工工资的平均值。
    
- **其他常用函数**：还有 `SUM()`（求和）、`MAX()`（最大值）、`MIN()`（最小值）。
    

## 3. 展示结果：`SELECT DNO, ...`

- **规则**：在使用 `GROUP BY` 时，`SELECT` 后面能跟的列非常有限。通常只能放：
    
    1. **被分组的列**（即 `DNO`），因为它对小组来说是唯一的。
        
    2. **聚合函数的结果**（如人数、平均分），因为它们代表了小组的属性。
        
- **错误示例**：如果你试图在 `SELECT` 里加上 `FNAME`（员工姓名），数据库会报错。因为它不知道在该显示小组里哪一个员工的名字。
    

---

## 举个例子（形象化）：

假设 `EMPLOYEE` 表数据如下：

|**FNAME**|**SALARY**|**DNO (部门)**|
|---|---|---|
|张三|5000|1|
|李四|7000|1|
|王五|9000|2|

执行 `SELECT DNO, COUNT(*), AVG(SALARY) FROM EMPLOYEE GROUP BY DNO;` 后，结果为：

|**DNO**|**COUNT(*)**|**AVG(SALARY)**|
|---|---|---|
|**1**|2|6000|
|**2**|1|9000|

---

## 进阶：如果你想过滤结果怎么办？

如果你只想看“平均工资超过 5000 的部门”，你**不能**用 `WHERE`，而要用 **`HAVING`**：

SQL

```
SELECT DNO, COUNT(*), AVG(SALARY)
FROM EMPLOYEE
GROUP BY DNO
HAVING AVG(SALARY) > 5000;
```

- **WHERE**：在分组**前**过滤掉不符合条件的行。
    
- **HAVING**：在分组**后**过滤掉不符合条件的小组。

# 嵌套查询（Nested Queries）

假设我们有两张表：

1. **DEPT（部门表）**：记录部门 ID 和 所在地（LOCATION）。
    
2. **EMP（员工表）**：记录员工姓名和所属的 部门 ID（DNO）。
    

---

## 1. 为什么需要嵌套查询？

如果你只看 `EMP` 表，你只知道员工在 “部门 1”，但不知道“部门 1”是不是在北京。

如果你只看 `DEPT` 表，你只知道“北京”是 “部门 1”，但不知道谁在那工作。

所以你的逻辑是：

- **第一步（内层）**：去 `DEPT` 表查出“北京”对应的部门 ID 是多少。
    
- **第二步（外层）**：拿着这个 ID，去 `EMP` 表找对应的员工。
    

---

## 2. SQL 代码实现

SQL

```
SELECT FNAME, LNAME
FROM EMPLOYEE
WHERE DNO IN (SELECT DNUMBER 
              FROM DEPARTMENT 
              WHERE LOCATION = 'Beijing');
```

## 3. 直观拆解

- **内层查询 (Inner Query)**：`SELECT DNUMBER FROM DEPARTMENT WHERE LOCATION = 'Beijing'`
    
    - **结果**：假设北京部门的 ID 是 `7`。
        
- **外层查询 (Outer Query)**：`SELECT FNAME, LNAME FROM EMPLOYEE WHERE DNO IN (7)`
    
    - **结果**：系统最终只会列出 `DNO` 为 7 的员工。
        

---

## 4. 为什么要用 `IN` 而不是 `=`？

这是一个非常重要的细节：

- 如果北京只有一个部门，用 `DNO = (SELECT ...)` 没问题。
    
- 但如果北京有两个部门（比如 ID 是 7 和 8），内层查询会返回一个**集合** `{7, 8}`。
    
- `=` 无法处理一个集合，而 **`IN`** 就像是一个“包含”判断：只要员工的部门 ID 在这个 `{7, 8}` 里面，就把他选出来。
    

# Join on 查询

用“北京部门”这个例子来解释 **JOIN**，你会发现它非常像是在玩**连连看**或者**拼图**。

## 1. 核心逻辑：把两张表“拼”成一张表

在嵌套查询里，我们是先去 `DEPT` 表拿 ID，再回 `EMP` 表找人。 在 **JOIN** 里，我们直接把 `EMP` 表和 `DEPT` 表横向连接起来，变成一个**信息超级全的大宽表**。

---

## 2. SQL 语句实现

SQL

```
SELECT E.FNAME, E.LNAME
FROM EMPLOYEE AS E JOIN DEPARTMENT AS D ON E.DNO = D.DNUMBER
WHERE D.LOCATION = 'Beijing';
```

---

## 3. 直观拆解（三步走）

#### 第一步：FROM ... JOIN (选择要拼的表)

我们告诉数据库：把 `EMPLOYEE` 表（简称 E）和 `DEPARTMENT` 表（简称 D）拉过来。

#### 第二步：ON E.DNO = D.DNUMBER (寻找连接点)

这是最关键的一步！**ON 后面写的是连接条件。**

- 数据库会逐行扫描 `EMP` 表，看到张三的 `DNO` 是 7，就会去 `DEPT` 表里找 `DNUMBER` 也是 7 的那一行，然后把它们拼在一起。
    
- **想象一下**：每一行员工数据都像一个带着插头的线，部门数据像插座。`ON` 条件就是规定只有“型号一致”的插头插座才能接在一起。
    

#### 第三步：WHERE D.LOCATION = 'Beijing' (过滤结果)

现在我们手里有一张大宽表了，每一行既有员工名，又有对应的部门地址。我们只需要在大表里筛选出地址是“北京”的行即可。

---

## 4. 拼完后的样子（逻辑草图）

**拼合前：**

- EMP表：[张三, DNO:7]、[李四, DNO:8]
    
- DEPT表：[ID:7, Beijing]、[ID:8, Shanghai]
    

**拼合（JOIN）后的大表：** 
| FNAME | DNO | DNUMBER | **LOCATION** |
| :--- | :--- | :--- | :--- |
| 张三 | 7 | 7 | **Beijing** | 
| 李四 | 8 | 8 | Shanghai |

**最后 SELECT：** 系统看到第一行是 Beijing，于是把“张三”拿出来。

# 视图（View）

## 1. 为什么要在这里用视图？

在之前的 `JOIN` 例子中，如果你想查看“北京部门的员工姓名”，你需要每次都写一遍复杂的连接逻辑：要把 `EMPLOYEE` 和 `DEPARTMENT` 连起来，还要记得关联键是 `DNO` 和 `DNUMBER`。

如果你的老板每天都要看这张表，或者你在开发 **KanForge** 时有多个页面都要显示这种“带地点的员工清单”，每次写长 SQL 既累又容易错。

---

## 2. 视图的例子：创建 `BEIJING_STAFF`

我们可以创建一个名为 `BEIJING_STAFF` 的视图，把它想象成一个**永久保存的快捷方式**。

SQL

```sql
CREATE VIEW BEIJING_STAFF AS
SELECT E.FNAME, E.LNAME, D.LOCATION
FROM EMPLOYEE AS E JOIN DEPARTMENT AS D ON E.DNO = D.DNUMBER
WHERE D.LOCATION = 'Beijing';
```

---

## 3. 创建之后怎么用？

视图创建好后，它在逻辑上就像一张真实的表一样存在于数据库中（虽然物理上它不存储数据，只是一个查询定义）。

现在，你只需要一行简单的代码就能获取结果：

SQL

```sql

SELECT * FROM BEIJING_STAFF;
```

---

## 4. 视图带给你的三大好处

- **简化查询**：你把复杂的 `JOIN` 逻辑“藏”进了视图里，以后调用时代码极简。
    
- **安全性**：假设你不想让财务看到员工的 `SALARY`（工资）列。你可以创建一个只包含 `FNAME` 和 `LNAME` 的视图给他们用，而不给他们原表的权限。
    
- **逻辑一致性**：如果哪天“北京”改名叫“雄安”了，你只需要修改视图的定义，所有引用这个视图的程序（如 KanForge 的前端）都不需要改代码。

# ORM & GORM

ORM 和 GORM 都是 **数据库操作相关的概念**，常见于后端开发。可以理解为：**ORM 是一种技术思想，而 GORM 是 Go 语言里实现 ORM 的一个框架**。

我分开给你讲清楚。

---

## 一、什么是 ORM

**ORM = Object Relational Mapping（对象关系映射）**

简单说：

> **用代码里的对象去操作数据库，而不是直接写 SQL。**

数据库是 **表结构（Table）**，  
代码是 **对象（Object / Struct / Class）**。

ORM 的作用就是：

|代码世界|数据库世界|
|---|---|
|struct / class|table|
|object|row|
|field|column|

通过 ORM，可以直接写代码操作数据库。

例如数据库表：

```
users
-------------------
id
name
age
```

在代码里可能这样定义：

```go
type User struct {
    ID   int
    Name string
    Age  int
}
```

---

### 不用 ORM（传统 SQL）

```go
db.Exec("INSERT INTO users(name,age) VALUES (?,?)", "Tom", 18)
```

---

### 用 ORM

```go
user := User{Name: "Tom", Age: 18}
db.Create(&user)
```

ORM 会自动生成 SQL：

```
INSERT INTO users(name, age) VALUES ('Tom', 18)
```

---

## 二、ORM 的优点

### 1 简化数据库操作

不用手写 SQL：

```
Create()
Find()
Delete()
Update()
```

---

### 2 面向对象

可以直接操作对象：

```
user.Name = "Alice"
db.Save(&user)
```

---

### 3 跨数据库

ORM 可以支持：

- MySQL
    
- PostgreSQL
    
- SQLite
    
- SQL Server
    

基本代码不用改。

---

### 4 自动映射

ORM 自动完成：

```
struct <-> table
字段 <-> column
```

---

## 三、ORM 的缺点

ORM 不是万能的。

### 1 性能有损耗

复杂 SQL：

```
JOIN
GROUP BY
复杂统计
```

ORM 生成 SQL 可能不够优化。

---

### 2 学习成本

很多 ORM 都有自己的：

```
API
规则
生命周期
```

---

### 3 不适合复杂查询

很多公司：

```
简单 CRUD 用 ORM
复杂查询 写原生 SQL
```

---

## 四、什么是 GORM

**GORM 是 Go 语言里最流行的 ORM 框架。**

官网：

```
https://gorm.io
```

特点：

- Go 生态最主流 ORM
    
- 支持 MySQL/Postgres/SQLite
    
- API 简洁
    
- 功能丰富
    

很多 Go Web 项目都会用。

---

## 五、GORM 示例

### 1 安装

```bash
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

---

### 2 连接数据库

```go
dsn := "user:password@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local"

db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

---

### 3 定义模型

```go
type User struct {
    ID   uint
    Name string
    Age  int
}
```

---

### 4 自动建表

```go
db.AutoMigrate(&User{})
```

会自动创建表：

```
users
```

---

### 5 插入数据

```go
user := User{Name: "Tom", Age: 20}

db.Create(&user)
```

---

### 6 查询数据

查询一条：

```go
var user User

db.First(&user, 1)
```

生成 SQL：

```
SELECT * FROM users WHERE id = 1 LIMIT 1;
```

---

查询多条：

```go
var users []User

db.Find(&users)
```

---

### 7 更新

```go
db.Model(&user).Update("Age", 21)
```

---

### 8 删除

```go
db.Delete(&user)
```

---

## 六、GORM 的高级功能

GORM 还有很多功能：

### 1 条件查询

```go
db.Where("age > ?", 18).Find(&users)
```

---

### 2 事务

```go
db.Transaction(func(tx *gorm.DB) error {

    tx.Create(&user1)
    tx.Create(&user2)

    return nil
})
```

---

### 3 关联关系

支持：

```
一对一
一对多
多对多
```

例如：

```
User
Order
```

---

### 4 Hook 生命周期

比如：

```
BeforeCreate
AfterCreate
BeforeDelete
```

---

## 七、总结

|概念|说明|
|---|---|
|ORM|一种思想：对象映射数据库|
|GORM|Go 语言实现 ORM 的框架|
|作用|用对象操作数据库|
|优点|简化 CRUD、面向对象|
|缺点|性能略低、复杂 SQL 不方便|

一句话总结：

> **ORM 是理念，GORM 是 Go 里的 ORM 实现。**

---

如果你是 **Go 后端开发**，我还可以给你讲几个非常关键的东西（很多人刚学 GORM 会踩坑）：

- GORM **结构体标签 tag**
    
- **软删除**
    
- **Preload 预加载**
    
- **N+1 查询问题**
    
- **gorm.Model 是什么**
    
- **GORM 为什么慢**
    

select 
