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

## 开发路线


  我们不要试图一口气吃成胖子，我们将项目分解为 5 个阶段：


   * 阶段一：骨架搭建 (Skeleton)
       * 目标：建立项目目录，跑通一个只会打印 "Hello World" 的命令行工具。
       * 学习：Go 模块管理 (go mod)，Cobra 基础。


   * 阶段二：配置系统 (Configuration)
       * 目标：程序能读取 config.json 文件，知道你的 API Key 是什么。
       * 学习：结构体定义，JSON 处理。

   * 阶段三：连接大脑 (Agent Core)
       * 目标：能通过命令行发送一句话给 AI，并收到回复。
       * 学习：HTTP 请求，或者使用 AI SDK。

   * 阶段四：持续对话 (REPL)
       * 目标：像聊天一样，你一句我一句，直到你输入 exit。
       * 学习：标准输入输出流处理。


   * 阶段五：构建网关 (Gateway - 进阶)
       * 目标：搭建一个简单的 Web 服务，通过网页或 API 接收消息。
       * 学习：HTTP Server，并发处理。

  ---

## 开发计划（3天）
