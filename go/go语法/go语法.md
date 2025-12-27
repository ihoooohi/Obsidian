## 一、go的类型
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
## 二、go的泛型（类似于Template）
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