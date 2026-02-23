## 一、类型
### 1. 第一级：基础类型 (Basic Types)

这些是语言“出厂自带”的，是最底层的原子单位。它们直接对应计算机内存中的数字或布尔值。

- **数字**：`int`, `float64`, `uint32`, `byte` 等。
    
- **文本**：`string`。
    
- **逻辑**：`bool`。
    
---

### 2. 第二级：复合类型 (Composite Types) —— 比如 Struct

这是你用基础类型“拼装”出来的复杂零件。

- **Struct**：把你关心的变量打包在一起。
    ```go
    type Player struct {
        Score int    // 这里的 Score 是 int 类型
        Name  string
    }
    ```
    

---

### 3. 第三级：抽象类型 (Abstract Types) —— 比如 Interface

这是你定义的“规格”或“协议”。

- **Interface**：不关心数据，只关心能不能干活。
    ```go
    type Clarifier interface {
        Clear() // 只要有 Clear 方法就行
    }
    ```
    
### 4. 第四级：函数类型 
    
```go
// 定义一个名为 Op 的函数类型 
type Op func(a int, b int) int 
func Calculate(x, y int, action Op) int { return action(x, y) // 像调用普通函数一样调用它 
}
```
    

---

### 5. 第五级：引用/容器类型 (Reference Types) —— “隐形指针”

这些类型看起来像普通变量，但它们内部其实包含指向底层数据的指针。直接传递它们时，修改的是同一份数据。

- **Slice (切片)**：动态数组的“窗口”。
    
- **Map (映射)**：哈希表，内置的键值对容器。
    
- **Channel (通道)**：Goroutine 之间通信的管道。
    
- **Pointer (指针)**：直接存储内存地址。
## 二、泛型（类似于Template）
### 关键字
`[T any]` 类似于 `template <typename T>`
### 例子
```go
// T 是类型参数，any 表示支持任何类型
func Swap[T any](a, b T) (T, T) {
    return b, a
}

func main() {
    // 交换整数
    i1, i2 := Swap[int](10, 20)
    fmt.Println(i1, i2) // 输出: 20 10

    // 交换字符串 (Go 会自动推导类型，可以省略 [string])
    s1, s2 := Swap("World", "Hello")
    fmt.Println(s1, s2) // 输出: Hello World
}
```


## 三、字面量赋值

**字面量赋值的核心公式：** `变量 := [&]类型{ 字段: 值, 字段: 值 }`

- **有 `&`**：得到的是**指针**（适合 LRU Cache 这种需要被修改的大对象）。
    
- **无 `&`**：得到的是**值**（适合 Point, Color 等简单的小数据）。
```go
c := &Cache{
	maxBytes: 100
	ll: list.New()
}
```
## 四、具名返回值

### 1. 文档自注释（提高可读性）

当一个函数返回多个**类型相同**的变量时，调用者很难一眼看出谁是谁。具名返回值就像给返回值贴上了“标签”。

- **普通写法**：`func Size() (int, int, int)` —— 谁是长？谁是宽？谁是高？
    
- **具名写法**：
    ```go
    func Size() (width int, height int, depth int) {
        return 10, 20, 30
    }
    ```
    

这样在生成的 API 文档中，用户一眼就能明白每个数字代表的含义。

---

### 2. 在 `defer` 中修改返回值（高级神技）

这是具名返回值不可替代的场景。

由于 defer 函数在 return 语句执行之后、函数彻底退出之前执行，只有具名返回值才能在 defer 中被捕获并修改。

**经典案例：捕获并修改错误**

```go
func DoSomething() (err error) {
    defer func() {
        if r := recover(); r != nil {
            // 如果程序崩溃了，我们在 defer 里把错误“救”回来
            // 只有具名变量 err 才能在这里被重新赋值
            err = fmt.Errorf("panic trapped: %v", r)
        }
    }()
    
    // ... 触发了 panic ...
    return nil
}
```

如果没有给 `error` 起名字，`defer` 里的匿名函数是无法触碰到即将返回的那个变量的。

---

### 3. 简化复杂的初始化逻辑

在一些逻辑非常复杂的长函数中，你可能在很多地方都要返回。如果使用具名返回值，你可以先给它们赋好默认值，然后在不同的 `if/else` 分支里直接写 `return`。

```go

func FindUser(id string) (user *User, err error) {
    // 默认值已经初始化为 nil, nil 了
    if id == "" {
        err = ErrInvalidID // 直接改写具名变量
        return             // 自动返回 (nil, ErrInvalidID)
    }
    
    user = db.Query(id)
    if user == nil {
        err = ErrNotFound
        return             // 自动返回 (nil, ErrNotFound)
    }
    
    return // 自动返回 (user, nil)
}
```

---

