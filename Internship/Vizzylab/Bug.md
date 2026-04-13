- **Bug 1** 🤖 subagent › "I need to find Supabase credentials and …" (210.4s) subagent is very slow, we can make optimization on parallel tool calling and sub agent for infra todos @Zane @Zuocan Ying
- **Bug 2** Amy nanobot button 
	- 不过有个小问题：大部分记录的 `bot_name` 都显示 "nanobot" 而不是具体的 bot 名字，说明 nanobot 容器的 `BOT_NAME` 环境变量可能没有正确设置。
- **Bug 3** cc难度路由问题，分级路由存在问题，回复要标注使用了什么模型（sonnet，opus）cost-quality tradeoff routing
### **Bug 4** nanobot 太慢

#### Amy vs Nanobot 的 `elapsed_sec` 计时方式对比  
  
Amy (OpenClaw):

_// 起点：stream 开始时_

controller.currentStreamStartTime _// 模型开始输出的时刻_

_// 终点：_

Date.now() _// 卡片 finalize_

计的是：模型开始输出 → 输出完成  
  
Nanobot:

start_time = time.time() _# 进入 handler 的第一行_

_# ... 加载历史、组装 prompt、调模型、执行工具 ..._

elapsed = int(time.time() - start_time) _# 全部完成_

计的是：从收到请求 → 全部处理完，包含了历史加载、prompt 组装、工具执行等所有前置工作  
  
 **这就是为什么 shadow 的 elapsed_sec 永远大一点**

|阶段|Amy 计时|Nanobot 计时|
|---|---|---|
|加载对话历史|❌ 不算|✅ 算在里面|
|组装 prompt|❌ 不算|✅ 算在里面|
|模型思考+输出|✅ 算|✅ 算|
|工具调用|✅ 算|✅ 算|

Nanobot 的计时器从更早的地方开始跑，所以数字永远大一点。不是模型更慢，是计时范围更大。

**Before**
![[Pasted image 20260411224808.png]]
**After**
![[Pasted image 20260411224904.png]]