## 第一天

理清框架，做一个最小的 mvp，先不实现 bus 功能（没有 bus 功能会违反 open/close design principle）
![[209289aebe5043b4406d61a3dd5ce692.jpg|500]]

## 第二天

实现 agent 模块

(目前的agent我觉得更像是自动化脚本的高级版本，而不是真能自主的个体)

第一步先创建配置文件`config.go` 重要的函数是 `LoadConfig()`，使其能传递大模型所需的配置，apikey，model，baseurl



### Eino 框架介绍

`agent.go`的目的是创建出一个 ReAct Agent，能够自己推理且执行的 agent，有大脑又有手有脚的 agent，所以主要函数就是`NewAgent()`, 返回类型就用`react`框架中的`*react.agent`
```go
func NewAgent() (*react.agent, error){}
```

那么具体如何用 eino 框架中的 react 来创建出一个能ReAct的Agent呢？

在 Eino 中，要实现 Agent 主要需要两个核心部分：ChatModel 和 Tool。[https://www.cloudwego.io/zh/docs/eino/quick_start/agent_llm_with_tools/]

创建ChatModel
[https://www.cloudwego.io/zh/docs/eino/core_modules/components/chat_model_guide/]

附录：context 标准库

引入Tool

第三步，给 ai 接上手脚，编写各式各样的 tool

tool calling用的是json语言
所谓的ai的工具调用的实质是ai生成相应的json文本，程序解析这段json，把其中的字段作为参数，然后由外部程序来执行对应的函数，**ai本身没有执行能力**



深入理解eino框架中的tool
eino用接口定义工具的规范
```go
// 基础工具接口（所有工具必须实现）
type BaseTool interface {
    Info(ctx context.Context) (*schema.ToolInfo, error) // 返回工具说明书
}

// 同步工具接口（最常用）
type InvokableTool interface {
    BaseTool // 继承基础要求
    InvokableRun(ctx context.Context, argumentsInJSON string, opts ...Option) (string, error)
}

// 流式工具接口（实时返回结果）
type StreamableTool interface {
    BaseTool
    StreamableRun(ctx context.Context, argumentsInJSON string, opts ...Option) 
        (*schema.StreamReader[string], error)
}
```
AI Agent需要知道"这个工具能干嘛、要传什么参数"，全靠ToolInfo：
```go
type ToolInfo struct {
    Name        string            // 工具名称（AI调用时的标识）
    Desc        string            // 工具功能描述
    ParamsOneOf *ParamsOneOf      // 参数规则
}
```

eino 框架有三种创建 tool 的方式

手动实现
手动实现 `tool.InvokableTool`
需要手动实现`Info()`方法和`InvokableRun()`方法

半自动实现
`tool/utils.NewTool`
不用自己实现接口，需要自己手动实现 schema
一个`info := &schema.Info{}`
一个执行函数`func()`

全自动实现
`toolutils.InferTool`
输入name，desc，和一个执行函数即可，执行函数的输入参数是个结构体，这个结构体只要绑定 json 的 tag，就能被自动推断

utils.InferTool 的工作原理 ：

- 自动从 execCommand 函数的参数（ execInput 结构体）推断出参数schema
- 生成JSON Schema供LLM理解tool的参数格式
- 返回一个实现了 tool.InvokableTool 接口的tool实例

使用`toolutils.InferTool`来创建tool


## Tool 核心概念

### 一、Tool 的本质 = 给 LLM 用的"函数"

```
传统函数定义:
┌─────────────────────────────────────┐
│  func exec(command string) string  │
│                                     │
│  - 函数名: exec                    │
│  - 参数: command (string)          │
│  - 返回值: string                 │
│  - 实现: 执行 shell 命令           │
└─────────────────────────────────────┘

LLM Tool 定义:
┌─────────────────────────────────────┐
│  Name: "exec"                      │
│  Description: "执行 shell 命令"    │
│  Parameters: {                      │
│    command: {                       │
│      type: "string",               │
│      required: true                │
│    }                               │
│  }                                 │
│  Run(): 执行并返回结果              │
└─────────────────────────────────────┘
```

---

### 二、Tool 的三个要素

让我结合代码来看：

#### 1. Name（名称）

```go
// tool/exec.go
func (t *ExecTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
    return &schema.ToolInfo{
        Name: "exec",  // ← 工具名称
    }
}
```

**作用**：让 LLM 知道"这个工具叫什么名字"

**LLM 收到的信息**：
```
可用工具:
- exec
```

---

#### 2. Description（描述）

```go
// tool/exec.go
func (t *ExecTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
    return &schema.ToolInfo{
        Name: "exec",
        Desc: "执行 shell 命令并返回输出结果。警告：此工具具有危险性，请仅在确认安全的情况下使用。",
    }
}
```

**作用**：让 LLM 理解"什么情况下应该用这个工具"

**LLM 收到的信息**：
```
工具说明:
- exec: 执行 shell 命令并返回输出结果
```

---

#### 3. Parameters（参数定义）

```go
// tool/exec.go
func (t *ExecTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
    return &schema.ToolInfo{
        Name: "exec",
        Desc: "执行 shell 命令...",
        ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
            "command": {           // ← 参数名
                Desc:     "要执行的 shell 命令",  // ← 参数说明
                Type:     schema.String,          // ← 参数类型
                Required: true,                   // ← 是否必需
            },
        }),
    }
}
```

**作用**：让 LLM 知道"怎么传参数"

**LLM 收到的信息**：
```
参数:
- command: string (必需) - 要执行的 shell 命令
```

---

### 三、LLM 如何知道调用哪个 Tool？

这是一个关键问题：**LLM 并不真正"理解"代码，它是根据描述来决定的！**

#### 发送给 LLM 的完整内容

```go
// 当用户说 "列出文件" 时，Eino 框架会自动构建这样的消息：

messages := []Message{
    {
        Role: "user",
        Content: "列出当前目录的文件",
    },
    {
        Role: "system",
        Content: `
可用工具:
1. exec - 执行 shell 命令
   参数: command (string, 必需)

你是一个助手。
        `,
    },
}

// 发送给 LLM
```

#### LLM 的"思考"过程（简化版）

```
用户: "列出当前目录的文件"

LLM 分析:
1. 用户想查看文件列表
2. 我知道有个叫 "exec" 的工具可以执行 shell 命令
3. "ls -la" 命令可以列出文件
4. 我应该调用 exec 工具，参数是 command="ls -la"

LLM 返回:
{
  tool_calls: [
    {
      name: "exec",
      arguments: {"command": "ls -la"}
    }
  ]
}
```

---

### 四、Tool 的执行过程

#### 1. Eino 框架解析 LLM 响应

```text
========== JSON Schema：给AI看的说明书 ==========

你的Go结构体：
type execInput struct {
    Command string `json:"command"`
}

转换成JSON Schema后：
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "要执行的shell命令"
    }
  },
  "required": [
    "command"
  ]
}

========== AI看到这个Schema后 ==========
AI就知道：
  - 这个工具需要一个JSON对象
  - 对象里有一个字段叫 command
  - command 是字符串类型
  - command 是必填的

所以AI会生成这样的调用：
{
  "name": "exec",
  "arguments": {
    "command": "ls -la"
  }
}
```
```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    Tool调用的完整流程                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  第一步：注册工具时（程序启动时）                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  你的代码：                                                        │ │
│  │  type execInput struct {                                          │ │
│  │      Command string `json:"command"`                              │ │
│  │  }                                                                │ │
│  │  func execCommand(ctx, input execInput) (string, error)           │ │
│  │                                                                   │ │
│  │  InferTool 做的事：                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  把上面的结构体 → 转成 JSON Schema                            │ │ │
│  │  │                                                             │ │ │
│  │  │  {                                                          │ │ │
│  │  │    "name": "exec",                                          │ │ │
│  │  │    "description": "执行shell命令",                           │ │ │
│  │  │    "parameters": {                                          │ │ │
│  │  │      "type": "object",                                      │ │ │
│  │  │      "properties": {                                        │ │ │
│  │  │        "command": {"type": "string"}                        │ │ │
│  │  │      }                                                      │ │ │
│  │  │    }                                                        │ │ │
│  │  │  }                                                          │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                    │
│  第二步：发送给AI（每次对话时）                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  发给AI的内容：                                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  1. 用户的提问："帮我看看当前目录有什么文件"                    │ │ │
│  │  │  2. 可用的工具列表（上面的JSON Schema）                        │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                    │
│  第三步：AI思考并返回（AI的输出）                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  AI的思考过程：                                                    │ │
│  │  "用户想看目录文件 → 我有个exec工具 → 我应该调用它"                  │ │
│  │                                                                   │ │
│  │  AI返回的JSON：                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  {                                                          │ │ │
│  │  │    "tool_calls": [{                                         │ │ │
│  │  │      "name": "exec",                                        │ │ │
│  │  │      "arguments": "{\"command\": \"ls -la\"}"               │ │ │
│  │  │    }]                                                       │ │ │
│  │  │  }                                                          │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  注意：AI只输出JSON文本，它不会真正执行命令！                        │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                    │
│  第四步：程序执行（你的代码真正运行）                                       │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  Eino框架做的事：                                                  │ │
│  │  1. 解析AI返回的JSON                                              │ │
│  │  2. 提取 arguments: {"command": "ls -la"}                        │ │
│  │  3. 调用你写好的函数：execCommand(ctx, input)                      │ │
│  │                                                                   │ │
│  │  你写的函数真正执行：                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  func execCommand(ctx, input execInput) (string, error) {   │ │ │
│  │  │      // 这里真正执行 shell 命令！                             │ │ │
│  │  │      cmd := exec.Command("sh", "-c", input.Command)         │ │ │
│  │  │      output, _ := cmd.CombinedOutput()                      │ │ │
│  │  │      return string(output), nil                             │ │ │
│  │  │  }                                                          │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  返回结果：                                                       │ │
│  │  "total 24\ndrwxr-xr-x  5 user ...\n..."                         │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                    │
│  第五步：结果返回给AI                                                     │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  把执行结果发给AI：                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  {                                                          │ │ │
│  │  │    "role": "tool",                                          │ │ │
│  │  │    "content": "total 24\ndrwxr-xr-x  5 user ...\n..."       │ │ │
│  │  │  }                                                          │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                    │
│  第六步：AI生成最终回复                                                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  AI看到执行结果后，生成给用户的回复：                                │ │
│  │                                                                   │ │
│  │  "当前目录下有以下内容：                                            │ │
│  │   - main.go                                                       │ │
│  │   - go.mod                                                        │ │
│  │   - internal/ 目录                                                 │ │
│  │   ..."                                                            │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```


---

### 五、完整流程图解

```
用户: "列出当前目录文件"
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Eino 框架构建消息                                      │
│  (包含 Tool 描述)                                       │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  LLM 分析                                               │
│  "用户要查看文件 → 用 exec 工具 → 参数 ls -la"          │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  LLM 返回 Tool Call                                    │
│  {name: "exec", arguments: {"command": "ls -la"}}     │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Eino 框架解析并调用 ExecTool.InvokableRun()           │
│  参数: argumentsInJSON = "{\"command\": \"ls -la\"}"   │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  ExecTool.InvokableRun() 内部:                         │
│  1. json.Unmarshal → params.Command = "ls -la"         │
│  2. 安全检查 → 通过                                     │
│  3. exec.Command("sh", "-c", "ls -la") → 执行!       │
│  4. 返回 output = "total 8\nagent_test.go\n"           │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Eino 把 Tool 结果发给 LLM                             │
│  LLM 生成最终回复                                       │
└─────────────────────────────────────────────────────────┘
         │
         ▼
   AI: "当前目录有 1 个文件: agent_test.go"
```


**tool 和 skill 一样都是渐进式披露的哲学**

没有 tool
![[Pasted image 20260214220138.png|500]]

有 tool
![[Pasted image 20260214220243.png|500]]

### 什么是 ReAct Agent

ReAct = Reasoning + Acting

是一个循环决策系统，崇尚一步一步调用工具

例如，用户问我要出门了，根据天气我该怎么穿

通常模型会输出这种结构：

```text
Thought: 我需要先查天气  
Action: get_weather  
Action Input: {"city": "北京"}
```

程序解析：

- 发现要调用 get_weather
- 执行
- 得到结果：{"temp": 5}
    

然后把结果喂回模型：

```text
Observation: {"temp": 5}
```

模型继续：

```text
Thought: 气温 5 度，比较冷，我应该建议穿外套  
Final Answer: 今天北京 5 度，建议穿外套。
```

![[Pasted image 20260215020319.png]]

这个过程直接封装在eino 框架的 react 库中
## 第三天

channel 模块

连接飞书

```text
Step 1: 定义接口
┌─────────────────────────────────────────┐
│  channel.go: Channel 接口 + 消息结构     │
│  base.go: BaseChannel + 白名单逻辑       │
└─────────────────────────────────────────┘

Step 2: 实现飞书
┌─────────────────────────────────────────┐
│  feishu.go: 使用官方 SDK                 │
│  - WebSocket 模式（无需公网 IP）          │
│  - 只处理文本消息（MVP）                  │
└─────────────────────────────────────────┘

Step 3: 简单测试
┌─────────────────────────────────────────┐
│  main.go: 直接启动 FeishuChannel         │
│  收到消息 → 打印到控制台                  │
│  验证飞书能收消息                         │
└─────────────────────────────────────────┘
```

### 扩展性考虑

|扩展点|设计|
|---|---|
|新增 Channel|实现 Channel 接口即可|
|消息类型|Metadata 存储额外信息，后续可扩展图片等|
|白名单|BaseChannel.IsAllowed() 统一处理|
|多 Channel|后续用 Bus 统一管理|

等有第二个 Channel 时再抽象 BaseChannel


gateway 模块

作用是将 channel 的信息返回给 agent，这部分需要单独解耦，如果跟 agent 交互的内容写在 channel 中那将是灾难，以后每加一个 channel 都要重新写，所以要用 gateway 模块解耦


```text
┌─────────────────────────────────────────────────────────────┐
│                    Step 4 完整流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   飞书用户: "查看当前目录"                                   │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────┐                                          │
│   │ Feishu      │                                          │
│   │ Channel     │ ──▶ InboundMessage                       │
│   └─────────────┘          │                               │
│                             ▼                               │
│                      ┌─────────────┐                       │
│                      │  Gateway    │                       │
│                      │ OnMessage() │                       │
│                      └──────┬──────┘                       │
│                             │                               │
│          ┌──────────────────┼──────────────────┐          │
│          ▼                  ▼                  ▼          │
│     1. 构建消息       2. 调用 Agent      3. 发送回复       │
│                             │                               │
│                             ▼                               │
│                      ┌─────────────┐                       │
│                      │   Agent     │                       │
│                      │  Generate() │                       │
│                      └─────────────┘                       │
│                             │                               │
│                             ▼                               │
│                      exec tool 执行                         │
│                      "ls -la"                               │
│                             │                               │
│                             ▼                               │
│                      Agent 生成回复                         │
│                      "当前目录有 5 个文件..."                │
│                             │                               │
│                             ▼                               │
│                      ┌─────────────┐                       │
│                      │ Feishu      │                       │
│                      │ Send()      │                       │
│                      └─────────────┘                       │
│                             │                               │
│                             ▼                               │
│   飞书用户收到: "当前目录有 5 个文件..."                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```go
type Gateway struct {
	agent *react.Agent
	sender Sender
}
```
而不是
```go
type Gateway struct {
	agent *react.Agent
	channel Channel
}
```

是为了后续 bus 的拓展

## 遇到的bug

1.回复的消息重复发送两次
2.之前发过的消息已经回复过了，过了一会又给我回复一遍

| 方案   | 改动            | 效果       |
| ---- | ------------- | -------- |
| 去重   | Gateway 层加缓存  | 治标不治本    |
| 异步处理 | Channel 层改成异步 | 从根本解决    |
| 消息队列 | 引入 Bus        | 更健壮，但改动大 |
```go
// feishu.go 第 197-207 行
func (c *FeishuChannel) handleMessageReceive(_ context.Context, event *larkim.P2MessageReceiveV1) error {
    // ... 解析消息 ...
    
    if c.handler != nil {
        c.handler(context.Background(), InboundMessage{...})  // ← 同步阻塞！
    }
    
    return nil  // ← ACK 在这里才发送，太晚了！
}
```

```go
func (c *FeishuChannel) handleMessageReceive(_ context.Context, event *larkim.P2MessageReceiveV1) error {
    // ... 解析消息 ...
    
    if c.handler != nil {
        go c.handler(context.Background(), InboundMessage{...})  // ← 异步！不阻塞！
    }
    
    return nil  // ← 立即返回，快速 ACK
}
```

```text
┌─────────────────────────────────────────────────────────────┐
│                    消息处理流程                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   阶段 1：接收消息                                          │
│   ┌─────────────────────────────────────┐                  │
│   │ 飞书推送 → 服务收到 → 返回 ACK       │  ← 快速！毫秒级  │
│   └─────────────────────────────────────┘                  │
│                                                             │
│   阶段 2：处理消息                                          │
│   ┌─────────────────────────────────────┐                  │
│   │ 调用 Agent → 生成回复 → 发送给用户  │  ← 慢！秒级      │
│   └─────────────────────────────────────┘                  │
│                                                             │
│   ACK 只关心阶段 1，不关心阶段 2                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

并发安全问题
```go
// 当前代码
if c.handler != nil {
    go c.handler(context.Background(), InboundMessage{...})  // 无限制并发！
}
```

大量消息同时到达，启动大量 goroutine，可能导致：

- 内存耗尽
- CPU 100%
- 服务崩溃

可以通过控制并发量来解决，如
```go
// 方案 A：简单信号量（不用 Bus）
var sem = make(chan struct{}, 3)  // 最多 3 个并发

func (c *FeishuChannel) handleMessageReceive(...) error {
    if c.handler != nil {
        go func() {
            sem <- struct{}{}        // 获取信号量
            defer func() { <-sem }() // 释放信号量
            c.handler(...)
        }()
    }
    return nil
}
```

但是这样就很死板，限制死了并发数量，引入 bus 可以解决这个问题

![[Pasted image 20260221142412.png]]

