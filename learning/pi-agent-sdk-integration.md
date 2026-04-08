# OpenClaw 中 Pi Agent SDK 的核心功能与集成方式

## 一、概述

OpenClaw 的 Agents 层并非从零构建 Agent 运行时，而是**嵌入式集成**了 `@mariozechner/pi-coding-agent`（及其底层 `@mariozechner/pi-agent-core`、`@mariozechner/pi-ai`）这套 SDK。OpenClaw 不是将 Pi 作为子进程或 RPC 服务运行，而是通过 SDK 直接在进程内创建 `AgentSession`，从而获得对会话生命周期的**完全控制**。

这套 SDK 提供了四个包：

| 包名 | 版本 | 核心职责 |
|------|------|---------|
| `@mariozechner/pi-ai` | 0.61.1 | LLM 抽象层：`Model`、`streamSimple`、消息类型、Provider API |
| `@mariozechner/pi-agent-core` | 0.61.1 | Agent 循环引擎、工具执行框架、`AgentMessage` 类型系统 |
| `@mariozechner/pi-coding-agent` | 0.61.1 | 高级 SDK：`createAgentSession`、`SessionManager`、`AuthStorage`、`ModelRegistry`、内置编码工具 |
| `@mariozechner/pi-tui` | 0.61.1 | 终端 UI 组件（用于 OpenClaw 的本地 TUI 模式） |

---

## 二、`pi-agent-core` 提供的核心抽象

`pi-agent-core` 是最底层的包，提供了 Agent 系统的**基础类型契约**。OpenClaw 在 **146+ 处**导入了它的类型。

### 2.1 核心类型清单

| 类型 | 用途 | 导入文件数 |
|------|------|-----------|
| **`AgentMessage`** | 会话消息的统一类型（user/assistant/tool_result 等） | ~30+ 文件 |
| **`AgentTool`** | 工具定义接口（name、description、parameters、execute） | ~10 文件 |
| **`AgentToolResult`** | 工具执行结果（content 数组） | ~25+ 文件 |
| **`AgentToolUpdateCallback`** | 工具执行进度回调 | 3 文件 |
| **`StreamFn`** | LLM 流式传输函数签名 | ~6 文件 |
| **`AgentEvent`** | 智能体生命周期事件 | 2 文件 |

### 2.2 `AgentMessage` — 最核心的类型

`AgentMessage` 是整个系统中使用最广泛的类型，贯穿了：

- **会话历史存储与回放**：`SessionManager` 持久化的消息格式
- **上下文引擎**：`context-engine/types.ts` 中的上下文消息表示
- **Compaction**：`compaction.ts` 中的对话压缩输入/输出
- **Turn 验证**：`pi-embedded-helpers/turns.ts` 中的消息顺序校验
- **Bootstrap 注入**：`pi-embedded-helpers/bootstrap.ts` 中的初始消息构造
- **图像处理**：`pi-embedded-helpers/images.ts` 中的图像消息解析
- **Tool Call ID 清洗**：`tool-call-id.ts` 中的跨 provider 兼容处理

### 2.3 `AgentTool` / `AgentToolResult` — 工具系统的基石

`AgentTool` 定义了工具的标准接口，OpenClaw 用 `AnyAgentTool` 作为类型别名：

```typescript
// src/agents/pi-tools.types.ts
import type { AgentTool } from "@mariozechner/pi-agent-core";
export type AnyAgentTool = AgentTool<any, unknown>;
```

`AgentToolResult` 在所有渠道的 action runtime 中使用（Discord、Slack、Telegram、WhatsApp、Matrix），作为工具执行结果的统一返回格式。

### 2.4 `StreamFn` — LLM 流式传输的统一签名

`StreamFn` 是 LLM 流式调用的函数签名类型，OpenClaw 用它来适配不同的 provider：