### 4. 补充笔记：具名返回值的“红黑榜”

你可以把这个总结进你的笔记，用来提醒自己什么时候该用它：

| **推荐使用（红榜）**            | **避免使用（黑榜）**              |
| ----------------------- | ------------------------- |
| 返回多个同类型变量（做文档说明）        | 函数逻辑很短、很直观（如你的 `Get`）     |
| 需要在 `defer` 中修改或记录返回值   | 存在变量遮蔽（Shadowing）风险时      |
| 需要在 `defer` 中捕获 `panic` | 会导致“裸返回” (`return`) 含义不明时 |
## 五、大小写规则

**是否能被包外访问，只取决于：名字首字母是否大写**

**🔹 大写 → 导出（public）**
🔹 **小写 → 未导出（private，包内可见）**

## 六、函数类型 & 接口类型 & 接口型函数

总的来说

**函数类型** ---> 抽象单个行为
**接口类型** ---> 抽象多个行为
**接口型函数** ---> 用函数快速进入接口世界（不用struct）
### 1、为什么有函数类型？

为什么要设计函数类型的变量？----**把“行为”抽象成数据，从而避免重复流程代码**

不用函数类型:
```go
func DoEat() {
    fmt.Println("开始")
    fmt.Println("吃饭")
    fmt.Println("结束")
}

func DoSleep() {
    fmt.Println("开始")
    fmt.Println("睡觉")
    fmt.Println("结束")
}

func DoStudy() {
    fmt.Println("开始")
    fmt.Println("学习")
    fmt.Println("结束")
}

```
用函数类型：

```go
func Do(action func()) {
	fmt.Println("开始")
	action()
	fmt.Println("结束")
}



func main() {
	Do(func() {
		fmt.Println("吃饭")
	})
	
	Do(func() {
		fmt.Println("睡觉")
	})
	
	Do(func() {
		fmt.Println("学习")
	})
}
```
### 2、为什么有接口类型？

知道为什么有函数类型后，我们会发现如果只有函数类型还是不够，函数类型变量只能抽象**单个行为**，不能抽象**一组行为**，于是接口诞生了

|     | 单个             | 一组        |
| --- | -------------- | --------- |
| 数据  | int/string/... | struct    |
| 行为  | func()         | interface |

**接口类型的变量，最常见、最重要的用途，就是作为函数参数**

```go
type Human interface {
	Walk()
	Speak()
}

func WalkSpeak(h Human) {
	h.Walk()
	h.Speak()
}

type American struct {}
func (w American) Walk() {
	fmt.Println("da")
}
func (w American) Speak() {
	fmt.Println("English")
}

type Chinese struct {}
func (w Chinese) Walk() {
	fmt.Println("dada")
}
func (w Chinese) Speak() {
	fmt.Println("Chinese")
}

func main() {
	var a American
	var c Chinese
	
	WalkSpeak(a)
	WalkSpeak(c)
}
```
### 3、为什么要有接口型函数？

因为参照上面的代码，我想实现 `WalkSpeak()` 一定要先弄个struct，这有点麻烦，我只是临时一次性的话能不能不要每次都搞个struct呢？于是接口型函数诞生了

接口型函数特点就是可以**不用struct**，适合**临时**

比如说将上面的代码改成接口型函数

```go
type Human interface {
	Walk()
	
}

func Walk(h Human) {
	h.Walk()
}

type HumanFunc func()

func (h HumanFunc) Walk() {
	h()
}

//使用
func main() {
	Walk(HumanFunc(func(){
		fmt.Println("da")
	}))
}
```

### 4、⭐接口型函数

什么是接口型函数？就是用函数**类型**实现接口

❗函数与函数类型是两个概念
- 函数 `func GetterFunc() () {}`
- 函数类型 `type GetterFunc func() ()`
❗只有**类型**才能实现接口，所以只有函数类型能实现接口

三部曲：
1. 写接口
```go
type Greeter interface {
	Greet() string
}
```
2. 写函数类型（**等价于要实现接口方法的struct**）
```go
type GreeterFunc func() string
```
3. 函数类型实现接口方法，**在方法内部调用自身（函数值
4. ）**
```go
func (g GreeterFunc) Greet() string {
	return g()
}
```

使用接口型函数（本质就是替换接口参数）
1. 先定义一个接口变量
```go
var g Greeter
```
2. 再将一个匿名函数转成接口函数类型，再赋值
```go
g = GreeterFunc(func() string {
	return "hello"
})
```
3. 调用Greet()
```go
g.Greet()
```


## 七、...的用法

### 1.可以表示接收很多参数

`func f(v ...T)`

### 2. 可以表示将接收的参数展开

`f(v...)`

## 八、锁（Mutex）与 等待（Wait）
### 1. 锁（Mutex）

### 1.概念

