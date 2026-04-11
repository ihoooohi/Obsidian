- **Bug 1** 🤖 subagent › "I need to find Supabase credentials and …" (210.4s) subagent is very slow, we can make optimization on parallel tool calling and sub agent for infra todos @Zane @Zuocan Ying
- **Bug 2** Amy nanobot button 
	- 不过有个小问题：大部分记录的 `bot_name` 都显示 "nanobot" 而不是具体的 bot 名字，说明 nanobot 容器的 `BOT_NAME` 环境变量可能没有正确设置。
- **Bug 3** cc难度路由问题，分级路由存在问题，回复要标注使用了什么模型（sonnet，opus）cost-quality tradeoff routing
- **Bug 4** nanobot 太慢
	- 实际耗时拆解

| 阶段       | Amy    | Shadow                  |
| -------- | ------ | ----------------------- |
| 收到消息     | 0ms    | +几ms（setImmediate）      |
| 模型思考     | ⏱ 模型耗时 | ⏱ 模型耗时                  |
| 结果展示     | 直接发卡片  | 回传 + DeepSeek 评判 + 更新卡片 |
| **额外开销** | **0**  | **回传 + 评判 = 几秒到十几秒**    |

所以即使 nanobot 模型比 Amy 快，最终展示也会慢——因为评判环节是必经之路。

