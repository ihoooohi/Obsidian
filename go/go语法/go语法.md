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
    
### 4. 第四级：逻辑类型 (Functional Types) —— 函数也是一种变量
    
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

|**推荐使用（红榜）**|**避免使用（黑榜）**|
|---|---|
|返回多个同类型变量（做文档说明）|函数逻辑很短、很直观（如你的 `Get`）|
|需要在 `defer` 中修改或记录返回值|存在变量遮蔽（Shadowing）风险时|
|需要在 `defer` 中捕获 `panic`|会导致“裸返回” (`return`) 含义不明时|
