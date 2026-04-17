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
#### 怎么修 在服务器上跑一条命令就行：  

cd /home/ubuntu/repos/VisPie_backend/auto-amy/docker && \

docker compose -f docker-compose.amy-nanobot.yml up -d --force-recreate

这会用最新的 compose 文件重建容器，所有新挂载就会生效，nanobot 就能访问 docker 了。

### Bug 7 nanobot shadow 经常没用

![[Pasted image 20260415164637.png]]
### Bug  8 NR监控Amy

要用 New Relic 监控 Amy，分两层，由浅到深：

#### 第一层：基础设施监控（必做）

在 Amy Server (`ec2-100-29-3-1`) 宿主机上装 Infrastructure Agent。  

_# 1. SSH 进服务器_

ssh -i your_key.pem ubuntu@ec2-100-29-3-1.compute-1.amazonaws.com _# 2. 安装 Infrastructure Agent_

curl -Ls https://download.newrelic.com/install/newrelic-cli/scripts/install.sh | bash

sudo NEW_RELIC_API_KEY=<YOUR_INGEST_KEY> NEW_RELIC_ACCOUNT_ID=<YOUR_ACCOUNT_ID> /usr/local/bin/newrelic install -n infrastructure-agent-installer _# 3. 安装 Docker 集成_

sudo apt-get install nri-docker -y _# 4. 重启 agent_

sudo systemctl restart newrelic-infra

  
能看到：  

- 宿主机 CPU / 内存 / 磁盘 / 网络
    

- 每个容器（`amy-prod`、`eva-prod`、所有 nanobot）的 CPU%、内存用量、重启事件
    

- 容器 OOM / 崩溃 / 重启告警
    

#### 第二层：APM 应用监控（可选，需改容器配置）

监控 Amy 的 Node.js 进程（请求耗时、错误率、具体调用链）。方法：给 `amy-prod` 容器加环境变量和 npm 包：  

_# 进入容器_

docker exec -it amy-prod bash _# 安装 New Relic Node.js agent_

npm install newrelic _# 在容器启动命令前加环境变量_

export NEW_RELIC_APP_NAME="amy-prod"

export NEW_RELIC_LICENSE_KEY="<YOUR_LICENSE_KEY>"

export NODE_OPTIONS="-r newrelic"

但这个方法有两个问题：  

1. 容器重启后安装的包会丢失（需要改 Dockerfile 持久化）
    

2. 加 `NODE_OPTIONS` 需要重启 Node 进程 = 重启 Amy gateway = 需要 George 操作  
    所以建议：先只做第一层，已经能监控 Amy 容器的健康状态。第二层等有需要再说。
    

|你需要准备的|东西|从哪来|
|---|---|---|
|New Relic 账号|[newrelic.com](https://newrelic.com) 注册免费版（100GB/月免费）|--|
|License Key / Ingest Key|New Relic 控制台 → API Keys|--|
|SSH PEM 密钥 + 用户名|找 George 要|--|

总结：注册 New Relic → 拿到 Key → 找 George 要 SSH 权限 → 在宿主机上跑几条命令 → 完成。

### Bug 9 shadow有时有概率不回复

![[Pasted image 20260416002905.png]]

![[Pasted image 20260416002941.png]]
![[Pasted image 20260416003158.png]]
### Bug 10 shadow 有时候输出不完整

![[Pasted image 20260416003109.png]]

确认了，磁盘版本是正确的 slice(0, 4000)。

  结论：只差一个 amy-prod 重启                                             
   
  磁盘 /data/state/extensions/vizzy-lark/dist/src/shadow-judge.js →        
  slice(0, 4000) ✅                                                        
  运行中 Node 进程（16:37 UTC 启动，PR 合并前 11 分钟）         → slice(0,
  1500) ❌                                                                 
                                                         
  Node.js require() 缓存 —— 文件已是新的，但进程里加载的还是旧代码。docker 
  restart amy-prod 之后立即生效。                        
                                                                           
  ⚠️  重启前必须走 checklist                              

  直接 docker restart amy-prod 在 2026-02-27 出过大事（OOM 级联 →          
  SSM/SSH/Cloudflare Tunnel 全挂 → EC2 reboot 才恢复）。完整流程：
                                                                           
  # 1) 先看有没有活跃 session（有人正在跟 Amy 聊）       
  curl -s http://localhost:18789/api/sessions 2>/dev/null | python3 -m     
  json.tool | head -60                                                     
  # 或者                                                                   
  sudo docker exec amy-prod ls /data/state/agents/main/sessions/ 2>&1 |    
  tail -20                                                                 
   
  # 2) 如果有活跃 session：                                                
  #    a) 要么等他们聊完                                 
  #    b) 要么通过 OpenClaw RPC sessions.abort 每一个
  #    （不要直接 restart，进行中的 session 会触发全部 OpenClaw 重新初始化 
  → 并行 Bedrock 调用 → OOM）                                              
                                                                           
  # 3) 确认空闲后，等 10 秒让 graceful shutdown 完成，然后：               
  sudo docker restart amy-prod                           
                                                                           
  # 4) 立即监控内存和日志                                                  
  free -m && sudo docker logs amy-prod --tail 30
                                                                           
  # 5) 验证新代码加载（应该还是 4000，但现在是进程内存里的 4000 了）：     
  sudo docker exec amy-prod grep -n "shadow.response.slice"
  /data/state/extensions/vizzy-lark/dist/src/shadow-judge.js               
                                                         
  建议时机                                                                 
                                                         
  - 最安全：北京时间晚上 / 周末，Amy 基本不被调用时                        
  - 比较安全：工作时间但当前没在跟 Amy 对话（先看 sessions.list）
  - 不要做：Amy 正在回复长消息、正在跑多步工具调用、正在执行 autopilot     
  的时候                                                                   
                                                                           
  我不帮你自动执行这个 docker restart，你亲自确认时机最稳。重启完如果      
  shadow 回复还是卡在 1500 附近，再告诉我 —— 那就说明还有没发现的截断路径。
### Bug 11 难度选择器不好

 ![[Pasted image 20260416023928.png]]
### Bug 12 shadow 表格无法渲染

 ![[Pasted image 20260417192426.png]]