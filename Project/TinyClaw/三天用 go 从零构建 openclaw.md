## 第一天

理清框架，做一个最小的 mvp，先不实现 bus 功能（没有 bus 功能会违反 open/close design principle）
![[209289aebe5043b4406d61a3dd5ce692.jpg|500]]

## 第二天

实现 agent 模块

在 Eino 中，要实现 Agent 主要需要两个核心部分：ChatModel 和 Tool。[https://www.cloudwego.io/zh/docs/eino/quick_start/agent_llm_with_tools/]

第一步先创建配置文件`config.go` 重要的函数是 `LoadConfig()`，使其能传递大模型所需的配置，apikey，model，baseurl

第二步创建 chatmodel，传入 config 

message --> chatmodel --> stream

```go
type Agent struct {
	chatModel *openai.ChatModel
	model string
}
```

Agent还有 model 字段是为了能够在运行时切换

第三步，给 ai 接上手脚，编写各式各样的 tool，使用 ReAct 模式

### ReAct
 
什么是 ReAct？
 
ReAct = Reasoning + Acting

![[Pasted image 20260215020319.png]]

这个过程直接封装在eino 框架的 react 库中

好的，让我从代码层面详细讲解 Tool 部分。

---

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

```go
// LLM 返回的是 JSON 格式的调用请求
{
  "name": "exec",
  "arguments": "{\"command\": \"ls -la\"}"
}
```

#### 2. 框架调用 Tool 的 Run 方法

```go
// Eino 框架内部大概是这样:
func executeTool(tool Tool, llmResponse LLMResponse) string {
    // 1. 获取参数
    arguments := llmResponse.tool_calls[0].arguments  // "{\"command\": \"ls -la\"}"
    
    // 2. 调用 Tool 的 InvokableRun 方法
    result := tool.InvokableRun(ctx, arguments)
    
    // 3. 返回结果
    return result
}
```

#### 3. 我们的 ExecTool 实际执行

```go
// tool/exec.go
func (t *ExecTool) InvokableRun(ctx context.Context, argumentsInJSON string, opts ...tool.Option) (string, error) {
    
    // 步骤1: 解析参数 JSON
    // argumentsInJSON = "{\"command\": \"ls -la\"}"
    var params struct{ Command string }
    json.Unmarshal([]byte(argumentsInJSON), &params)
    // params.Command = "ls -la"
    
    // 步骤2: 安全检查（防止危险命令）
    if guardErr := t.guardCommand(params.Command); guardErr != "" {
        return "", fmt.Errorf(guardErr)
    }
    
    // 步骤3: 真的执行 shell 命令！
    cmd := exec.CommandContext(ctx, "sh", "-c", params.Command)
    output, _ := cmd.CombinedOutput()
    // output = "total 8\ndrwxr-xr-x  3 ihoo staff  96 Feb 14 21:44 .\ndrwxr-xr-x  1 ihoo staff  96 Feb 14 21:44 ..\nagent_test.go\n"
    
    // 步骤4: 返回结果
    return string(output), nil
}
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

|方案|改动|效果|
|---|---|---|
|去重|Gateway 层加缓存|治标不治本|
|异步处理|Channel 层改成异步|从根本解决|
|消息队列|引入 Bus|更健壮，但改动大|