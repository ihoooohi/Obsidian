# OpenClaw 开发

## 项目概述

OpenClaw 是一个功能强大的 AI Agent 服务，支持多通道接入（飞书、Telegram、Slack 等），用户可通过 IM 远程控制服务器。

## 技术栈

- **语言**: TypeScript/JavaScript
- **运行时**: Node.js
- **框架**: 自研 Agent 框架
- **容器**: Docker (沙箱支持)

## 核心架构

### Tool 系统

OpenClaw 的 Tool 系统是其核心特性，采用分层架构：

```
Tool Catalog (工具目录)
    ↓
Tool Factory (工具工厂)
    ↓
Tool Policy (工具策略)
    ↓
Agent (执行)
```

### Tool 分类

| 分类 | ID | 说明 |
|------|-----|------|
| Files | fs | read, write, edit, apply_patch |
| Runtime | runtime | exec, process |
| Web | web | web_search, web_fetch |
| Memory | memory | memory_search, memory_get |
| Sessions | sessions | sessions_list, sessions_send, subagents |
| UI | ui | browser, canvas |
| Messaging | messaging | message |
| Automation | automation | cron, gateway |
| Nodes | nodes | nodes (设备控制) |
| Agents | agents | agents_list |
| Media | media | image, tts |

### Tool Profile

- **minimal**: 最小化配置，仅 session_status
- **coding**: 编程场景，包含文件操作、命令执行、记忆等
- **messaging**: 消息场景，包含会话管理、消息发送
- **full**: 全部工具

## Exec Tool 详解

### 功能特性

```typescript
{
  name: "exec",
  description: "Execute shell commands with background continuation",
  parameters: {
    command: string,      // 要执行的命令
    workdir?: string,     // 工作目录
    env?: Record<string, string>,  // 环境变量
    yieldMs?: number,     // 超时后转为后台执行
    background?: boolean,  // 直接后台执行
    timeout?: number,     // 超时时间（秒）
    pty?: boolean,        // 是否使用 PTY（终端 UI）
    elevated?: boolean,   // 是否提权
    host?: string,        // 执行主机
    security?: string,    // 安全级别
    ask?: string,         // 审批模式
    node?: string,       // 远程节点
  }
}
```

### 安全机制

#### 1. Host 配置

| Host | 说明 |
|------|------|
| sandbox | Docker 沙箱执行（隔离） |
| gateway | 本地执行 |
| node | 远程节点执行 |

#### 2. Security 级别

| 级别 | 说明 |
|------|------|
| allowlist | 白名单模式（只允许指定命令） |
| deny | 黑名单模式（禁止危险命令） |
| full | 完全权限 |

#### 3. Ask 模式（审批）

| 模式 | 说明 |
|------|------|
| on-miss | 白名单外需审批 |
| always | 总是需要审批 |
| off | 无需审批 |

#### 4. Elevated（提权）

```typescript
{
  enabled: boolean,        // 是否启用
  allowed: string[],      // 允许的通道
  defaultLevel: "full" | "ask" | "off",  // 默认级别
  allowFrom: {
    feishu: true,         // 飞书允许
    telegram: false,      // Telegram 不允许
  }
}
```

#### 5. Safe Bins（安全二进制）

```typescript
{
  safeBins: ["ls", "cat", "grep"],  // 白名单命令
  safeBinProfiles: {                 // 命令配置
    python: {
      allowed: true,
      args: ["-u", "--version"],
    }
  },
  safeBinTrustedDirs: ["/usr/bin"],  // 信任目录
}
```

### 后台执行

```typescript
// 方式 1: yieldMs 超时转后台
{
  command: "long-running-task",
  yieldMs: 5000,  // 5 秒后转为后台
}

// 方式 2: 直接后台执行
{
  command: "background-task",
  background: true,
}

// 后台进程管理
process.list()      // 列出后台进程
process.poll(id)    // 查询进程状态
process.log(id)     // 查看进程日志
process.kill(id)    // 终止进程
```

## Tool Policy Pipeline

### 多层权限检查

```typescript
applyToolPolicyPipeline({
  tools: tools,
  steps: [
    { policy: profilePolicy, label: "profile" },
    { policy: globalPolicy, label: "global" },
    { policy: agentPolicy, label: "agent" },
    { policy: groupPolicy, label: "group" },
    { policy: sandbox?.tools, label: "sandbox" },
    { policy: subagentPolicy, label: "subagent" },
  ],
});
```