go的**进程内锁**在`sync`包中，包括**互斥锁**（`Mutex`）和**读写锁**（`RWMutex`）

**①互斥锁**（`Mutex`）:

一次只能一个goroutine，锁期间不能读不能写

**②读写锁**（`RWMutex`）

RWMutex有两把锁，**写锁**（`Lock()`）和 **读锁**（`RLock()`）

写锁锁后，只能一个goroutine写
```go
var mu sync.RWMutex

mu.Lock()
data = 32
mu.UnLock()
```
读锁锁后，多个goroutine能读

```go
var mu sync.RWMutex

mu.RLock()
v := data
mu.RUnlock()
```

读锁是共享的，写锁必需独占

go中写优先，必须写锁释放读锁才能释放；同时必须读锁释放写锁才能拿到
```
G1: RLock()  ─────── RUnlock()
G2: RLock()     ─────── RUnlock()
G3: Lock()           ─────── Unlock()
G4: RLock()                （阻塞）

```
G1，G2读，读锁释放后，G3写锁，G4读锁被阻塞不能读，G3写锁释放后，G4拿到读锁

### 2.使用场景

| 使用场景       | 建议      |
| ---------- | ------- |
| 通常场景       | Mutex   |
| 写多         | Mutex   |
| 临界区极小      | Mutex   |
| 读比写多，且读很耗时 | RWMutex |


### 2. 等待（Wait）

Go 没有一个叫 `Wait` 的万能东西，而是**针对不同语义，给了不同工具**。
### 1️⃣ 等“一组 goroutine 全部完成”

👉 `sync.WaitGroup`

### 2️⃣ 等“某个条件成立”

👉 `sync.Cond`

### 3️⃣ 等“某个结果出现”

👉 `channel`

---

从`WaitGroup` 开始讲起

`wg.Add(n)`：初始化等待的协程个数，计数器

`wg.Done()`：相当于`wg.Add(-1) `

`wg.Wait()`： 阻塞当前goroutine（通常是main），等待器

```go
wg.Add(3)

for i := 0; i < 3; i++ {
    go func(i int) {
        defer wg.Done()
        fmt.Println(i)
    }(i)
}

wg.Wait()
fmt.Println("main结束")
```

必须等计数器为0，才会通过wg.Wait()的阻塞，执行后面语句，结果：

```text
1
0
2
main 结束

```
1、0、2的顺序是随机的，但是  **“main 结束”一定在他们后面**

### 3. 锁与等待共同使用

**原则**：等待必须在锁外

```go
group.Lock()
//code
group.UnLock()

call.wg.Wait()
```

**原因**：
![[images/53254281f1ede6fbcd8700e21d564875.png]]

如果等待在锁内，那等待就只能挡住一条，没意义

而且如果等待的goroutine等待的是锁外的主goroutine就会**死锁**

![[images/a3b604cc2b5a0c6f03b5b4557ee88d46.jpg]]

g2被锁挡住永远执行不了，而解锁的条件是g1执行，但是g1必须等待g2执行，所以**死锁**

**总结**：有**阻塞**（如wait（）和Done（））还要**慢操作**（等待中的操作fn（））要放锁外，Add（）可以放锁内

```go
//可以锁内，为了原子性
c.wg.Add(1)

//必须锁外
c.value = fn()
c.wg.DOne()
```

## 九、go相关八股
### slice的底层实现

#### 1. slice 本身
slice不是一种具体的数据结构，而是底层数组的**抽象视图**

slice由三部分组成：

| **地址指针** | **长度** | **长度** |
| -------- | ------ | ------ |

```go
type slice struct {
	array unsafe.Pointer // 指向底层数组的起始地址
	len int // 当前切片长度
	cap int // 底层数组容量
}
```
数组与切片（slice）
```go
a := [5]int{1,2,3,4,5} //初始化数组
b := a[1:4] //左闭右开
```
![[images/e26030142679518b84e74370575db0bc.jpg]]

```go
len = high - low
cap = cap(base) - low //eg. 5 - 1
```
#### 2.slice的扩容机制

slice的扩容就是找到一个空间充足的连续内存空间，把原来的slice的内容复制过去，这个阶段，slice的**指向底层数组的指针改变了**

**Go 的 slice 扩容规则是：**

- 当 **原 cap 较小（< 256）** 时，**新 cap = 原 cap × 2**
    
- 当 **原 cap 较大（≥ 256）** 时，**新 cap 按 ~1.25 倍逐步增长**
    
- 如果 **需要的最小容量 > 2×原 cap**，则 **直接扩到需要的最小容量**

#### 3.slice的常见陷阱（陷阱相关）
##### ❌ 1. 误以为 append 不会影响原 slice