- `ollama-stream.ts` — Ollama 原生 API 流式实现
- `custom-api-registry.ts` — 自定义 API 注册
- `anthropic-payload-log.ts` — Anthropic 请求日志
- `extensions/xai/stream.ts` — xAI provider 流适配
- `extensions/openrouter/index.ts` — OpenRouter 流适配

### 2.5 `AgentEvent` — 智能体生命周期事件

`AgentEvent` 定义了 Agent 运行过程中的事件类型，在 `pi-embedded-subscribe.handlers.compaction.ts` 中用于处理自动 compaction 事件：

```typescript
import type { AgentEvent } from "@mariozechner/pi-agent-core";

export function handleAutoCompactionStart(ctx: EmbeddedPiSubscribeContext) {
  ctx.state.compactionInFlight = true;
  // ...
}
```

---

## 三、`pi-coding-agent` 提供的高级 SDK 功能

`pi-coding-agent` 是上层 SDK，提供了完整的 Agent 会话管理能力。OpenClaw 在 **68+ 处**导入了它。

### 3.1 核心功能清单

| 导入项 | 类型 | 核心用途 |
|--------|------|---------|
| **`createAgentSession`** | 函数 | 创建嵌入式 Agent 会话 — **最核心的入口** |
| **`SessionManager`** | 类 | 会话文件持久化管理（open/save/compact） |
| **`AuthStorage`** | 类 | 模型 API 密钥/认证存储 |
| **`ModelRegistry`** | 类 | 模型注册表（已知模型、能力、上下文窗口） |
| **`DefaultResourceLoader`** | 类 | 扩展工厂的资源加载器 |
| **`SettingsManager`** | 类 | Pi 项目设置管理 |
| **`estimateTokens`** | 函数 | Token 估算（用于 compaction 预算计算） |
| **`CURRENT_SESSION_VERSION`** | 常量 | 会话文件版本号（兼容性检查） |
| **`codingTools`** | 值 | Pi 内置编码工具集合（read/write/edit/bash） |
| **`readTool` / `createReadTool`** | 值/函数 | 内置 read 工具实例/创建函数 |
| **`loadSkillsFromDir`** | 函数 | 从目录加载 Skill 定义 |
| **`formatSkillsForPrompt`** | 函数 | 将 Skill 格式化为 system prompt 文本 |
| **`AgentSession`** | type | Agent 会话对象类型 |
| **`SessionEntry`** | type | 会话条目数据结构 |
| **`CompactionEntry`** | type | Compaction 条目 |
| **`ToolDefinition`** | type | SDK 内部工具定义格式 |
| **`Skill`** | type | Skill 定义结构 |
| **`ExtensionFactory`** | type | 扩展工厂接口 |
| **`ExtensionAPI`** | type | 扩展 API 接口 |
| **`ExtensionContext`** | type | 扩展上下文 |
| **`ContextEvent`** | type | 上下文事件 |
| **`FileOperations`** | type | 文件操作接口 |

### 3.2 `createAgentSession()` — 最核心的入口

这是 OpenClaw 使用 Pi SDK 的**核心入口点**，在两个关键文件中被调用：