### 策略优先级

```
Profile < Global < Agent < Group < Sandbox < Subagent
```

## 沙箱支持

### Docker 沙箱

```typescript
{
  sandbox: {
    mode: "all" | "non-main",  // 沙箱模式
    workspaceDir: "/workspace",  // 工作目录
    containerWorkdir: "/work",   // 容器内工作目录
    env: {                      // 环境变量
      PATH: "/usr/local/bin:/usr/bin",
    },
    browser: {
      bridgeUrl: "http://localhost:9222",  // 浏览器桥接
      allowHostControl: true,
    },
    fsBridge: fsBridge,  // 文件系统桥接
  }
}
```

### 沙箱优势

- **隔离性**: 命令在容器内执行，不影响宿主机
- **安全性**: 可以限制网络、文件系统访问
- **可重现性**: 统一的执行环境

## 与 TinyClaw 对比

| 方面 | OpenClaw | TinyClaw |
|------|----------|----------|
| 语言 | TypeScript | Go |
| 工具数量 | 25+ | 1 (exec) |
| 安全机制 | 多层策略管道 | 简单黑名单 |
| 后台执行 | 支持 | 不支持 |
| 沙箱 | Docker 沙箱 | 无 |
| 审批流程 | 支持 | 无 |
| 配置粒度 | 全局/Agent/群组/Profile | 仅全局 |
| 通道支持 | 10+ | 1 (飞书) |

## 可借鉴的设计

### 1. 工具目录化

```typescript
// 定义工具元信息
const CORE_TOOL_DEFINITIONS = [
  { id: "read", label: "read", description: "Read file contents", ... },
  { id: "exec", label: "exec", description: "Run shell commands", ... },
];

// 支持 Profile
const CORE_TOOL_PROFILES = {
  minimal: { allow: ["session_status"] },
  coding: { allow: ["read", "write", "exec", ...] },
};
```

### 2. 安全策略分层

```typescript
// 全局策略
globalPolicy = { allow: ["ls", "cat"] }

// Agent 策略
agentPolicy = { deny: ["rm -rf"] }

// 群组策略
groupPolicy = { allow: ["git"] }

// Profile 策略
profilePolicy = { allow: ["read", "write"] }
```

### 3. 后台执行

```typescript
// 超时转后台
{
  command: "long-task",
  yieldMs: 5000,
}

// 后台进程管理
process.list()    // 列出进程
process.poll(id)  // 查询状态
process.kill(id)  // 终止进程
```

### 4. 审批流程

```typescript
{
  ask: "on-miss",  // 白名单外需审批
  // 或
  ask: "always",   // 总是需要审批
}
```

## 核心文件

| 文件 | 作用 |
|------|------|
| `tool-catalog.ts` | 工具目录定义 |
| `pi-tools.ts` | 工具工厂入口 |
| `bash-tools.exec.ts` | Exec 工具实现 |
| `tool-policy.ts` | 工具权限策略 |
| `tool-policy-pipeline.ts` | 策略管道 |
| `sandbox.ts` | 沙箱管理 |

## 开发建议

### TinyClaw 改进方向

1. **工具目录化**: 定义工具元信息，支持分类和 Profile
2. **安全策略分层**: 支持全局/Agent/用户级别策略
3. **后台执行**: 实现 yieldMs 和 process 工具
4. **审批流程**: 支持危险命令审批
5. **沙箱支持**: 集成 Docker 沙箱

### 优先级

| 优先级 | 功能 | 说明 |
|--------|------|------|
| 高 | 后台执行 | 长时间任务不阻塞 |
| 高 | 审批流程 | 安全性提升 |
| 中 | 工具目录化 | 可扩展性 |
| 中 | 安全策略分层 | 灵活性 |
| 低 | 沙箱支持 | 隔离性 |

## 参考资料

- [OpenClaw GitHub](https://github.com/sipeed/openclaw)
- [Tool Catalog](src/agents/tool-catalog.ts)
- [Exec Tool](src/agents/bash-tools.exec.ts)
- [Tool Policy](src/agents/tool-policy.ts)
