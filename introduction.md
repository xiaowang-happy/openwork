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

### L1.5: Agent 层 (Agent Layer)

```
listAgents()  【获取 Agent 列表】
│   │  获取当前工作区可用的 Agent 列表
│   │
│   ├── client()  【获取客户端】
│   │       获取 OpenCode 客户端实例
│   │
│   ├── client.app.agents()  【调用 Agent 列表 API】
│   │       从 OpenCode 获取 Agent 列表
│   │
│   └── unwrap()  【处理响应】
│           解析响应，过滤隐藏的 Agent
│
│
loadAgentOptions(force)  【加载 Agent 选项】
│   │  加载 Agent 列表供 UI 选择器使用
│   │
│   ├── listAgents()  【获取 Agent 列表】
│   │
│   └── sort(agents)  【排序】
│           按名称字母顺序排序
│
│
setSessionAgent(sessionID, agent)  【设置会话 Agent】
│   │  为指定会话设置使用的 Agent
│   │
│   └── setSessionAgentById()  【更新状态】
│           更新全局状态中的会话 Agent
│
│
buildPromptParts(draft)  【构建提示 Parts】
│   │  构建发送给 LLM 的消息 parts
│   │
│   ├── [遍历 draft.parts]
│   │   │
│   │   ├── if type === "agent"  【Agent 引用】
│   │   │   └── { type: "agent", name: part.name }  【添加 Agent Part】
│   │   │
│   │   └── if type === "file"  【文件引用】
│   │       └── { type: "file", url: file://... }  【添加文件 Part】
│   │
│   └── [处理 attachments]  【附件处理】
│           将附件转换为 file parts
│
│
sendPrompt(draft)  【发送提示】
│   │  发送用户输入到会话
│   │
│   ├── buildPromptParts(draft)  【构建 Parts】
│   │       将用户输入转换为 SDK 格式
│   │
│   ├── selectedSessionAgent()  【获取会话 Agent】
│   │       获取当前会话选定的 Agent
│   │
│   ├── if mode === "shell"  【Shell 模式】
│   │   └── shellInSession(c, sessionID, content)
│   │
│   ├── if command  【命令模式】
│   │   └── client.session.command({ sessionID, command, agent, ... })
│   │
│   └── else  【普通提示模式】
│       └── client.session.promptAsync({ sessionID, model, agent, parts })
│
│
Agent 文件结构 (.opencode/agent/*.md)
│   │
│   ├── ---  【YAML Frontmatter】
│   │   ├── mode: primary | subagent
│   │   ├── hidden: true | false
│   │   ├── model: opencode/xxx
│   │   ├── color: "#xxx"
│   │   └── tools: { "*": false, "tool-name": true }
│   │
│   └── ##  【Markdown 描述】
│       └── Agent 的系统提示和行为规范
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
- **Agent 选择**: 支持 @agent 提及和 Agent 切换

## 七、Agent 架构详解

### Agent 定义

Agent 是 OpenCode 中的可配置 AI 行为模式，通过 `.opencode/agent/` 目录下的 Markdown 文件定义。

### Agent 文件格式

```yaml
---
mode: primary              # primary | subagent - 主 Agent 或子 Agent
hidden: true              # 是否在 UI 中隐藏
model: opencode/claude-haiku-4-5  # 使用的模型
color: "#44BA81"          # UI 显示颜色
tools:                   # 工具权限配置
  "*": false              # 禁用所有工具
  "github-triage": true   # 仅启用指定工具
---

# Agent 描述（Markdown 格式）
You are a triage agent responsible for triaging github issues.
...
```

### Agent 使用流程

```
用户在 Composer 中输入 @agentname
        │
        ▼
Composer 识别 @ 触发 mention
        │
        ▼
agentPicker 显示 Agent 列表（loadAgentOptions）
        │
        ▼
用户选择 Agent
        │
        ▼
part.type = "agent", part.name = "agentname"
        │
        ▼
sendPrompt() 构建 parts
        │
        ▼