#### 3.2.1 `attempt.ts` 中的主会话创建

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts 第 2183 行
({ session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage: params.authStorage,
  modelRegistry: params.modelRegistry,
  model: params.model,
  thinkingLevel: mapThinkingLevel(params.thinkLevel),
  tools: builtInTools,        // Pi 内置工具（经 OpenClaw 替换后）
  customTools: allCustomTools, // OpenClaw 自定义工具
  sessionManager,
  settingsManager,
  resourceLoader,
}));
```

#### 3.2.2 `compact.ts` 中的 compaction 会话创建

```typescript
// src/agents/pi-embedded-runner/compact.ts 第 1005 行
const { session } = await createAgentSession({
  cwd: effectiveWorkspace,
  agentDir,
  // ... 类似参数
});
```

### 3.3 `AgentSession` 对象 — 会话的完整控制面

`createAgentSession()` 返回的 `session` 对象是 OpenClaw 控制 Agent 行为的核心手柄，其 API 包括：

| API | 用途 |
|-----|------|
| `session.subscribe(handler)` | 订阅会话事件流（消息、工具、compaction） |
| `session.agent.streamFn` | 获取/替换 LLM 流式传输函数 |
| `session.agent.replaceMessages(msgs)` | 替换会话消息历史（截断、清洗后） |
| `session.agent.steer(msg)` | 向会话注入引导消息（排队追加用户消息） |
| `session.agent.setTransport(t)` | 设置传输方式（HTTP/WS） |
| `session.agent.transport` | 获取当前传输方式 |
| `session.messages` | 访问消息历史数组 |
| `session.run(prompt)` / `session.prompt(text, opts)` | 运行 Agent（发送 prompt 并启动工具循环） |
| `session.steer(text)` | 队列消息（在运行中追加用户消息） |
| `session.abort()` | 中止当前运行 |
| `session.isCompacting` | 检查是否正在 compaction |
| `session.abortCompaction()` | 中止 compaction |
| `session.sendCustomMessage(msg)` | 发送自定义消息 |
| `session.isStreaming` | 检查是否正在流式输出 |
| `session.sessionId` | 获取会话 ID |
| `session.sessionFile` | 获取会话文件路径 |
| `session.dispose()` | 释放资源 |

### 3.4 `SessionManager` — 会话持久化管理器

`SessionManager` 是 Pi SDK 中负责会话文件读写的核心类，在 OpenClaw 中广泛使用：

```typescript
// 打开会话文件
const sessionManager = SessionManager.open(params.sessionFile);

// 用于：
// - 读取/写入会话历史（JSONL 格式）
// - Session 截断（session-truncation.ts）
// - Tool result 截断（tool-result-truncation.ts）
// - Transcript 重写（transcript-rewrite.ts）
// - 会话导出（commands-export-session.ts）
// - 会话分叉（session-fork.runtime.ts）
```

### 3.5 `codingTools` — 内置编码工具集

Pi SDK 提供了一组基础编码工具（`codingTools`），OpenClaw 以此为基础进行替换和增强：

```typescript
// src/agents/pi-tools.ts
import { codingTools, createReadTool, readTool } from "@mariozechner/pi-coding-agent";

// 遍历 codingTools，按名称替换：
const base = (codingTools as AnyAgentTool[]).flatMap((tool) => {
  if (tool.name === readTool.name) {
    // → 替换为 OpenClaw 自定义的 read 工具（加入图像清理、上下文窗口限制）
    const freshReadTool = createReadTool(workspaceRoot);
    return [createOpenClawReadTool(freshReadTool, { ... })];
  }
  if (tool.name === "bash" || tool.name === "exec") {
    return []; // → 完全移除，用 OpenClaw 的 createExecTool 替代
  }
  if (tool.name === "write") {
    return [createHostWorkspaceWriteTool(workspaceRoot, { ... })]; // → 替换
  }
  if (tool.name === "edit") {
    return [createHostWorkspaceEditTool(workspaceRoot, { ... })]; // → 替换
  }
  return [tool]; // 其他工具保留
});
```

### 3.6 Skill 系统

OpenClaw 使用 Pi SDK 的 Skill 加载和格式化能力：

```typescript
// src/agents/skills/bundled-context.ts
import { loadSkillsFromDir } from "@mariozechner/pi-coding-agent";

// src/agents/skills/workspace.ts
import { formatSkillsForPrompt, loadSkillsFromDir, type Skill } from "@mariozechner/pi-coding-agent";
```

### 3.7 Extension 系统

OpenClaw 通过 Pi SDK 的扩展机制注入自定义行为：

```typescript
// src/agents/pi-extensions/compaction-safeguard.ts
import type { ExtensionAPI, FileOperations } from "@mariozechner/pi-coding-agent";

// src/agents/pi-extensions/context-pruning/extension.ts
import type { ContextEvent, ExtensionAPI, ExtensionContext } from "@mariozechner/pi-coding-agent";

