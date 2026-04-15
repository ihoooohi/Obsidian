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
### Bug 5 nanobot 回复质量差
#### 现象

![[Pasted image 20260414232406.png]]
#### 原因
核心差异在 context（上下文），不是模型能力。  
  
OpenClaw primary 为什么回复质量更高：  

1. 完整的 system prompt — 我的 AGENTS.md、SOUL.md、USER.md、MEMORY.md 在每次对话开始时都被注入，里面有公司架构、Supabase、飞书、各种 API 的详细信息。所以你问"数据库是哪个"，我能直接列出来。
    

2. 对话历史连续 — OpenClaw 维护了完整的 session，之前你问过的每个问题和我的回答都在上下文里，所以我能引用之前的对话。
    

3. 记忆系统 — 每次对话会带上 `[Relevant memories from past conversations]`，这些是 mem0 里存的历史信息，帮我回忆之前讨论过的内容。
    

Shadow nanobot 为什么差一些：  

1. context 更少 — shadow 收到的是 `shadowForward()` 转发的原始 webhook 消息，不一定带完整的 system prompt 和 session 历史。它可能只看到你当前这一条消息，没有之前的对话上下文。
    

2. 没有记忆注入 — shadow 的处理路径可能跳过了 mem0 查询和 memory file 加载，所以它对公司架构、之前讨论过的内容一无所知。
    

3. 模型可能不同 — primary 用的是 Opus 4.6，shadow 的 `SUBAGENT_MODEL` 是 Sonnet 4.6。模型能力本身就有差距。
    

一句话总结： 不是 shadow 笨，是它看到的信息比我少。同样的模型，给完整 context 和给裸消息，输出质量天差地别。Context is everything。

### Bug 6 nanobot 没 docker 权限

确认了。 当前运行的 `amy-nanobot` 容器的挂载列表里完全没有 docker socket 和 docker CLI 相关的挂载：  

- ❌ `/var/run/docker.sock:/var/run/docker.sock` — 不存在
    

- ❌ `/usr/bin/docker:/usr/bin/docker:ro` — 不存在
    

- ❌ `/home/ubuntu/amy-data/prod/state:/data/state:ro` — 不存在
    

- ❌ `/home/ubuntu/amy-data/prod/workspace:/data/workspace:ro` — 不存在
    

- ❌ `/home/ubuntu/eva-data/prod/...` — 不存在
    

对比 compose 文件，这些挂载应该都有但实际都没有。  
  
结论很清楚： 当前运行的容器是用旧版本的 compose 文件创建的。compose 文件后来更新了（加了 docker socket、docker CLI、以及 `/data/state`、`/data/workspace` 等"Capability alignment with OpenClaw"的挂载），但容器没有重建，所以新配置没有生效。  
### 怎么修 在服务器上跑一条命令就行：  

cd /home/ubuntu/repos/VisPie_backend/auto-amy/docker && \

docker compose -f docker-compose.amy-nanobot.yml up -d --force-recreate

这会用最新的 compose 文件重建容器，所有新挂载就会生效，nanobot 就能访问 docker 了。

### Bug 7 nanobot shadow 经常没用

![[Pasted image 20260415164637.png]]