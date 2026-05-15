# AiToEarn 项目架构分析

> 基于 GitNexus 索引（28,709 符号 / 62,964 关系 / 300 执行流程）与源码核查的整体架构概览。
> 适用场景：新成员上手、跨模块协作、二次开发或私有化部署前的快速通读。

---

## 一、整体形态

AiToEarn 是一个 **AI Agent 驱动的多平台内容发布 / 变现** 系统：

> 用户在 Chat UI 向 Agent 下达意图 → Agent 通过 MCP 协议调用后端工具 → 后端 Providers 适配 13+ 内容平台（海外 API + 国内 Cookie 通道）

仓库为 Monorepo，包含三个**可独立部署**的子工程：

```
project/
├── aitoearn-backend/   NestJS Monorepo (Nx) — 服务端 + AI Agent 服务
├── aitoearn-web/       Next.js (App Router, i18n [lng]) — Web 控制台
└── aitoearn-electron/  Electron + Vite + React — 桌面客户端
```

---

## 二、后端 `aitoearn-backend` — NestJS Nx Monorepo

### 2.1 Apps（独立运行进程）

| App | 角色 |
| --- | --- |
| `aitoearn-server` | 业务主服务：账号、渠道、发布、内容、积分、通知、对外 MCP 网关 |
| `aitoearn-ai`     | AI 子服务：Agent 运行时、Claude-code-router、素材改写、草稿生成 |

两者通过 `libs/aitoearn-server-client` 与 `libs/aitoearn-ai-client` 互相 RPC，是典型的微服务切分。

### 2.2 `aitoearn-server/src/core` 业务领域

```
account/        api-key/        channel/        content/
credits/        fingerprint/    internal/       notification/
publish-record/ relay/          short-link/     tools/
unified-mcp/    user/
```

#### 核心模块：`channel/`

```
channel/
├── publishing/
│   └── providers/                        # 每个平台一个 Service（13 个）
│       ├── base.service.ts               # 统一抽象基类
│       ├── douyin.service.ts             kwai.service.ts
│       ├── bilibili.service.ts           wx-gzh.service.ts
│       ├── tiktok.service.ts             youtube.service.ts
│       ├── instgram.service.ts           facebook.service.ts
│       ├── threads.service.ts            twitter.service.ts
│       ├── linkedin.service.ts           pinterest.service.ts
│       └── google-business.service.ts
├── platforms/        # 平台元数据
├── interact/         # 互动（评论、回复）
├── engagement/       # 互动量数据
├── data-cube/        # 数据回流 / 报表
├── publish.controller.ts
├── publish.mcp.controller.ts      # 把发布能力暴露成 MCP 工具
└── publish-mcp.schema.ts
```

要点：
- 所有平台 Provider 继承 `base.service.ts`，统一 `immediatePublish / finalizePublish` 等生命周期方法。
- `publish.mcp.controller.ts` 把"发布"能力以 **MCP 协议** 暴露给外部 Agent（Claude / Cursor / OpenClaw 等）。

### 2.3 `aitoearn-ai/src/core/agent` — Agent 运行时

```
agent/
├── agent.controller.ts        agent.service.ts
├── services/
│   └── agent-runtime.service.ts        # AgentRuntimeService.claudeQuery (197–370)
├── claude-code-router/                 # 适配 Claude Code 协议
├── mcp/                                # Agent 侧的 MCP 客户端（subtitle.mcp 等）
├── skills/                             # 预置技能
├── skill-init.service.ts
└── agent-task-timeout.scheduler.ts
```

即"**用 Claude Code 路由 + MCP 工具集**做后端 Agent runtime"。

### 2.4 `libs/` 横切能力

```
aitoearn-auth          aitoearn-queue         aitoearn-server-client
aitoearn-ai-client     mongodb                redis
redlock                channel-db             ali-oss
aws-s3                 ali-sms                mail
nest-mcp               common                 helpers
```