buildPromptParts() 转换为 AgentPartInput
        │
        ▼
client.session.promptAsync({ agent: "agentname", ... })
        │
        ▼
OpenCode Server 处理 Agent 执行
```

### Agent 选择器状态管理

```typescript
// 状态定义
agentOptions: Agent[]        // 可用 Agent 列表
agentPickerOpen: boolean     // 选择器是否打开
agentPickerBusy: boolean     // 加载中状态
agentPickerError: string    // 错误信息
selectedSessionAgent: string // 当前会话选定的 Agent

// 选择器触发
// 1. 用户输入 @ 字符
// 2. 检测 mentionQuery 变化
// 3. 打开 mentionGroups 弹窗
// 4. 显示 category="agent" 的选项

// Agent 过滤逻辑
// - 隐藏 hidden: true 的 Agent
// - 排除 mode: "subagent" 的 Agent
// - 按名称字母排序
```

### Agent 与 Session 的关联

每个 Session 可以关联不同的 Agent：

```typescript
// 设置会话 Agent
setSessionAgent(sessionID, agentName)

// 获取会话 Agent
selectedSessionAgent()  // 返回当前会话的 Agent 名称

// 在 promptAsync 中传递
client.session.promptAsync({
  sessionID,
  model,
  agent: agent ?? undefined,  // 可选的 Agent 参数
  parts
})
```

### 现有 Agent 示例

OpenWork 项目中定义的 Agent（位于 `.opencode/agent/`）：

| Agent 名称 | 用途 | 工具 |
|-----------|------|------|
| triage | GitHub Issue 分类 | github-triage |
| docs | 文档相关任务 | - |
| css | CSS 相关任务 | - |
| duplicate-pr | 重复 PR 检测 | - |

## 八、Skill 架构详解

### Skill 定义

Skill 是 OpenCode 中的可复用行为模式，通过 `.opencode/skills/` 目录下的 Markdown 文件定义。

### Skill 文件结构

```
.opencode/skills/<skill-name>/
└── SKILL.md          # Skill 定义文件
```

### Skill 文件格式

```yaml
---
name: skill-name      # Skill 名称
description: 描述    # Skill 描述
trigger: 使用条件    # 触发条件（可选）
---

# When to use
- 使用场景描述

# Skill 描述
你是一个...
```

### Skill 类型

| 类型 | 路径 | 说明 |
|------|------|------|
| Project Skills | `.opencode/skills/` | 项目级 Skill |
| Global Skills | `~/.config/opencode/skills` | 全局 Skill |
| Claude Skills | `.claude/skills/` | 兼容 Claude 的 Skill |
| Hub Skills | GitHub 不同AI/openwork-hub | 远程 Skill 市场 |

### Skill 核心函数

```
listSkills(workspaceRoot, includeGlobal)  【列出 Skills】
│   │  获取工作区的 Skill 列表
│   │
│   ├── findWorkspaceRoots()  【查找工作区根目录】
│   │       向上遍历找到所有 Git 根目录
│   │
│   ├── listSkillsInDir(dir, scope)  【列出目录中的 Skills】
│   │       │  扫描目录下的 Skill 文件夹
│   │       │
│   │       ├── parseSkillEntry()  【解析 Skill 项】
│   │       │       读取 SKILL.md，解析 Frontmatter
│   │       │
│   │       └── extractTriggerFromBody()  【提取触发条件】
│   │               从 Markdown 标题 "When to use" 提取
│   │
│   └── [合并全局 Skills]  【includeGlobal 时】
│           添加 ~/.config/opencode/skills
│
│
upsertSkill(workspaceRoot, payload)  【创建/更新 Skill】
│   │  创建或更新 Skill
│   │
│   ├── validateSkillName()  【验证名称】
│   │
│   ├── parseFrontmatter(content)  【解析 Frontmatter】
│   │
│   ├── buildFrontmatter()  【构建 Frontmatter】
│   │
│   └── writeFile()  【写入文件】
│           写入 .opencode/skills/<name>/SKILL.md
│
│
deleteSkill(workspaceRoot, name)  【删除 Skill】
│   │  删除指定 Skill
│   │
│   └── rm(skillDir, { recursive: true })  【删除目录】
│
│
listHubSkills(repo)  【列出 Hub Skills】
│   │  从 GitHub 获取远程 Skill 目录
│   │
│   ├── fetchJson()  【获取目录列表】
│   │       GET https://api.github.com/repos/different-ai/openwork-hub/contents/skills
│   │
│   └── fetchText()  【获取 Skill 内容】
│           获取每个 Skill 的详细信息
```

### Skill UI 流程

```
用户打开 Skills 页面
        │
        ▼
