## 技术选型

**语言**：使用go语言，内存占用小，适合个人服务器
**框架**：eino框架

## 架构

### openclaw

![[Pasted image 20260213125623.png]]

### myclaw

```text
┌─────────────────────────────────────────────────────────┐
│                      CLI (cobra)                        │
│              agent | gateway | onboard | status          │
└──────┬──────────────────┬───────────────────────────────┘
       │                  │
       ▼                  ▼
┌──────────────┐  ┌───────────────────────────────────────┐
│  Agent Mode  │  │              Gateway                  │
│  (single /   │  │                                       │
│   REPL)      │  │  ┌─────────┐  ┌──────┐  ┌─────────┐  │
└──────┬───────┘  │  │ Channel │  │ Cron │  │Heartbeat│  │
       │          │  │ Manager │  │      │  │         │  │
       │          │  └────┬────┘  └──┬───┘  └────┬────┘  │
       │          │       │          │           │        │
       ▼          │       ▼          ▼           ▼        │
┌──────────────┐  │  ┌─────────────────────────────────┐  │
│  agentsdk-go │  │  │          Message Bus             │  │
│   Runtime    │◄─┤  │    Inbound ←── Channels          │  │
│              │  │  │    Outbound ──► Channels          │  │
└──────────────┘  │  └──────────────┬──────────────────┘  │
                  │                 │                      │
                  │                 ▼                      │
                  │  ┌──────────────────────────────────┐  │
                  │  │      agentsdk-go Runtime         │  │
                  │  │   (ReAct loop + tool execution)  │  │
                  │  └──────────────────────────────────┘  │
                  │                                       │
                  │  ┌──────────┐  ┌────────────────────┐  │
                  │  │  Memory  │  │      Config        │  │
                  │  │ (MEMORY  │  │  (JSON + env vars) │  │
                  │  │  + daily)│  │                    │  │
                  │  └──────────┘  └────────────────────┘  │
                  └───────────────────────────────────────┘

Data Flow (Gateway Mode):
  Telegram/Feishu/WeCom/WhatsApp/WebUI ──► Channel ──► Bus.Inbound ──► processLoop
                                                                      │
                                                                      ▼
                                                               Runtime.Run()
                                                                      │
                                                                      ▼
                                       Bus.Outbound ──► Channel ──► Telegram/Feishu/WeCom/WhatsApp/WebUI
```


## 项目功能

通过telegram可以指挥agent在服务器工作，支持长期记忆，支持skill，不用webui
- cli agent
- feishu channel
- long term memory
- skill （后期功能）
- cron(后期功能)

写做最小的mvp，用飞书能查询服务器状态，
## 项目模块

### CLI
cobra

### Gateway

### Agent

```text
用户消息 + Memory Context + Skills
           │
           ▼
    ┌─────────────────┐
    │  System Prompt  │  ← AGENTS.md + SOUL.md + Memory
    └────────┬────────┘
             │
             ▼
┌────────────────────────────────────────┐
│        agentsdk-go Runtime             │
│  ┌──────────────────────────────────┐  │
│  │     ReAct Loop (迭代推理)        │  │
│  │  1. Prompt → LLM                 │  │
│  │  2. LLM decide: 回答/工具        │  │
│  │  3. 执行工具                     │  │
│  │  4. 结果返回 LLM                 │  │
│  │  5. 重复直到完成 (最多20次)      │  │
│  └──────────────────────────────────┘  │
└────────┬───────────────────────────────┘
         │
         ▼
      最终回复

```
#### ReAct Loop


## 开发路线






  ---

## 开发计划（3天）