亮点：

| Lib | 价值 |
| --- | --- |
| `nest-mcp` | 自研 NestJS × MCP 集成；`mcp-registry.service` + `mcp-tools.handler` 让任意 Module 声明的工具自动注册成 MCP endpoint |
| `aitoearn-server-client` / `aitoearn-ai-client` | server 与 ai 两个 app 之间的 RPC 客户端 |
| `channel-db` | 渠道相关数据模型独立成库，方便多服务共享 |
| `redlock` | 分布式锁，配合 `aitoearn-queue` 做发布任务的并发控制 |

---

## 三、Web `aitoearn-web` — Next.js App Router

```
src/
├── app/
│   └── [lng]/                          # 国际化路由
│       ├── chat/[taskId]/              # Chat 主体（核心入口）
│       └── websit/                     # 营销 / 政策页
├── components/
│   └── Chat/
│       ├── ChatMessage/                # 消息渲染（含 WorkflowComponents / PluginPublishCard）
│       ├── ActionCard/                 # Agent 操作卡片
│       └── ChannelManager/             # 渠道账号管理弹窗
├── store/
│   ├── agent/                          # Agent 端到端状态机（重点）
│   │   ├── agent.methods.ts            # createStoreMethods (48–630)
│   │   ├── handlers/action.handlers.ts
│   │   └── task-instance/
│   │       ├── sse.handler.ts          # handleStreamEvent — SSE 流事件解析
│   │       └── workflow.handler.ts     # handleToolCallComplete — 工具调用 / 工作流
│   ├── account.ts  user.ts  system.ts  notification.ts  plugin.ts
│   └── publishDetailCache.ts  thumbnailCache.ts
├── api/                                # 后端 HTTP 客户端 (ai.ts, account.ts, …)
└── hooks/  lib/  utils/  middleware.ts
```

GitNexus 索引中前 20 个执行流程几乎全部围绕：

```
ChatDetailPage / ShareModal / PluginGuideContent / SharedChatPage
  ↔ ParseClaudePrompt / processAssistantMessage / handleToolCallComplete
```

即 **Web 端核心是"Claude 风格的 Chat + 工具调用可视化"**。
发布、账号绑定、通知等业务能力均以"卡片"形态被 Agent 调起，而非用户在传统表单页直接操作。

---

## 四、Electron `aitoearn-electron` — 桌面端的"为什么存在"

```
electron/
├── plat/
│   ├── Kwai/           bilibili/        douyin/
│   ├── shipinhao/      xiaohongshu/
│   ├── coomont.ts      requestNet.ts
├── db/  main/  preload/  tray/  util/
└── src/                                  # Vite + React + Tailwind，独立桌面 UI
```

关键观察：

- `electron/plat/` 下**只有国内平台**（抖音 / 快手 / B 站 / 视频号 / 小红书）。
- 而后端 `providers/` 中**没有**小红书、视频号。

**架构决策：**

> Electron 承担"必须本机浏览器登陆 / Cookie 模式"的国内平台通道；
> 后端 `providers/` 走"有开放 API"的海外平台 + 微信公众号。

这是该项目最关键的工程取舍 —— 同一套 Agent 能力，按平台特性分流到 Web/服务端 vs 桌面端两条物理通道。

---

## 五、对外暴露方式

| 入口 | 实现位置 |
| --- | --- |
| 网页直接用 | `aitoearn-web` |
| Docker 部署 | `docker-compose.yml` + `aitoearn-backend` + `nginx/` |
| MCP 给 Claude / Cursor 等 | `aitoearn-server/src/core/channel/publish.mcp.controller.ts` + `libs/nest-mcp` |
| OpenClaw（龙虾）集成 | 走 MCP / API Key |
| 桌面端 | `aitoearn-electron`（含国内平台 Cookie 通道） |

---

## 六、调用链全景（一句话总结）

