# OpenWork 架构分析

## 一、项目概述

OpenWork 是一个 AI 编程助手客户端，其核心定位是 **"OpenCode 的消费者"**。OpenWork 通过包装 OpenCode Server，提供跨平台（桌面、移动、Web）的统一用户体验，并在此基础上增加了工作区（Workspace）抽象、编排器（Orchestrator）和多运行时支持。

本文档详细分析 OpenWork 的架构设计和各组件间的协作流程。

## 二、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        用户界面层 (OpenWork UI)                            │
│  packages/app - SolidJS 前端应用                                           │
│  └── 组件: Session, Composer, Sidebar, Settings, Skills                   │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OpenWork Server API 层                               │
│  packages/server/src/server.ts                                             │
│  └── 路由: /w/:id/*, /workspace/:id/*, /workspaces/*                     │
│  └── 代理: proxyOpencodeRequest() → OpenCode Server                      │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OpenCode Server (被调用方)                           │
│  由 OpenWork 启动或连接的外部服务                                          │
│  └── Session.prompt(), Session.create(), Event.subscribe()                │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        编排器层 (Orchestrator)                              │
│  packages/orchestrator/src/cli.ts                                          │
│  └── 启动和管理 OpenCode Server、OpenWork Server、OpenCodeRouter           │
│  └── 沙箱模式: none, docker, container                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 三、运行时模式

OpenWork 支持三种运行时连接模式：

### Mode A - Host (桌面/服务端)

- OpenWork 运行在桌面/笔记本上，**本地启动** OpenCode Server
- OpenCode Server 运行在回环地址（默认 `127.0.0.1:4096`）
- OpenWork UI 通过官方 SDK 连接并监听事件

### Mode B - Client (桌面/移动端)

- OpenWork 运行在 iOS/Android 上作为**远程控制器**
- 连接到由可信设备托管的已运行 OpenCode Server
- 配对使用二维码/一次性令牌和安全传输（LAN 或隧道）

### Mode C - Hosted OpenWork Cloud

- 用户登录托管 OpenWork Web/App 界面
- 用户从托管控制平面启动云端 Worker
- OpenWork 返回远程连接凭据（`/w/ws_*` URL + 访问令牌）
- 用户通过"添加 Worker" → "连接远程"在 OpenWork App 中连接

## 四、函数层级包含关系

### L0: 入口层 (Entry Layer)

```
createClient(baseUrl, directory, auth)  【创建 OpenCode 客户端】
│   │  创建与 OpenCode Server 通信的客户端
│   │
│   ├── resolveAuthHeader(auth)  【解析认证头】
│   │       处理 basic auth 或 bearer token
│   │
│   ├── createTauriFetch(auth)  【创建 Tauri fetch】
│   │       为 Tauri 运行时创建带认证的 fetch 函数
│   │
│   └── createOpencodeClient({...})  【调用 SDK】
│           使用 @opencode-ai/sdk/v2 创建客户端
│
│
waitForHealthy(client, options)  【等待服务健康】
│   │  等待 OpenCode Server 变为健康状态
│   │
│   └── client.global.health()  【健康检查】
│           轮询健康端点直到超时
```

### L1: UI 会话层 (UI Session Layer)

```
abortSession(client, sessionID)  【中止会话】
│   │  中止一个活跃的会话
│   │
│   └── client.session.abort({ sessionID })  【调用 SDK】
│
│
revertSession(client, sessionID, messageID)  【回退会话】
│   │  将会话回退到指定消息边界
│   │
│   └── client.session.revert({ sessionID, messageID })
│
│
shellInSession(client, sessionID, command, options)  【Shell 执行】
│   │  在会话中执行 shell 命令
│   │
│   └── client.session.shell({ sessionID, command })
│
│
compactSession(client, sessionID, model, options)  【压缩会话】
│   │  压缩/总结长会话以减少上下文大小
│   │
│   ├── client.session.summarize({...})  【优先使用 summarize】
│   │
│   └── client.session.command({ command: "compact" })  【回退到命令】
```

### L2: Server 路由层 (Server Routing Layer)

```
createServer(config)  【创建 OpenWork Server】
│   │  创建 HTTP 服务器，注册所有路由
│   │
│   ├── createServerLogger(config)  【创建日志器】
│   │       创建结构化日志输出
│   │
│   ├── addRoute(routes, method, path, auth, handler)  【添加路由】
│   │       注册 API 路由
│   │       │
│   │       ├── [Workspace 路由]
│   │       │   ├── GET /w/:id/workspaces
│   │       │   ├── GET /workspaces
│   │       │   ├── POST /workspaces/:id/activate
│   │       │   └── DELETE /workspaces/:id
│   │       │
│   │       ├── [Session 路由 - 代理到 OpenCode]
│   │       │   ├── GET /workspace/:id/sessions/:sessionId
│   │       │   └── DELETE /workspace/:id/sessions/:sessionId
│   │       │
│   │       ├── [Config 路由]
│   │       │   ├── GET /workspace/:id/config
│   │       │   └── PATCH /workspace/:id/config
│   │       │
│   │       ├── [Plugins 路由]
│   │       │   ├── GET /workspace/:id/plugins
│   │       │   ├── POST /workspace/:id/plugins
│   │       │   └── DELETE /workspace/:id/plugins/:name
│   │       │
│   │       ├── [Skills 路由]
│   │       │   ├── GET /workspace/:id/skills
│   │       │   └── POST /workspace/:id/skills
│   │       │
│   │       ├── [Events 路由]
│   │       │   └── GET /workspace/:id/events
│   │       │
│   │       ├── [Engine 路由]
│   │       │   └── POST /workspace/:id/engine/reload
│   │       │
│   │       └── [OpenCodeRouter 路由]
│   │           ├── GET/POST /workspace/:id/opencode-router/*
│   │           └── GET/POST /workspace/:id/opencode-router/identities/*
│   │
│   └── server.listen()  【启动服务器】
│
│
addRoute(routes, method, path, auth, handler)  【路由注册】
│   │  将路由添加到路由数组
│   │
│   └── 返回添加的 Route 对象
```

### L3: 代理层 (Proxy Layer)

```
proxyOpencodeRequest(input)  【代理请求到 OpenCode】
│   │  将请求转发到 OpenCode Server
│   │
│   ├── resolveOpencodeDirectory(workspace)  【获取工作目录】
│   │       解析工作区对应的文件系统路径
│   │
│   ├── buildOpencodeProxyUrl(baseUrl, path, search)  【构建目标 URL】
│   │       构建转发到 OpenCode 的目标 URL
│   │
│   ├── buildOpencodeAuthHeader(workspace)  【构建认证头】
│   │       为 OpenCode 请求生成认证头
│   │
│   └── fetch(targetUrl, { method, headers, body })  【转发请求】
│           实际执行 HTTP 请求
│
│
proxyOpenCodeRouterRequest(input)  【代理请求到 OpenCodeRouter】
│   │  将请求转发到 OpenCodeRouter
│   │
│   ├── resolveOpenCodeRouterBaseUrl()  【获取 Router 地址】
│   │       从环境变量获取 OpenCodeRouter 端口
│   │
│   ├── buildOpenCodeRouterProxyUrl(baseUrl, path, search)  【构建 URL】
│   │
│   └── fetch(targetUrl, { method, headers, body })  【转发请求】
```

### L4: 编排器层 (Orchestrator Layer)

```
main()  【Orchestrator 主入口】
│   │  主编排逻辑
│   │
│   ├── parseArgs(argv)  【解析命令行参数】
│   │       解析位置参数和标志
│   │
│   ├── resolveSidecarConfig()  【解析 Sidecar 配置】
│   │       │  确定 sidecar 的来源（bundled/external/downloaded）
│   │       │
│   │       ├── checkBundled()  【检查捆绑版本】
│   │       ├── checkExternal()  【检查外部版本】
│   │       └── checkRemote()  【检查远程版本】
│   │
│   ├── startDaemon()  【启动守护进程】
│   │       │  启动 OpenWork Server 作为守护进程
│   │       │
│   │       ├── spawnOpenworkServer()  【生成 Server 进程】
│   │       │       启动 OpenWork Server
│   │       │
│   │       └── waitForHealthy()  【等待健康】
│   │               等待 Server 就绪
│   │
│   ├── startOpencode()  【启动 OpenCode】
│   │       │  启动 OpenCode Server
│   │       │
│   │       ├── spawnOpencodeServe()  【生成 OpenCode 进程】
│   │       │       使用子进程启动 OpenCode
│   │       │
│   │       └── waitForHealthy()  【等待健康】
│   │
│   ├── startOpenCodeRouter()  【启动 OpenCodeRouter】
│   │       │  启动 OpenCodeRouter（可选）
│   │       │
│   │       ├── spawnOpenCodeRouter()  【生成 Router 进程】
│   │       └── waitForHealthy()  【等待健康】
│   │
│   └── createClient()  【创建客户端连接】
│           创建到 OpenWork Server 的客户端
```

### L5: 认证与授权层 (Auth Layer)

```
requireClient(request, config, tokens)  【客户端认证】
│   │  验证客户端请求的认证
│   │
│   ├── extractTokenFromHeader(request)  【提取令牌】
│   │       从 Authorization 头提取令牌
│   │
│   ├── tokens.verify()  【验证令牌】
│   │       验证 JWT 令牌有效性
│   │
│   └── 返回 Actor 对象
│
│
requireHost(request, config, tokens)  【主机认证】
│   │  验证主机请求的认证
│   │
│   ├── verifyHostToken()  【验证主机令牌】
│   │       验证 x-openwork-host-token
│   │
│   └── 返回 Actor 对象
│
│
resolveAuthMode(pathname, method)  【解析认证模式】
│   │  根据路径和方法确定需要的认证级别
│   │
│   ├── resolveOpenCodeRouterProxyPolicy()  【Router 策略】
│   │       确定 OpenCodeRouter 路径的认证要求
│   │
│   └── 返回 AuthMode (none | client | host)
```

## 五、核心数据流

### 会话创建流程

```
用户点击 "New Session"
        │
        ▼
UI: client.session.create({ title })
        │
        ▼
OpenWork Server: POST /workspace/:id/sessions
        │
        ▼
proxyOpencodeRequest() 
        │
        ▼
OpenCode Server: Session.create()
        │
        ▼
返回 session ID
        │
        ▼
UI 订阅事件流
```

### 消息发送流程

```
用户输入消息
        │
        ▼
UI: client.session.prompt({ sessionID, parts })
        │
        ▼
OpenWork Server: POST /session/:id/prompt (代理)
        │
        ▼
proxyOpencodeRequest() → OpenCode Server
        │
        ▼
LLM 处理 → 流式响应
        │
        ▼
OpenCode Server → SSE 事件
        │
        ▼
OpenWork Server 代理事件流
        │
        ▼
UI: client.event.subscribe() 消费事件
        │
        ▼
实时更新 UI
```

### 权限处理流程

```
OpenCode 请求权限
        │
        ▼
OpenCode Server → 权限事件
        │
        ▼
OpenWork Server 捕获事件
        │
        ▼
通过 SSE 发送到 UI
        │
        ▼
UI 显示权限请求对话框
        │
        ▼
用户选择 (once / always / reject)
        │
        ▼
UI: client.permission.reply({ requestID, reply })
        │
        ▼
OpenWork Server → OpenCode Server
        │
        ▼
继续/终止执行
```

## 六、关键模块说明

### packages/server

OpenWork Server 是整个系统的核心 API 表面，提供：

- **工作区管理**: 激活、删除、配置工作区
- **会话代理**: 将会话相关请求代理到 OpenCode
- **扩展管理**: Skills、Plugins、MCP 配置
- **事件流**: SSE 实时事件推送
- **认证授权**: 客户端和主机认证
- **OpenCodeRouter 集成**: Telegram/Slack 消息集成

### packages/orchestrator

OpenWork Orchestrator 是 CLI 主机，负责：

- **多 Sidecar 管理**: 同时运行 OpenWork Server、OpenCode Server、OpenCodeRouter
- **沙箱模式**: 支持 none、docker、container 模式
- **版本管理**: bundled、external、downloaded 模式
- **进程生命周期**: 启动、健康检查、停止

### packages/app

OpenWork UI 是 SolidJS 前端，提供：

- **会话管理**: 创建、切换、删除会话
- **消息编辑**: 富文本编辑器、Markdown 预览
- **上下文面板**: 文件浏览、Artifacts
- **技能市场**: Skill 安装和管理
- **设置面板**: 模型、Provider、认证配置

## 七、与 OpenCode 的关系

OpenWork 与 OpenCode 的核心区别：

| 特性 | OpenCode | OpenWork |
|------|----------|----------|
| 定位 | AI 编程引擎 | 客户端体验 |
| 核心流程 | Session Loop | 代理 + 编排 |
| 会话管理 | 内部实现 | 代理到 OpenCode |
| 工作区 | 无 | Workspace 抽象 |
| 运行时 | 单一 | 多模式（Host/Client/Cloud） |
| 编排 | 无 | Orchestrator |

OpenWork 本质上是 **OpenCode 的包装层**，在 OpenCode 基础上增加了：
1. 跨平台 UI（桌面、移动、Web）
2. 工作区（Workspace）抽象
3. 多运行时支持
4. 编排器（Orchestrator）
5. 消息集成（OpenCodeRouter）

## 八、API 路由速查

### Workspace 路由

```
GET    /w/:id/workspaces          # 获取工作区列表
GET    /workspaces                # 获取所有工作区
POST   /workspaces/:id/activate   # 激活工作区
DELETE /workspaces/:id            # 删除工作区
GET    /workspace/:id/config      # 获取配置
PATCH  /workspace/:id/config      # 更新配置
GET    /workspace/:id/audit       # 获取审计日志
```

### Session 路由 (代理到 OpenCode)

```
GET    /workspace/:id/sessions/:sessionId  # 获取会话
DELETE /workspace/:id/sessions/:sessionId # 删除会话
```

### Extension 路由

```
GET    /workspace/:id/plugins     # 获取插件列表
POST   /workspace/:id/plugins     # 添加插件
DELETE /workspace/:id/plugins/:name  # 删除插件
GET    /workspace/:id/skills     # 获取技能列表
POST   /workspace/:id/skills      # 添加技能
GET    /hub/skills               # 获取技能中心
```

### Engine 路由

```
POST   /workspace/:id/engine/reload  # 重载 OpenCode 引擎
```

### Event 路由

```
GET    /workspace/:id/events      # 订阅 SSE 事件流
```
