## go的类型
### 1. 第一级：基础类型 (Basic Types)

这些是语言“出厂自带”的，是最底层的原子单位。它们直接对应计算机内存中的数字或布尔值。

- **数字**：`int`, `float64`, `uint32`, `byte` 等。
    
- **文本**：`string`。
    
- **逻辑**：`bool`。
    

**特点**：它们内部没有字段（不像 struct），也没有方法列表（不像 interface）。它们就是纯粹的数据。

---

### 2. 第二级：复合类型 (Composite Types) —— 比如 Struct

这是你用基础类型“拼装”出来的复杂零件。

- **Struct**：把你关心的变量打包在一起。
    
    Go
    
    ```
    type Player struct {
        Score int    // 这里的 Score 是 int 类型
        Name  string
    }
    ```
    

---

### 3. 第三级：抽象类型 (Abstract Types) —— 比如 Interface

这是你定义的“规格”或“协议”。

- **Interface**：不关心数据，只关心能不能干活。
    
    Go
    
    ```
    type Clarifier interface {
        Clear() // 只要有 Clear 方法就行
    }
    ```
    