export default function contextPruningExtension(api: ExtensionAPI): void {
  api.on("context", (event: ContextEvent, ctx: ExtensionContext) => {
    // 上下文修剪逻辑
  });
}
```

注册扩展通过 `ExtensionFactory` 和 `DefaultResourceLoader` 完成：

```typescript
const extensionFactories = buildEmbeddedExtensionFactories({ ... });
const resourceLoader = new DefaultResourceLoader({
  cwd: resolvedWorkspace,
  agentDir,
  settingsManager,
  extensionFactories,  // ← 注入自定义扩展
});
await resourceLoader.reload();
```

---

## 四、`pi-ai` 提供的 LLM 抽象

`pi-ai` 包提供了 LLM 调用的底层抽象：

| 导入项 | 用途 |
|--------|------|
| `streamSimple` | 标准 LLM 流式调用函数（默认 StreamFn 实现） |
| `Model<Api>` | 模型定义类型（包含 provider、contextWindow、api 等） |
| `Api` | API 类型标识（"anthropic-messages"、"openai"、"ollama" 等） |
| `AssistantMessage` | 助手消息类型 |

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts
import { streamSimple } from "@mariozechner/pi-ai";

// 默认情况下使用 streamSimple 作为 StreamFn
activeSession.agent.streamFn = streamSimple;
```

---

## 五、OpenClaw 对 Pi SDK 的核心定制

### 5.1 `StreamFn` 洋葱模型包装链

OpenClaw 最深度的定制在于对 `session.agent.streamFn` 的多层包装。创建会话后，`attempt.ts` 会根据 provider 和配置，在原始 `streamFn` 外层层层包裹中间件：

```
原始 streamFn (streamSimple / ollamaStreamFn / openAIWebSocketStreamFn / anthropicVertexStreamFn)
  │
  ├── wrapOllamaCompatNumCtx()          — Ollama num_ctx 注入
  ├── cacheTrace.wrapStreamFn()         — 缓存追踪
  ├── dropThinkingBlocks()              — 清理 thinking block（Anthropic）
  ├── sanitizeToolCallIds()             — Tool Call ID 格式清洗
  ├── downgradeOpenAIFunctionCallReasoningPairs() — OpenAI 兼容处理
  ├── yieldAbortedResponse()            — sessions_yield 中止处理
  ├── wrapStreamFnSanitizeMalformedToolCalls()   — 工具名清洗
  ├── wrapStreamFnTrimToolCallNames()   — 工具名 trim
  ├── wrapStreamFnRepairMalformedToolCallArguments() — 参数修复（Anthropic）
  ├── wrapStreamFnDecodeXaiToolCallArguments()   — xAI HTML entity 解码
  └── anthropicPayloadLogger.wrapStreamFn()      — 请求日志
```

每一层都是 `(model, context, options) => ...` 签名的函数，形成洋葱模型。

### 5.2 工具格式转换：`AgentTool` ↔ `ToolDefinition`

Pi SDK 内部使用两种工具格式：
- `AgentTool`（pi-agent-core）：外部工具接口
- `ToolDefinition`（pi-coding-agent）：SDK 内部工具格式

`pi-tool-definition-adapter.ts` 负责在两者之间转换：

```typescript
// AgentTool[] → ToolDefinition[]
export function toToolDefinitions(tools: AnyAgentTool[]): ToolDefinition[] {
  return tools.map((tool) => ({
    name: tool.name,
    label: tool.label ?? tool.name,
    description: tool.description ?? "",
    parameters: tool.parameters,
    execute: async (...args) => {
      // 1. before-tool-call hook 检查
      // 2. 调用 tool.execute()
      // 3. 结果规范化 (normalizeToolExecutionResult)
      // 4. 错误处理 (buildToolExecutionErrorResult)
    },
  }));
}
```

### 5.3 事件订阅桥接：`subscribeEmbeddedPiSession()`

