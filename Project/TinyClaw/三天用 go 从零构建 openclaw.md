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

**tool 和 skill 一样都是渐进式披露的哲学**

没有 tool
![[Pasted image 20260214220138.png|500]]

有 tool
![[Pasted image 20260214220243.png|500]]

