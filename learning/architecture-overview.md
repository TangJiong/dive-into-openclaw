# OpenClaw 顶层架构解读

## 一句话定位

**OpenClaw 是一个"多通道 AI 网关"**——把 Telegram、Discord、Slack、WhatsApp、iMessage、Signal 等 20+ 消息平台统一接入，将消息路由给 AI Agent，Agent 调用各类 LLM 和工具，再把回复发回原始平台。

---

## 核心层次结构

```
用户 (手机/桌面 App / 消息平台)
        │
        ▼
┌─────────────────────────────┐
│  客户端层                    │
│  apps/ios  apps/android     │
│  apps/macos  ui/（Web UI）  │
└───────────┬─────────────────┘
            │ WebSocket / HTTP
            ▼
┌─────────────────────────────────────────┐
│  CLI 入口层  src/entry.ts               │
│  src/cli/  src/commands/               │
│  Commander.js 解析命令行参数            │
└───────────┬─────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────┐
│  Gateway（网关核心）  src/gateway/       │
│  - 启动/守护进程管理                    │
│  - WebSocket Server                     │
│  - Session 管理                         │
│  - Boot 自检（BOOT.md 机制）            │
└─────┬──────────────────┬────────────────┘
      │                  │
      ▼                  ▼
┌───────────┐    ┌──────────────────────────┐
│ Routing   │    │  Channel 层              │
│ src/routing│    │  src/channels/           │
│ - 路由解析 │    │  + extensions/telegram   │
│ - Binding  │    │  + extensions/discord    │
│ - Session  │    │  + extensions/slack      │
│   Key      │    │  + extensions/whatsapp   │
└─────┬──────┘    │  + extensions/signal     │
      │           │  + extensions/imessage   │
      │           │  + …20+ 渠道插件         │
      │           └──────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  Agents 层  src/agents/                 │
│  - Pi Agent Framework                   │
│  - 对话上下文管理                       │
│  - 模型调用（Anthropic/OpenAI/…）       │
│  - ACP（Agent Cooperation Protocol）    │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  Plugin SDK / Extensions                │
│  src/plugin-sdk/  extensions/           │
│  - 18+ LLM Provider 插件                │
│  - 工具/技能插件                        │
│  - 内存插件（memory-core, lancedb）     │
│  - 媒体插件（TTS, 图像生成）            │
└─────────────────────────────────────────┘
```

---

## 各层详解

### 1. 入口层：`src/entry.ts` + `src/index.ts`

这是整个程序的启动入口，有两种角色：
- **CLI 主程序**：`isMainModule()` 判断，加载 `src/cli/run-main.ts` 启动 Commander
- **库模式**：被其他代码 `import` 时，暴露 `library.ts` 导出的公共 API

关键设计：**Respawn 机制**（`entry.respawn.ts`）——支持 CLI 在需要时以不同参数重新启动自身进程（如切换 profile/container）。

---

### 2. CLI 层：`src/cli/` + `src/commands/`

```
src/cli/
  ├── run-main.ts          # Commander 根程序
  ├── argv.ts              # 参数解析
  ├── profile.ts           # --profile / --dev 切换
  ├── container-target.ts  # --container 路由
  └── program/             # 各子命令注册

src/commands/              # 300+ 具体命令实现
  ├── agent.ts
  ├── gateway.ts
  ├── message/
  ├── channels/
  └── …
```

---

### 3. Gateway 层：`src/gateway/`

这是**运行时核心控制平面**，主要职责：
- **WebSocket Server**：与客户端 App（iOS/Android/macOS/Web）通信
- **Session 管理**：每个对话维护独立 Session，持久化在 `~/.openclaw/sessions/`
- **Boot 自检**：启动时读取 `BOOT.md`，让 AI Agent 执行初始化任务
- **Channel Health Monitor**：监控各消息通道连接状态

---

### 4. Channel 层：`src/channels/` + `extensions/*`

这是消息平台抽象层，核心设计：

```
src/channels/
  ├── allow-from.ts      # 消息来源白名单（安全控制）
  ├── allowlists/        # 允许名单管理
  ├── command-gating.ts  # 命令权限控制
  ├── chat-type.ts       # dm / group / thread 类型
  ├── draft-stream-*     # 流式回复（打字机效果）
  └── plugins/           # Channel 插件接口定义
```

每个消息平台实现为独立 Extension 包（`extensions/telegram`、`extensions/discord` 等），遵循统一的 **Channel Contract**（`src/plugin-sdk/channel-contract.ts`）接口。

---

### 5. Routing 层：`src/routing/`

**消息路由的核心**，解决"哪条消息该给哪个 Agent"的问题：

```typescript
// resolve-route.ts 核心逻辑
type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;
  channel: string;           // 来自哪个平台
  accountId?: string;        // 哪个账号
  peer?: RoutePeer;          // 谁发的（dm / group）
  guildId?: string;          // Discord Guild
  memberRoleIds?: string[];  // 角色路由
};

type ResolvedAgentRoute = {
  agentId: string;    // 路由到哪个 Agent
  sessionKey: string; // 对应的 Session Key
  matchedBy: "binding.peer" | "binding.guild" | "default" | …;
};
```

路由优先级（从高到低）：
`peer 绑定 > 父级 peer > Guild+角色 > Guild > Team > Account > Channel > default`

---

### 6. Agents 层：`src/agents/`