`subscribeEmbeddedPiSession()` 是 OpenClaw 与 Pi SDK 事件系统之间的桥梁。它调用 `session.subscribe()` 注册事件处理器，将 Pi SDK 的原始事件转换为 OpenClaw 的业务事件：

```typescript
// src/agents/pi-embedded-subscribe.ts
export function subscribeEmbeddedPiSession(params) {
  // ... 大量状态初始化（~600 行）
  
  // 核心：订阅 Pi session 的事件
  const sessionUnsubscribe = params.session.subscribe(
    createEmbeddedPiSessionEventHandler(ctx)
  );
  
  return {
    assistantTexts,      // 助手回复文本列表
    toolMetas,           // 工具执行元数据
    unsubscribe,         // 取消订阅
    isCompacting,        // compaction 状态
    waitForCompactionRetry, // 等待 compaction 完成
    getUsageTotals,      // Token 使用统计
    // ...
  };
}
```

事件处理器通过 `createEmbeddedPiSessionEventHandler()` 创建，处理 8 种事件类型：

```typescript
// src/agents/pi-embedded-subscribe.handlers.ts
export function createEmbeddedPiSessionEventHandler(ctx) {
  return (evt) => {
    switch (evt.type) {
      case "message_start":       handleMessageStart(ctx, evt);
      case "message_update":      handleMessageUpdate(ctx, evt);
      case "message_end":         handleMessageEnd(ctx, evt);
      case "tool_execution_start": handleToolExecutionStart(ctx, evt);
      case "tool_execution_update": handleToolExecutionUpdate(ctx, evt);
      case "tool_execution_end":  handleToolExecutionEnd(ctx, evt);
      case "agent_start":         handleAgentStart(ctx);
      case "agent_end":           handleAgentEnd(ctx);
      case "auto_compaction_start": handleAutoCompactionStart(ctx);
      case "auto_compaction_end": handleAutoCompactionEnd(ctx, evt);
    }
  };
}
```

### 5.4 System Prompt 覆盖

OpenClaw 生成自己的 system prompt 后，通过 `applySystemPromptOverrideToSession()` 注入到 Pi session，**覆盖** Pi SDK 的默认 system prompt：

```typescript
// 创建 session 后立即覆盖
({ session } = await createAgentSession({ ... }));
applySystemPromptOverrideToSession(session, systemPromptText);
```

### 5.5 Extension 注入

OpenClaw 通过 Pi SDK 的扩展机制注入了两个自定义扩展：

1. **compaction-safeguard**：在 compaction 过程中保护关键数据不被丢失
2. **context-pruning**：基于 cache-TTL 的上下文修剪，减少不必要的 token 消耗

---

## 六、Pi SDK 在 OpenClaw 中的使用矩阵