```
┌──────────────────────────┐         ┌──────────────────────────┐
│ aitoearn-web (Chat UI)   │         │ aitoearn-electron        │
│ Next.js + SSE + 工具卡片 │         │ 本机平台 Cookie 通道     │
└────────────┬─────────────┘         └────────────┬─────────────┘
             │ HTTP / SSE                          │ 本机请求
             ▼                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ aitoearn-backend                                                │
│                                                                 │
│  ┌────────────────────┐         ┌───────────────────────────┐   │
│  │ aitoearn-ai        │◄──RPC──►│ aitoearn-server            │  │
│  │ Agent Runtime      │         │ channel/publishing/providers│ │
│  │ (Claude-code-router│         │ → 13 平台                  │  │
│  │  + MCP 工具)       │         │ publish.mcp.controller     │  │
│  └────────────────────┘         │ → 对外 MCP 网关            │  │
│                                  └───────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                  ▲
                  │ MCP 协议
                  │
        ┌─────────┴──────────┐
        │ 外部 Agent          │
        │ Claude / Cursor /   │
        │ OpenClaw / …        │
        └────────────────────┘
```

> **Web (Next.js Chat) + Electron (本机平台通道) → 调用 NestJS 后端 →
> 后端通过 `channel/publishing/providers/*` 适配海外平台 API，通过 Electron `plat/*` 适配国内 Cookie 平台；
> 整套发布能力同时以 MCP 协议外抛，使 Claude / Cursor / OpenClaw 等 Agent 也能直接调用，
> 而 `aitoearn-ai` 内部又跑了一个 Claude-code-router 风格的 Agent runtime 作为"自家 Agent"。**

---

## 七、功能聚类（GitNexus 自动聚类，Top 20）

| 模块 | 符号数 | 内聚度 | 主要职责 |
| --- | ---: | ---: | --- |
| Ui              | 274 | 71% | Web UI 组件 |
| Youtube         | 163 | 63% | YouTube 平台适配 |
| Tiktok          | 126 | 70% | TikTok 平台适配 |
| Douyin          | 114 | 69% | 抖音平台适配 |
| Providers       |  89 | 72% | 平台 Provider 公共抽象 |
| User            |  81 | 71% | 用户域 |
| Components      |  81 | 67% | 通用组件 |
| Task            |  69 | 76% | 任务调度 |
| Account         |  67 | 64% | 账号绑定 |
| Tools           |  62 | 56% | Agent 工具集 |
| Services        |  61 | 65% | 通用服务 |
| Twitter         |  56 | 61% | Twitter / X 适配 |
| Kwai            |  53 | 71% | 快手适配 |
| Meta            |  52 | 73% | Facebook / Instagram / Threads 共用 |
| Task-instance   |  52 | 69% | 任务运行实例（含 SSE / workflow handler） |
| Content         |  51 | 77% | 内容域 |
| Mcp             |  47 | 79% | MCP 协议适配 |
| Bilibili        |  46 | 80% | B 站适配（内聚度最高） |
| Agent           |  42 | 62% | Agent 主流程 |
| Repositories    |  39 | 69% | 数据访问层 |

---

## 八、建议的二次阅读路径

1. **先读 `aitoearn-backend/apps/aitoearn-server/src/core/channel/publishing/providers/base.service.ts`** — 理解平台适配的统一契约。
2. **再读 `aitoearn-backend/apps/aitoearn-ai/src/core/agent/services/agent-runtime.service.ts`** — 理解 Agent 如何调用 MCP 工具。
3. **再读 `aitoearn-web/src/store/agent/agent.methods.ts` + `task-instance/sse.handler.ts`** — 理解前端如何消费 Agent 的流式输出。
4. **最后读 `aitoearn-electron/electron/plat/*`** — 理解国内平台 Cookie 通道为什么必须放在桌面端。

---

*文档生成自 GitNexus 索引快照，如代码已大幅变更，请在仓库根目录执行 `npx gitnexus analyze` 后重新生成。*