refreshSkills()  加载 Skills
        │
        ├── client.workspace.skills.list()  【获取本地 Skills】
        │
        └── refreshHubSkills()  【获取 Hub Skills】
                │
                └── fetch GitHub API  【远程获取】
                        │
                        └── listHubSkills()  【缓存 5 分钟】
        │
        ▼
显示 Skill 列表（skills(), hubSkills()）
        │
        ├── 点击 Skill → 查看详情
        │
        ├── 安装 Hub Skill → installHubSkill()
        │       └── POST /workspace/:id/skills
        │
        └── 删除 Skill → uninstallSkill()
                └── DELETE /workspace/:id/skills/:name
```

### Skill 与命令的关系

在 Composer 的 slash 命令中显示为 `source: "skill"`：

```typescript
// composer.tsx 中的命令加载
cmd.source === "skill"  // 显示 "Skill" 标签
```

## 九、MCP Tool 架构详解

### MCP 定义

MCP (Model Context Protocol) 是开放协议，允许 AI 助手连接到外部工具和服务。

### MCP 配置

MCP 通过 `opencode.jsonc` 配置文件定义：

```jsonc
{
  "mcp": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@example/mcp-server"]
    }
  }
}
```

### MCP 核心函数

```
listMcp(workspaceRoot)  【列出 MCP 服务器】
│   │  获取工作区配置的 MCP 服务器列表
│   │
│   ├── readJsoncFile(opencodeConfigPath)  【读取项目配置】
│   │       读取 .opencode/opencode.jsonc
│   │
│   ├── readJsoncFile(globalOpenCodeConfigPath)  【读取全局配置】
│   │       读取 ~/.config/opencode/opencode.jsonc
│   │
│   ├── getMcpConfig(config)  【提取 MCP 配置】
│   │       从配置中提取 mcp 字段
│   │
│   └── isMcpDisabledByTools(config, name)  【检查是否被禁用】
│           检查 tools.deny 模式
│
│
addMcp(workspaceRoot, name, config)  【添加 MCP 服务器】
│   │  添加新的 MCP 服务器配置
│   │
│   ├── validateMcpName()  【验证名称】
│   │
│   ├── validateMcpConfig()  【验证配置】
│   │
│   └── updateJsoncTopLevel()  【更新配置】
│           更新 opencode.jsonc 的 mcp 字段
│
│
removeMcp(workspaceRoot, name)  【移除 MCP 服务器】
│   │  移除 MCP 服务器配置
│   │
│   └── updateJsoncTopLevel()  【更新配置】
│           删除 mcp 字段中的对应项
```

### MCP 路由

```
GET    /workspace/:id/mcp           # 获取 MCP 列表
POST   /workspace/:id/mcp           # 添加 MCP
DELETE /workspace/:id/mcp/:name     # 删除 MCP
```

### MCP 状态管理

```
MCP 状态类型：
- connected      # 已连接
- needs_auth     # 需要认证
- needs_client_registration  # 需要客户端注册
- failed         # 失败
- disabled       # 已禁用
- disconnected   # 已断开
```

### MCP 与命令的关系

在 Composer 的 slash 命令中显示为 `source: "mcp"`：

```typescript
// composer.tsx 中的命令加载
cmd.source === "mcp"  // 显示 "MCP" 标签
```

## 十、与 OpenCode 的关系

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

## 十一、API 路由速查

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
