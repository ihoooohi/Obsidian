```markdown
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
└──
 build/

```



---

# ⭐ 5. 汇编器接口（Assembly → Machine Code → MIF/HEX）

## ✔ 5.1 汇编器词法分析器（AsmLexer）

`include/assembler/AsmLexer.hpp`

```cpp
class AsmLexer {
public:
    explicit AsmLexer(const std::string &input);
    AsmToken nextToken();
    bool eof() const;
};
```

---

## ✔ 5.2 汇编器语法分析（AsmParser）

负责解析：

* 标签
* 指令
* 操作符
* immediate / offset

`include/assembler/AsmParser.hpp`

```cpp
class AsmParser {
public:
    explicit AsmParser(const std::vector<AsmToken>& tokens);
    std::vector<AsmInstruction> parse();
};
```

---

## ✔ 5.3 汇编编码器（AsmEncoder）

负责根据 ISA 编码成 32bit 机器码（你的关键模块）

`include/assembler/AsmEncoder.hpp`

```cpp
class AsmEncoder {
public:
    uint32_t encode(const AsmInstruction &inst);
    std::vector<uint32_t> encodeProgram(const std::vector<AsmInstruction>& insts);
};
```

---

## ✔ 5.4 MIF / HEX 文件写入

`include/assembler/MIFWriter.hpp`

```cpp
class MIFWriter {
public:
    static void writeMIF(const std::vector<uint32_t>& words, const std::string &path);
    static void writeHex(const std::vector<uint32_t>& words, const std::string &path);
};
```

---

# ⭐ 6. 编译器主流程伪代码（C → Machine Code）

下面是完整端到端流程的伪代码，非常贴近你之后要写的 main 函数逻辑。

---

# 🚀 **主流程伪代码**

## **compileToMachineCode()**

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

---

# ⭐ 7. compileToMIF() 伪代码

```
function compileToMIF(sourceCode, path):

    words = compileToMachineCode(sourceCode)

    MIFWriter.writeMIF(words, path)
```

---

# ⭐ 8. compileToAssembly() 伪代码

```
function compileToAssembly(sourceCode):

    tokens = Lexer(sourceCode).tokenize()
    ast = Parser(tokens).parseProgram()
    ir = IRBuilder().generateIR(ast)
    asmLines = CodeGen().generateAssembly(ir)
    
    return join(asmLines, "\n")
```

---

