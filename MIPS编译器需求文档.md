---
semester: fall 2025
date: 2025-11-28
tags:
  - assignments
  - ece550k
status: true
---
我们需要针对一个MIPS ISA编写一个编译器套件（包含编译器和汇编器两个部分），用来将C代码翻译成机器码。我们准备使用C++编写这个编译器套件。我们要构建的工具链包含两个主要组件：

```
C Compiler (C → IR → Assembly) Assembler (Assembly → Machine Code)
```

你的 ISA 是一个 **简单32位RISC**，具体的ISA结构如下：

# ISA

我们使用的ISA和以往有所不同，具体如下：

| 传统MIPS ISA指令   | 我们对应使用的指令                     |
| -------------- | ----------------------------- |
| **blt, bne**   | **bgt, beq**（注意 op1 和 op2 写法） |
| **sra**（算术右移）  | **srl**（逻辑右移）                 |
| 无 input/output | **新增 input / output 指令**      |
除此之外：

- **opcode 重新编排了**
- **内存是 word-addressed**（每个地址对应 32 bit，而非 byte-addressed）

## ISA 表格（处理器支持的全部指令）

### R-type 指令（寄存器-寄存器）

| 指令  | opcode | 功能                       |
| --- | ------ | ------------------------ |
| add | 00000  | rd = rs + rt             |
| sub | 00001  | rd = rs – rt             |
| and | 00010  | rd = rs & rt             |
| or  | 00011  | rd = rs \| rt            |
| sll | 00100  | rd = rs << rt[4:0]       |
| srl | 00101  | rd = rs >> rt[4:0]（逻辑右移） |

**R-type 格式：**  

```
opcode[31:27], rd, rs, rt, 12 bits zero
```

### I-type 指令（带立即数或分支）

| 指令         | opcode | 语法             | 功能                                   |
| ---------- | ------ | -------------- | ------------------------------------ |
| addi       | 00110  | addi rd, rs, N | rd = rs + N                          |
| lw         | 00111  | lw rd, N(rs)   | rd = Mem[rs+N]                       |
| sw         | 01000  | sw rd, N(rs)   | Mem[rs+N] = rd                       |
| beq        | 01001  | beq rd, rs, N  | if rd==rs → PC = PC + 1 + N          |
| bgt        | 01010  | bgt rd, rs, N  | if rd>rs → PC = PC + 1 + N           |
| jr         | 01011  | jr rd          | PC = rd                              |
| **input**  | 01110  | input rd       | rd = keyboard_in；键盘握手 keyboard_ack=1 |
| **output** | 01111  | output rd      | 将 rd[7:0] 输出到 LCD                    |

**I-type 格式：**  

```
opcode[31:27], rd, rs, immediate[16:0]（**带符号扩展**）
```

### J-type 指令（跳转）

| 指令  | opcode | 功能                      |
| --- | ------ | ----------------------- |
| j   | 01100  | PC = target             |
| jal | 01101  | r31 = PC+1; PC = target |

**J-type 格式：**  

```
opcode[31:27], target[26:0]
```

# 注意事项
## 1. 新增 input / output 指令讲解

和以往的MIPS ISA不同，我们这次要实现的ISA有两个自定义的指令：input和output。

### `input $rd`

- 从 PS/2 键盘读取 32-bit value
- 同周期：
    - keyboard_ack = 1（只持续 1 cycle）
    - rd 写入键盘的值

### `output $rd`

- 将 $rd 的低 **8 位** 输出到 LCD
- 同周期：
    - lcd_write = 1（只持续 1 cycle）
    - lcd_data = rd

➡ 所以你打印字符时，**必须写出 ASCII 码**，比如：

```
addi $r1, $r0, 72   # 'H' 的ASCII码
output $r1
```

## 2. 内存是 word-addressed（与 MIPS 完全不同）

文档说明：

Final Project Processor Readme

> **Memory is word-addressed**  
> meaning address N refers to the N-th 32-bit word.

换句话说：

- lw/sw 的 offset 单位是 “word”（不是 byte）
- 地址 0 → 第 0 个 word
- 地址 1 → 第 1 个 word
- ……

所以 `lw r1, 4(r2)` 并不是 +4 bytes，而是 **+4 个 word = +16 byte**
这对写程序和 dmem 布局非常关键。

### 3. bgt / beq 的比较方式要注意

文档说：

Final Project Processor Readme