AI 对话的大脑：
- **ACP（Agent Cooperation Protocol）**：`acp-spawn.ts` 支持 Agent 之间协作
- **对话上下文**：`src/sessions/` 管理持久化对话历史
- **模型适配**：支持 Anthropic、OpenAI、Gemini 等，通过 `extensions/anthropic`、`extensions/openai` 等插件接入
- **认证策略**：`auth-mode-policy.ts` 控制 Allow-from 规则

---

### 7. Plugin SDK：`src/plugin-sdk/`

这是**面向插件开发者的公共 API 层**，50+ 子路径导出，例如：

| 子路径 | 用途 |
|--------|------|
| `openclaw/plugin-sdk/core` | 核心类型和工具 |
| `openclaw/plugin-sdk/channel-contract` | 自定义消息渠道接口 |
| `openclaw/plugin-sdk/routing` | 路由钩子 |
| `openclaw/plugin-sdk/runtime` | 运行时 API |
| `openclaw/plugin-sdk/sandbox` | 沙箱执行环境 |
| `openclaw/plugin-sdk/setup` | 插件安装/配置流程 |
| `openclaw/plugin-sdk/provider-setup` | LLM Provider 注册 |

---

### 8. Extensions（插件生态）：`extensions/`

84 个 npm 包，分三大类：

**LLM Provider 插件（18+）**

| 插件 | 说明 |
|------|------|
| `extensions/anthropic` | Claude 系列模型 |
| `extensions/openai` | GPT 系列模型 |
| `extensions/google` | Gemini 系列模型 |
| `extensions/deepseek` | DeepSeek 模型 |
| `extensions/groq` | Groq 推理加速 |
| `extensions/ollama` | 本地模型 |
| `extensions/mistral` | Mistral 模型 |
| `extensions/xai` | Grok 模型 |
| `extensions/amazon-bedrock` | AWS Bedrock |
| `extensions/microsoft` | Azure OpenAI |

**消息渠道插件（20+）**

| 插件 | 说明 |
|------|------|
| `extensions/telegram` | Telegram Bot |
| `extensions/discord` | Discord Bot |
| `extensions/slack` | Slack App |
| `extensions/whatsapp` | WhatsApp Web |
| `extensions/signal` | Signal |
| `extensions/imessage` | iMessage（macOS） |
| `extensions/matrix` | Matrix 协议 |
| `extensions/msteams` | Microsoft Teams |
| `extensions/line` | LINE |
| `extensions/feishu` | 飞书 |
| `extensions/zalo` | Zalo |
| `extensions/irc` | IRC |
| `extensions/twitch` | Twitch Chat |
| `extensions/nostr` | Nostr 协议 |

**功能/服务插件**

| 插件 | 说明 |
|------|------|
| `extensions/memory-core` | 对话记忆 |
| `extensions/memory-lancedb` | 向量存储记忆 |
| `extensions/browser` | 浏览器工具 |
| `extensions/elevenlabs` | TTS 语音合成 |
| `extensions/deepgram` | 语音转文字 |
| `extensions/fal` | 图像生成 |
| `extensions/firecrawl` | 网页抓取 |
| `extensions/exa` / `extensions/tavily` | 搜索 |
| `extensions/diffs` | 代码 Diff |

---

## 关键架构模式

### 1. 依赖注入：`RuntimeEnv`

几乎所有核心函数都接受 `RuntimeEnv` 参数，而不直接调用 `console.log` / `process.exit`，便于测试和环境隔离：

```typescript
// src/runtime.ts
export type RuntimeEnv = {
  log: (...args: unknown[]) => void;
  error: (...args: unknown[]) => void;
  exit: (code: number) => void;
};
```

### 2. 动态加载边界

插件通过 `*.runtime.ts` 文件做懒加载边界，避免混用 `await import()` 和静态 `import`，防止打包工具产生 `[INEFFECTIVE_DYNAMIC_IMPORT]` 警告。

### 3. 安全优先：Allow-From 策略

每条入站消息都经过 `allow-from.ts` 检查，默认只允许白名单内的账号/群组触发 Agent，防止陌生人滥用。

### 4. 插件命名约定

每个插件都有一个 Canonical ID（如 `telegram`、`anthropic`），与目录名、npm 包名、`openclaw.plugin.json:id` 保持一致，由 invariant 测试守护。

---

## 数据流（消息从 Telegram 到 AI 再回 Telegram）

```
Telegram 消息
    │
    ▼  extensions/telegram 接收
    │
    ▼  src/channels/ 标准化为 InboundEnvelope
    │
    ▼  src/routing/ 解析 → AgentId + SessionKey
    │
    ▼  src/agents/ 加载对话历史 + 构造 Prompt
    │
    ▼  extensions/anthropic（或其他 LLM）调用模型 API
    │
    ▼  工具调用（浏览器/搜索/代码执行…）
    │
    ▼  生成最终回复
    │
    ▼  src/channels/draft-stream-loop.ts 流式发送
    │
    ▼  extensions/telegram 调用 Bot API 发回消息
```

---

## 总结

OpenClaw 的架构设计核心思想是**"核心精简、能力外置"**：

- **Core 只做核心四件事**：路由、Session、通道抽象、安全控制
- **所有扩展能力皆为插件**：LLM Provider、消息平台、功能工具都是可插拔的 npm 包
- **严格的 SDK 接口边界**：Plugin SDK 保证插件不会侵入核心稳定性
- **TypeScript 全栈**：Vitest 测试 + Oxlint 代码规范，保证可维护性