```go
a := []int{1, 2, 3}
b := a[:2]
b = append(b, 100)

fmt.Println(a) // a被修改！a: [1,2,100]

```
原因：容量cap = 3够用，没有触发扩容机制，append 直接写入原数组。
![[images/749a374e565ed3a14fe389b92674878d.jpg]]
若是
```go
a := []int{1, 2, 3}
b := a[:3]
b = append(b, 100)

fmt.Println(a) // a不会被修改！a: [1,2,3]
```
原因：容量cap = 3不够用，触发扩容机制，b的底层数组不再是a。![[images/d4cfb5be04ecad9200a52c0c25b1e02b.jpg]]

---

##### ❌ 2. slice 作为函数参数是“引用传递”？

```go
func f(s []int) {
    s = append(s, 10)
}
```

- slice **本身是值传递**
    
- 但它内部包含指针
    
- append 后是否影响外部，取决于是否扩容


### map的底层实现

#### map本身（hmap）

go中的map是一个哈希表（hash table），结构体名为 `type hmap struct` 
```go
type hmap struct {
	buckets *unsafe.Pointer
}
```
![[images/9885c837c68deef4635c960517a4c721.jpg|450]]

注意⚠️：
1. tophash是什么
		存放每个key的hash后的值的高8位，用来快速比较（**快速预过滤**）
2. key和value不是交错放，而是key和value**分别连续存放**
		**原因**：减少 padding（无用字节）
		- 例如`map[int64]int8`
		如果交错存（key,val,key,val）：
		- `int64` 需要 8 字节对齐
		- `int8` 只有 1 字节
		- 但为了对齐，value 后面可能要**填很多 padding（无用字节）**
		- 结果 bucket 变大，cache 命中率变差，性能变差
3. overflow是如果这一个bucket装满了，则用指针链接一个新的bucket（类似**链地址法**）

#### map的扩容机制

map 有两种情况需要扩容:
1. kv的数量 / bucket 的数量 太多了（> 6.5）
2. 一个 bucket 中 overflow 太多了（哈希冲突太多了）

##### 针对情况 1，解决办法：

---- **翻倍增加 bucket 数量**

**问题 1：** 但是这样有一个问题，如果 map 已经很大了，一次性迁移数据太慢了，很浪费时间

于是go 的解决办法是 ---- **渐进式 rehash**

即不是一次性搬完，而是同时保留 **旧 bucket** 的指针和 **新 bucket 的指针**

```go
type hmap struct {
...
buckets unsafe.Pointer
oldbuckets unsafe.Pointer
...
}
```

每次读前或写前，先搬迁 1,2 个 bucket

**问题 2：** key 的 hash 值对应的桶编号是怎么计算的？

用 `B` 来计算，`B` = log2（桶数）

```go
type hmap struct {
...
B unint8
buckets unsafe.Pointer
oldbuckets unsafe.Pointer
...
}
```

**扩容前：**
```text
桶数：2^B
B: B
桶号：hash 的低 B 位
```

**扩容后：**
```text
桶数：2^(B+1)
B: B+1
桶号：hash 的低 B+1 位
```

hash 值是不变的，假设 key0 的哈希值是 001110， key1 的哈希值是 001011，扩容前B 是 2，扩容后 B 是 3

则 key0 桶号：10(2) ---> 110(2+4=6)
则 key1 桶号：11(3) ---> 011(3)

新多出来的这一位，0 --> 还在原桶，1 --> 去新的桶的同样相对位置

所以扩容后 key：
- 要么还在原桶        
- 要么去 原桶 + 原桶数

![[images/a5f5de62fcaa6ea6cdc108d2bf1d456d.jpg]]
##### 针对情况2，解决办法：

---- **等量扩容（same-size-grow）**

也就是不改变 bucket 的大小，新开辟一块**同样大小**的内存，把数据依然是 **渐进式** 地迁移过去（buckets 指向新内存， oldbuckets 指向原内存），然后让 kv 的分布更均匀

这就有一个问题，新的 buckets 怎么就保证分布更均匀了而不是和旧的一样

答案是渐进式搬迁中，用**新的哈希扰动/种子（hash seed）** 重新计算 key 的哈希结果


### go的垃圾回收（GC）

#### 什么是go 的垃圾回收

自动回收**不可达**对象

什么是不可达对象？

```go
d1 := &Document{name: "A"}
d1 := &Document{name: "B"}
```

d1从指向 A 变成指向 B 了，B 不再可达，所以 B 是不可达对象，这是 GC 需要回收的对象
### go的逃逸分析


## channel

无缓冲需同步配对，有缓冲可暂存

例：

```go
// 无缓冲（会死锁）
ch := make(chan int)
ch <- 1
<-ch
```

```go
// 有缓冲（不会死锁）
ch := make(chan int, 1)
ch <- 1
<-ch
```