> operand order for bgt and beq is different from most other instructions  
> ALU gives you isLessThan, not isGreaterThan.

意思是：

```
bgt $rd, $rs, N
```

检查的是：

`if ($rd > $rs) jump`

不是你熟悉的：

```
if ($rd > $rs) jump
```

寄存器顺序非常重要！

# 目录结构

假设根目录为 `MIPS-compiler/`，建议的目录结构如下：

```
MIPS-compiler/
│
├── CMakeLists.txt
├── README.md
│
├── include/
│   ├── compiler/          # C Compiler 前端 & 中端
│   │   ├── Lexer.hpp
│   │   ├── Parser.hpp
│   │   ├── AST.hpp
│   │   ├── SemanticAnalyzer.hpp
│   │   ├── IR.hpp
│   │   ├── CodeGen.hpp    # IR → Assembly
│   │   └── SymbolTable.hpp
│   │
│   ├── assembler/         # 汇编器
│   │   ├── AsmLexer.hpp
│   │   ├── AsmParser.hpp
│   │   ├── AsmEncoder.hpp
│   │   ├── InstructionFormat.hpp
│   │   └── MIFWriter.hpp
│   │
│   ├── isa/               # 你的 ISA 定义
│   │   ├── Instructions.hpp
│   │   ├── Opcode.hpp
│   │   └── Registers.hpp
│   │
│   └── utils/
│       ├── StringUtils.hpp
│       ├── FileIO.hpp
│       └── Error.hpp
│
├── src/
│   ├── compiler/
│   │   ├── Lexer.cpp
│   │   ├── Parser.cpp
│   │   ├── AST.cpp
│   │   ├── SemanticAnalyzer.cpp
│   │   ├── IR.cpp
│   │   ├── CodeGen.cpp
│   │   └── SymbolTable.cpp
│   │
│   ├── assembler/
│   │   ├── AsmLexer.cpp
│   │   ├── AsmParser.cpp
│   │   ├── AsmEncoder.cpp
│   │   ├── InstructionFormat.cpp
│   │   └── MIFWriter.cpp
│   │
│   ├── isa/
│   │   ├── Instructions.cpp
│   │   └── Opcode.cpp
│   │
│   ├── utils/
│   │   ├── FileIO.cpp
│   │   ├── Error.cpp
│   │   └── StringUtils.cpp
│   │
│   └── main/
│       ├── compiler_main.cpp      # C → assembly
│       ├── assembler_main.cpp     # assembly → machine code
│       └── toolchain.cpp          # 融合两者：C → asm → machine code
│
├── tests/
│   ├── asm_tests/
│   │   ├── test_encoder.cpp
│   │   ├── test_lexer.cpp
│   │   └── test_parser.cpp
│   │
│   ├── compiler_tests/
│   │   ├── test_parser.cpp
│   │   ├── test_ast.cpp
│   │   ├── test_ir.cpp
│   │   └── test_codegen.cpp
│   │
│   └── programs/
│       ├── hello_world.c
│       ├── fibonacci.c
│       └── io_test.c
│
└── build/
```

# 模块职责详解（重中之重）

## 1. `isa/` —— 定义你的 RISC ISA

你需要对照 README 中的 ISA（add/sub/sll/srl/beq/bgt/input/output/j/jal/etc）：

- opcode 编码
- 寄存器表（r0–r31）
- 指令格式（R/I/J）
- 立即数范围（signed 17 bits）


例如：

### `Opcode.hpp`

```
enum class Opcode : uint8_t {
    ADD = 0b00000,
    SUB = 0b00001,
    AND = 0b00010,
    OR  = 0b00011,
    SLL = 0b00100,
    SRL = 0b00101,
    ADDI = 0b00110,
    …
    OUTPUT = 0b01111
};
```

### `InstructionFormat.hpp`

存储完整 32-bit 指令格式。



## 2. 汇编器 Assembler（assembly → machine code）

目录：`assembler/`

汇编器由三部分组成：

### AsmLexer

对 asm 做词法分析  
把字符串拆成 token：

```
add $r1, $r2, $r3
 ↓
[ADD, REG1, COMMA, REG2, COMMA, REG3]
```



### AsmParser

构建汇编语法树：

```
ADD rd rs rt
LW rd offset(rs)
BEQ rd rs label
LABEL:
```

需要处理：

- 标签收集（第一遍扫描）