```
┌─────────────────────────────────────────────────────────────────────┐
│                          OpenClaw Agents 层                         │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ pi-ai (LLM 抽象)                                           │   │
│  │                                                             │   │
│  │  streamSimple ←──── 默认 StreamFn 实现                      │   │
│  │  Model<Api>   ←──── 模型定义（provider、contextWindow、api） │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ pi-agent-core (底层类型)                                    │   │
│  │                                                             │   │
│  │  AgentMessage    ←── 消息历史/上下文/compaction (30+ 文件)   │   │
│  │  AgentTool       ←── 工具定义接口 (AnyAgentTool 别名)       │   │
│  │  AgentToolResult ←── 工具执行结果 (25+ 文件，含所有渠道)     │   │
│  │  StreamFn        ←── LLM 流式函数签名 (6 文件)              │   │
│  │  AgentEvent      ←── 智能体事件 (compaction 处理)           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ pi-coding-agent (高级 SDK)                                  │   │
│  │                                                             │   │
│  │  createAgentSession() ←── 核心入口，创建 AgentSession       │   │
│  │  SessionManager       ←── 会话文件持久化 (15+ 文件)         │   │
│  │  AuthStorage          ←── API key 存储/认证管理              │   │
│  │  ModelRegistry        ←── 模型注册表/能力查询                │   │
│  │  DefaultResourceLoader ←── 扩展加载器                        │   │
│  │  SettingsManager      ←── 项目设置管理                       │   │
│  │  estimateTokens       ←── Token 估算                         │   │
│  │  codingTools          ←── 内置工具基础集合                    │   │
│  │  loadSkillsFromDir    ←── Skill 加载                         │   │
│  │  formatSkillsForPrompt ←── Skill 格式化                      │   │
│  │  ExtensionFactory/API ←── 扩展机制                           │   │
│  │  ToolDefinition       ←── SDK 内部工具格式                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ OpenClaw 定制层                                             │   │
│  │                                                             │   │
│  │  StreamFn 洋葱包装链 (10+ 层中间件)                          │   │
│  │  工具替换 (read/write/edit/bash → 自定义实现)                │   │
│  │  System Prompt 覆盖 (buildAgentSystemPrompt → override)     │   │
│  │  事件桥接 (subscribeEmbeddedPiSession)                      │   │
│  │  工具格式转换 (AgentTool ↔ ToolDefinition)                  │   │
│  │  自定义扩展 (compaction-safeguard + context-pruning)         │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 七、总结：Pi SDK 为 OpenClaw 提供了什么

Pi Agent SDK 为 OpenClaw 提供了 **五大核心能力**：

### 7.1 Agent 循环引擎
`createAgentSession()` + `session.prompt()` 提供了完整的 Agent 循环：发送 prompt → LLM 推理 → 工具调用 → 结果反馈 → 继续推理。OpenClaw 不需要自己实现这个循环。

### 7.2 会话持久化框架
`SessionManager` 提供了 JSONL 格式的会话文件读写、版本管理、compaction 支持。OpenClaw 在此基础上添加了截断、修复、导出等能力。

### 7.3 统一的类型系统
`AgentMessage`、`AgentTool`、`AgentToolResult` 等类型构成了跨渠道、跨 provider 的统一抽象。所有 Discord/Slack/Telegram/WhatsApp/Matrix 渠道都使用相同的工具结果类型。

### 7.4 可替换的 LLM 流式传输
`StreamFn` 签名允许 OpenClaw 无缝切换不同的 LLM provider（Anthropic、OpenAI、Ollama、Gemini、xAI、OpenRouter），并在流式传输层插入各种中间件（缓存、日志、清洗、兼容性修复）。

### 7.5 可扩展的工具与 Skill 框架
`codingTools` 提供了基础编码工具，`ExtensionFactory/API` 提供了扩展机制，`Skill` 系统提供了技能加载和提示注入。OpenClaw 在所有这些框架上进行了深度定制。

---

## 八、关键设计模式

### 8.1 嵌入式 SDK 模式
不是子进程，不是 RPC，而是直接在进程内创建 `AgentSession`。这给了 OpenClaw 对会话生命周期、工具执行、消息历史的**完全控制**。

### 8.2 StreamFn 洋葱模型
通过多层 `streamFn` 包装，在不修改 SDK 源码的情况下，对每次 LLM 请求进行拦截和定制。

### 8.3 工具替换策略
以 Pi 的 `codingTools` 为基础，**选择性替换**而非全部重写，保留了 SDK 的核心工具逻辑，同时注入 OpenClaw 特有的安全和功能增强。

### 8.4 事件驱动桥接
通过 `session.subscribe()` + `subscribeEmbeddedPiSession()` 建立事件驱动的响应处理链，将 SDK 的内部事件转化为 OpenClaw 的业务事件（block reply、reasoning stream、tool result 等）。

### 8.5 类型桥接适配器
`pi-tool-definition-adapter.ts` 作为两种工具格式之间的桥梁，使得 OpenClaw 可以使用自己的 `AgentTool` 接口定义工具，同时与 Pi SDK 内部的 `ToolDefinition` 格式兼容。