- 指令编码（第二遍扫描）

- 分支偏移计算

- 符号表（标签 → 地址）




### AsmEncoder

根据 ISA 的格式生成 32-bit 指令：

- R 型编码器

- I 型编码器（带符号扩展）

- J 型编码器


### MIFWriter

生成 `imem.mif` 或 .hex 文件：

```
WIDTH=32;
DEPTH=2048;
```


## 3. C Compiler（C → lexical → parse → AST → IR → assembly）

目录：`compiler/`

### Lexer（词法分析器）

将 C 源码分成 token：

```
int a = 5;
↓
INT, IDENT(a), ASSIGN, NUMBER(5), SEMICOLON
```

### Parser（语法分析器）

产生 AST（语法树）

示例：

```c
a = a + 1;
```

AST 结构：

```
Assign
 ├── Identifier: a
 └── Add
      ├── Identifier: a
      └── Number: 1
```



### SemanticAnalyzer（语义分析）

负责：

- 变量类型检查

- 声明检查

- 作用域处理

- 常量折叠（optional）




### IR（中间代码）

你可以设计一个 **简化版的三地址指令 IR**，比如：

```
t1 = a
t2 = t1 + 1
a = t2
```

IR 便于优化（未来扩展）。

### CodeGen（后端：IR → Assembly）

此模块对照 ISA，生成最优汇编：

例如：

```c
t2 = t1 + 1
```

→

```nasm
addi $t2, $t1, 1
```

或者复杂表达式会变成树状求值。



## 4. utils（工具类）

- Error（错误处理）

- FileIO（读写文件）

- StringUtils（字符串分割、trim 等其它字符串工具函数）


## 5. main/（最终可执行工具）

- compiler_main.cpp  
    输入：test.c → 输出：test.asm

- assembler_main.cpp  
    输入：test.asm → 输出：test.hex / test.mif

- toolchain.cpp  
    完整管线：test.c → asm → machine code → 写入文件


## 6. tests/

你应该分别给：

- Lexer

- Parser

- IR

- CodeGen

- Assembler


写单元测试。

另外也应该包含示例程序（tests/programs）。

# 接口

## 1. 顶层接口设计（Compiler API）

我们将把整个 C 编译器规划成几个模块，每个模块提供一组明确的接口。

### 1.1 Compiler 顶层接口（给外部程序用）

路径：`include/compiler/Compiler.hpp`

```cpp
class Compiler {
public:
    // C → Assembly
    std::string compileToAssembly(const std::string &sourceCode);

    // C → Machine Code (binary words)
    std::vector<uint32_t> compileToMachineCode(const std::string &sourceCode);

    // C → MIF/HEX file
    void compileToMIF(const std::string &sourceCode, const std::string &outputPath);
    void compileToHex(const std::string &sourceCode, const std::string &outputPath);
};
```

## 2. 前端接口设计（C → AST）

### 2.1 Lexer API（C 源码 → Token 流）

路径：`include/compiler/Lexer.hpp`

```cpp
class Lexer {
public:
    explicit Lexer(const std::string &input);
    Token nextToken();
    bool eof() const;
};
```

### 2.2 Parser API（Token → AST）

路径：`include/compiler/Parser.hpp`

```cpp
class Parser {
public:
    explicit Parser(const std::vector<Token> &tokens);
    std::unique_ptr<ASTNode> parseProgram();
};
```

AST 结构建议采用面向对象继承：

```cpp
class ASTNode { public: virtual ~ASTNode() = default; };

class ASTBinaryOp : public ASTNode { … };
class ASTNumber   : public ASTNode { … };
class ASTVariable : public ASTNode { … };
class ASTAssign   : public ASTNode { … };
class ASTIf       : public ASTNode { … };
class ASTWhile    : public ASTNode { … };
class ASTCall     : public ASTNode { … };
```

## 3. 中端接口设计（AST → IR）

### 3.1 IR（中间表示）接口

路径：`include/compiler/IR.hpp`

我们使用 **三地址中间代码**（TAC, Three-Address Code）：

```cpp
struct IRInstruction {
    std::string op;       // "add", "sub", "mov", "call"…  
    std::string dst;
    std::string src1;
    std::string src2;
};
```

IR 程序结构：

```cpp
class IRProgram {
public:
    std::vector<IRInstruction> instructions;
    void emit(const IRInstruction &instr);
};
```

### 3.2 IR Builder（AST → IR）

路径：`include/compiler/IRBuilder.hpp`

```cpp
class IRBuilder {
public:
    IRProgram generateIR(const ASTNode *root);

private:
    std::string newTemp();        // 生成临时寄存器 t0, t1, …
    void genExpr(const ASTNode *expr, std::string &result);
    void genStmt(const ASTNode *stmt);
};
```

## 4. 后端接口设计（IR → Assembly）

### 4.1 CodeGen（IR → Assembly）

路径：`include/compiler/CodeGen.hpp`

```cpp
class CodeGen {
public:
    std::vector<std::string> generateAssembly(const IRProgram &program);

private:
    std::string allocateRegister(const std::string &var); 
    void freeRegister(const std::string &reg);
};
```

> 🚀 可未来扩展：寄存器分配器（Register Allocator）

## 5. 汇编器接口（Assembly → Machine Code → MIF/HEX）

### 5.1 汇编器词法分析器（AsmLexer）

`include/assembler/AsmLexer.hpp`

```cpp
class AsmLexer {
public:
    explicit AsmLexer(const std::string &input);
    AsmToken nextToken();
    bool eof() const;
};
```

### 5.2 汇编器语法分析（AsmParser）

负责解析：

- 标签

- 指令

- 操作符

- immediate / offset

`include/assembler/AsmParser.hpp`

```cpp
class AsmParser {
public:
    explicit AsmParser(const std::vector<AsmToken>& tokens);
    std::vector<AsmInstruction> parse();
};
```


### 5.3 汇编编码器（AsmEncoder）

负责根据 ISA 编码成 32bit 机器码（你的关键模块）

`include/assembler/AsmEncoder.hpp`

```cpp
class AsmEncoder {
public:
    uint32_t encode(const AsmInstruction &inst);
    std::vector<uint32_t> encodeProgram(const std::vector<AsmInstruction>& insts);
};
```


### 5.4 MIF / HEX 文件写入

`include/assembler/MIFWriter.hpp`

```cpp
class MIFWriter {
public:
    static void writeMIF(const std::vector<uint32_t>& words, const std::string &path);
    static void writeHex(const std::vector<uint32_t>& words, const std::string &path);
};
```


## 6. 编译器主流程伪代码（C → Machine Code）

下面是完整端到端流程的伪代码，非常贴近你之后要写的 main 函数逻辑。


### 主流程伪代码

#### `compileToMachineCode()`

```
function compileToMachineCode(sourceCode):

    # ---------- 前端：词法分析 ----------
    lexer = Lexer(sourceCode)
    tokens = []
    while not lexer.eof():
        tokens.append(lexer.nextToken())

    # ---------- 前端：语法分析 ----------
    parser = Parser(tokens)
    ast = parser.parseProgram()

    # ---------- 中端：AST → IR ----------
    irBuilder = IRBuilder()
    ir = irBuilder.generateIR(ast)

    # ---------- 后端：IR → Assembly ----------
    codegen = CodeGen()
    asmLines = codegen.generateAssembly(ir)

    # ---------- 汇编器：Assembly → Instructions ----------
    asmText = join(asmLines, "\n")
    asmLexer = AsmLexer(asmText)

    asmTokens = []
    while not asmLexer.eof():
        asmTokens.append(asmLexer.nextToken())

    asmParser = AsmParser(asmTokens)
    asmIR = asmParser.parse()

    # ---------- 汇编器：指令编码 ----------
    encoder = AsmEncoder()
    machineWords = encoder.encodeProgram(asmIR)

    return machineWords
```

## 7. `compileToMIF()` 伪代码

```
function compileToMIF(sourceCode, path):

    words = compileToMachineCode(sourceCode)

    MIFWriter.writeMIF(words, path)
```


## 8. `compileToAssembly()` 伪代码

```
function compileToAssembly(sourceCode):

    tokens = Lexer(sourceCode).tokenize()
    ast = Parser(tokens).parseProgram()
    ir = IRBuilder().generateIR(ast)
    asmLines = CodeGen().generateAssembly(ir)
    
    return join(asmLines, "\n")
```

## 9. toolchain：完整流程（CLI 工具）

```
> mipscc input.c -o -mif/-hex output.mif 
```

主程序：

```
int main() {
    Compiler c;
    c.compileToMIF(readfile("input.c"), "output.mif");
}
```
