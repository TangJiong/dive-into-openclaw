# OpenClaw Agents 层架构深度解读

## 一、整体架构概览

Agents 层是 OpenClaw 的"大脑"，位于 Gateway/Routing 层之下、Plugin SDK 层之上。其核心职责是：**接收经路由分发的消息，构建上下文，驱动 LLM 完成多轮对话与工具调用，并管理会话记忆**。

核心调用链路：

```
Gateway 消息入口
  → agentCommand() [agent-command.ts]
    → runWithModelFallback() [model fallback 层]
      → runAgentAttempt() [选择 runner]
        → runEmbeddedPiAgent() [pi-embedded-runner/run.ts - Agent Loop]
          → runEmbeddedAttempt() [attempt.ts - 单次执行]
            → createOpenClawCodingTools() [工具创建]
            → buildEmbeddedSystemPrompt() [Prompt 构建]
            → createAgentSession() + subscribe [LLM 交互]
```

### 关键源码文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/agents/agent-command.ts` | ~1334 | Agent 命令入口，消息→Agent 的桥梁 |
| `src/agents/system-prompt.ts` | ~704 | 系统提示词多段式组装 |
| `src/agents/pi-embedded-runner/run.ts` | ~1860 | Agent Loop 核心（while 循环 + 重试） |
| `src/agents/pi-embedded-runner/run/attempt.ts` | ~3241 | 单次执行尝试的完整实现 |
| `src/agents/pi-embedded-runner/system-prompt.ts` | ~107 | 嵌入式 system prompt 桥接 |
| `src/agents/pi-tools.ts` | ~635 | 工具创建与策略过滤 |
| `src/agents/openclaw-tools.ts` | ~282 | OpenClaw 特有工具聚合 |
| `src/agents/tools/common.ts` | ~369 | 工具基础类型与辅助函数 |
| `src/agents/memory-search.ts` | ~407 | Memory 搜索配置解析 |
| `src/agents/tools/memory-tool.ts` | ~321 | memory_search / memory_get 实现 |
| `extensions/memory-core/index.ts` | ~75 | Memory 插件入口 |
| `src/agents/compaction.ts` | ~465 | 对话历史压缩（compaction） |
| `src/agents/acp-spawn.ts` | ~933 | ACP Agent 协作协议 spawn |
| `src/agents/tool-call-id.ts` | ~310 | Tool call ID 跨 provider 兼容性 |

---

## 二、Prompt 构建机制

### 2.1 核心函数：`buildAgentSystemPrompt()`

文件位于 `src/agents/system-prompt.ts`，这是**整个系统提示词的组装中枢**。它采用"多段式拼接"策略，将十几个独立的 section 按序组装为最终的 system prompt：

```typescript
export function buildAgentSystemPrompt(params: {
  workspaceDir: string
  toolNames?: string[]
  toolSummaries?: Record<string, string>
  contextFiles?: EmbeddedContextFile[]
  skillsPrompt?: string
  promptMode?: PromptMode
  // ... 30+ 参数
})
```

### 2.2 组装的 Sections

| Section | 内容 | 条件 |
|---------|------|------|
| **Tooling** | 工具清单 + 摘要描述 | `promptMode !== "none"` |
| **Tool Call Style** | 工具调用格式规范 | 同上 |
| **Safety** | 安全规则（禁止私钥泄露等） | 同上 |
| **CLI Quick Reference** | OpenClaw CLI 常用命令速查 | full mode |
| **Skills (mandatory)** | 技能提示词注入 | `skillsPrompt` 非空 |
| **Memory Recall** | 记忆检索结果 | `memoryRecall` 非空 |
| **Model Aliases** | 模型别名映射表 | 有别名时 |
| **Workspace** | 工作目录信息 | full mode |
| **Sandbox** | 沙箱环境说明 | 沙箱模式 |
| **Authorized Senders** | 授权发送者列表 | full mode + 有数据 |
| **Reply Tags** | 回复目标标签 | 有 replyTags |
| **Messaging** | 消息渠道规范 | full mode |
| **Voice** | 语音交互规范 | 启用 TTS |
| **Reasoning Format** | 思维链格式（`<think>` 标签） | 启用 thinking |
| **Project Context** | 注入的上下文文件内容 | 有 contextFiles |
| **Silent Replies** | 静默回复机制 | full mode |
| **Heartbeats** | 心跳回复策略 | full mode |
| **Runtime** | 当前时间、版本等运行时信息 | 始终 |

### 2.3 三种 Prompt Mode

```typescript
type PromptMode = "full" | "minimal" | "none"
```

- **`full`**：主 Agent 使用，包含全部 section
- **`minimal`**：子 Agent 使用，省略 CLI reference、messaging、heartbeats 等
- **`none`**：最简模式，仅保留 skills、memory、context files 和 runtime 信息

### 2.4 从 attempt 到 system prompt 的桥接

在 `pi-embedded-runner/system-prompt.ts` 中，`buildEmbeddedSystemPrompt()` 作为桥接函数，将运行时参数（工具名、技能、上下文文件等）转换为 `buildAgentSystemPrompt()` 的调用参数：

```typescript
export function buildEmbeddedSystemPrompt(params: {
  workspaceDir: string
  tools: AnyAgentTool[]
  config: OpenClawConfig
  contextFiles: EmbeddedContextFile[]
  skillsPrompt: string | undefined
  memoryRecall: MemoryRecallResult | undefined
  promptMode: PromptMode
  // ...
}): string {
  const toolNames = params.tools.map((t) => t.name)
  const toolSummaries: Record<string, string> = {}
  for (const t of params.tools) {
    if (t.description) toolSummaries[t.name] = t.description
  }
  return buildAgentSystemPrompt({
    workspaceDir: params.workspaceDir,
    toolNames,
    toolSummaries,
    contextFiles: params.contextFiles,
    skillsPrompt: params.skillsPrompt,
    // ...
  })
}
```

生成的 prompt 通过 `applySystemPromptOverrideToSession()` 注入到 pi-agent session 中，**覆盖**底层 pi-coding-agent 的默认 system prompt。

### 2.5 Bootstrap 文件与 Context Files

在 `attempt.ts` 中，Agent 启动前会加载 **bootstrap 文件**（如 `AGENTS.md`、`CLAUDE.md` 等项目指导文件），这些文件会作为 `contextFiles` 注入到 system prompt 的 `Project Context` section 中：

```typescript
// attempt.ts 中的 bootstrap 文件加载逻辑
const bootstrapFiles = await loadBootstrapFiles(workspaceDir, config)
// 这些文件最终会出现在 system prompt 的 <project_context> 标签中
```

---

## 三、Agent Loop（代理循环）

### 3.1 外层：Model Fallback

`agent-command.ts` 中的 `runWithModelFallback()` 提供**模型级别的容错**：

```typescript
// agent-command.ts 中的关键调用
const result = await runWithModelFallback(
  async (modelConfig) => {
    return runAgentAttempt(modelConfig, /* ... */)
  },
  modelConfigs,  // 主模型 + 备选模型列表
)
```

当主模型抛出 `FailoverError` 时，自动切换到下一个备选模型重试。

### 3.2 核心：`runEmbeddedPiAgent()` 的 while(true) 循环

这是 Agent Loop 的**心脏**，位于 `src/agents/pi-embedded-runner/run.ts`。其重试循环处理了极其复杂的故障场景：

```
while (true) {  // 最大迭代: BASE(24) + profileCount * 8，范围 32-160
  try {
    ① 选择 Auth Profile（API key 轮换）
    ② 可选：Runtime Auth Refresh（OAuth token 刷新）
    ③ 调用 runEmbeddedAttempt() 执行单次 Agent 交互
    ④ 成功 → break 退出循环
  } catch (error) {
    根据错误类型决定重试策略：

    ├── ContextWindowOverflowError
    │   → compaction（最多3次）→ 重试
    │
    ├── TimeoutError
    │   → timeout compaction（最多2次）→ 重试
    │
    ├── OversizedToolResultError
    │   → truncate tool result → 重试
    │
    ├── ThinkingUnsupportedError / ThinkingBudgetError
    │   → 降级 thinking level（xhigh→high→off）→ 重试
    │
    ├── AuthenticationError / RateLimitError
    │   → advanceAuthProfile() 切换 API key → 重试
    │
    ├── OverloadError
    │   → 指数退避（2^n 秒，最大64秒）→ 重试
    │
    ├── ModelNotFoundError（特定模型不可用）
    │   → 抛出 FailoverError → 触发外层 model fallback
    │
    └── 其他未知错误
        → 达到最大重试次数后抛出
  }
}
```

### 3.3 Auth Profile 轮换机制

OpenClaw 支持配置多个 API key（auth profiles），循环使用：

```typescript
function advanceAuthProfile(state) {
  state.profileIndex = (state.profileIndex + 1) % state.profiles.length
  // 带 cooldown 机制：刚失败的 profile 短时间内不会被重新选中
}
```

每个 profile 有独立的 cooldown 时间戳，避免在短时间内重复使用已失败的 key。

### 3.4 Compaction（上下文压缩）

当对话历史超出模型的 context window 时，触发 compaction：

```typescript
// compaction.ts 核心逻辑
export async function summarizeInStages(messages, options) {
  // 1. 将消息按 token 份额分割为多个 chunk
  const chunks = splitByTokenShare(messages, chunkTokenBudget)

  // 2. 对每个 chunk 生成摘要
  const chunkSummaries = await Promise.all(
    chunks.map(chunk => summarizeChunk(chunk, model))
  )

  // 3. 合并所有摘要为最终的 compacted context
  return mergedSummary
}
```

compaction 的安全边际设置为 `SAFETY_MARGIN = 1.2`（20% buffer），确保压缩后的上下文不会再次溢出。

### 3.5 重试上限计算

```typescript
const MAX_ITERATIONS = BASE_RUN_RETRY_ITERATIONS + profileCount * PROFILE_RETRY_MULTIPLIER
// BASE_RUN_RETRY_ITERATIONS = 24
// PROFILE_RETRY_MULTIPLIER = 8
// 1 个 profile: 24 + 1*8 = 32 次
// 最多 17 个 profile: 24 + 17*8 = 160 次
```

---

## 四、Memory 管理

OpenClaw 的 Memory 系统采用**双层架构**：短期记忆（Session 级别）+ 长期记忆（跨 Session 持久化）。

### 4.1 短期记忆：Session Transcript

每次 Agent 对话的完整消息历史通过 `SessionManager` 管理，以文件形式持久化到磁盘：

- **消息历史存储**：JSON 文件
- **Tool result truncation**：超大工具结果截断
- **Context window guard**：上下文窗口守护
- **Compaction**：对话摘要压缩

当对话过长时，compaction 机制会将早期对话压缩为摘要，保留最近的交互细节。

### 4.2 长期记忆：memory-core 插件 + LanceDB 向量存储

长期记忆通过插件机制实现，核心入口在 `extensions/memory-core/index.ts`：

```typescript
// Memory 插件注册的三个核心组件：
plugin.registerPromptSection(/* memory prompt section */)  // 注入记忆相关的 prompt 指令
plugin.registerTool("memory_search", /* ... */)            // 语义搜索工具
plugin.registerTool("memory_get", /* ... */)               // 按路径读取工具
plugin.registerCLI("memory", /* ... */)                    // CLI 管理命令
```

### 4.3 `memory_search` 工具实现

位于 `src/agents/tools/memory-tool.ts`：

```typescript
export function createMemorySearchTool(config) {
  return {
    name: "memory_search",
    description: "Search through memory files using semantic similarity",
    parameters: {
      query: { type: "string", description: "Natural language search query" },
      limit: { type: "number", description: "Max results to return" },
    },
    async execute({ query, limit }) {
      // 1. 搜索范围：MEMORY.md + memory/*.md 文件
      // 2. 使用向量 embedding 进行语义搜索
      // 3. 支持 hybrid search（向量 + 关键词）
      // 4. 支持 MMR（最大边际相关性）去重
      // 5. 支持 temporal decay（时间衰减）
      // 6. 返回带 citations 的搜索结果
    }
  }
}
```

### 4.4 Memory 搜索配置

`src/agents/memory-search.ts` 中的 `resolveMemorySearchConfig()` 负责合并全局和 Agent 级别的配置：

```typescript
// 支持的 embedding provider
type EmbeddingProvider = "openai" | "gemini" | "voyage" | "mistral" | "ollama" | "local"

// 配置项包括：
{
  store: "sqlite",          // 存储后端
  chunking: { /* ... */ },  // 文本分块策略
  sync: { /* ... */ },      // 同步策略
  query: {
    hybridSearch: true,     // 混合搜索
    mmr: true,              // 最大边际相关性
    temporalDecay: 0.5,     // 时间衰减系数
  },
  cache: { /* ... */ },     // 缓存策略
}
```

### 4.5 `memory_get` 工具

提供精确的记忆文件读取能力：

```typescript
// memory_get：按路径+行号读取 memory 文件片段
export function createMemoryGetTool() {
  return {
    name: "memory_get",
    parameters: {
      path: { type: "string" },   // memory 文件路径
      line: { type: "number" },    // 起始行号
      count: { type: "number" },   // 读取行数
    }
  }
}
```

---

## 五、Tool Call（工具调用）

### 5.1 工具创建：`createOpenClawCodingTools()`

位于 `src/agents/pi-tools.ts`，这是工具系统的**总装车间**，创建 30+ 个工具：

```typescript
export function createOpenClawCodingTools(params) {
  const tools: AnyAgentTool[] = []

  // ① 基础文件操作工具（来自 pi-coding-agent）
  tools.push(...createBaseCodingTools())  // read, write, edit

  // ② 命令执行工具
  tools.push(createExecTool())           // bash/shell 执行
  tools.push(createProcessTool())        // 进程管理

  // ③ apply_patch（OpenAI 专用补丁工具）
  if (isOpenAIProvider) {
    tools.push(createApplyPatchTool())
  }

  // ④ 渠道特定工具
  tools.push(...channelTools)            // 由消息渠道提供

  // ⑤ OpenClaw 特有工具
  tools.push(...createOpenClawTools())   // 见下文

  // ⑥ 插件工具
  tools.push(...pluginTools)             // 由已安装插件提供

  return tools
}
```

### 5.2 OpenClaw 特有工具集

`src/agents/openclaw-tools.ts` 中的 `createOpenClawTools()` 聚合了所有特色工具：

| 工具类别 | 工具名 | 功能 |
|---------|--------|------|
| **浏览器** | `browser` | 网页浏览与交互 |
| **画布** | `canvas` | 画布操作 |
| **节点** | `nodes` | 节点管理 |
| **定时任务** | `cron` | Cron 任务管理 |
| **消息** | `message` | 消息发送 |
| **语音** | `tts` | 文字转语音 |
| **图像** | `image_generate` | 图像生成 |
| **网关** | `gateway` | 网关操作 |
| **会话管理** | `sessions_list/history/send/spawn/yield` | 多会话管理 |
| **子代理** | `subagents` | 子 Agent 管理 |
| **搜索** | `web_search`, `web_fetch` | Web 搜索与页面获取 |
| **记忆** | `memory_search`, `memory_get` | 记忆检索（由插件注入） |
| **图像理解** | `image` | 图像分析 |
| **PDF** | `pdf` | PDF 处理 |

### 5.3 多层策略过滤管道（Tool Policy Pipeline）

工具创建后，必须经过**七层策略过滤**才能最终暴露给 LLM：

```
原始工具集（30+ 工具）
  │
  ├── ① Profile Policy    → 基于 auth profile 的工具白/黑名单
  ├── ② Provider Policy    → 基于模型提供商的工具限制
  ├── ③ Global Policy      → 全局工具策略
  ├── ④ Agent Policy       → Agent 级别的工具策略
  ├── ⑤ Group Policy       → 群组/频道级别的工具策略
  ├── ⑥ Sandbox Policy     → 沙箱环境的工具限制（如禁用 exec）
  └── ⑦ Subagent Policy    → 子 Agent 的工具限制（通常更严格）
  │
  附加过滤：
  ├── Owner-Only Filter    → 某些工具仅 owner 可用
  ├── Message Provider     → 基于消息来源渠道的过滤
  └── Model Provider       → 基于模型能力的过滤（如非 vision 模型移除图像工具）
  │
  ▼
最终工具集（暴露给 LLM）
```

### 5.4 工具执行与结果处理

工具的基础类型定义在 `src/agents/tools/common.ts`：

```typescript
// 工具基础接口
type AnyAgentTool = {
  name: string
  description: string
  parameters: JSONSchema
  execute: (params: Record<string, unknown>) => Promise<ToolResult>
}

// 工具结果构造函数
function textResult(text: string): ToolResult
function jsonResult(data: unknown): ToolResult
function imageResult(base64: string, mimeType: string): ToolResult

// 工具错误类型
class ToolInputError extends Error { /* 参数校验错误 */ }
class ToolAuthorizationError extends Error { /* 权限不足错误 */ }
```

### 5.5 Tool Call ID 兼容性处理

不同 LLM 提供商对 tool call ID 的格式要求不同，`src/agents/tool-call-id.ts` 负责处理这一差异：

```typescript
// 两种 ID 格式模式
type ToolCallIdMode = "strict" | "strict9"

function sanitizeToolCallIdsForCloudCodeAssist(messages, mode) {
  // strict: 标准格式（如 Anthropic）
  // strict9: 9字符限制格式（如某些 OpenAI 端点）
  // 确保跨 provider 的 tool call ID 兼容性
}
```

---

## 六、ACP（Agent Cooperation Protocol）

ACP 是 Agent 间的**协作协议**，通过 `src/agents/acp-spawn.ts` 实现：

```typescript
export async function spawnAcpDirect(params) {
  // 1. 安全检查：沙箱会话不能 spawn ACP
  // 2. Agent policy 检查
  // 3. 创建隔离的 ACP 会话
  //    - 独立的 session ID
  //    - 可选的 thread binding（绑定到特定对话线程）
  //    - stream-to-parent（将子 Agent 输出流式传回父 Agent）
  // 4. 支持两种模式：
  //    - persistent：持久会话，可多次交互
  //    - oneshot：一次性任务，完成后销毁
}
```

---

## 七、架构设计亮点总结

### 7.1 极致的容错性

从 Auth Profile 轮换 → Thinking 降级 → Context Compaction → Model Fallback，**四层防御**确保 Agent 在各种故障下仍能正常运行。

### 7.2 可插拔的工具系统

核心工具 + 插件工具 + 渠道工具的**三层工具架构**，配合七层策略过滤管道，实现了灵活且安全的工具管理。

### 7.3 双层记忆系统

短期（Session Compaction）+ 长期（向量搜索）的组合，既保证了当前对话的连贯性，又支持跨会话的知识积累。

### 7.4 多段式 Prompt 组装

将 system prompt 拆分为十几个独立 section，通过 PromptMode 和条件判断灵活组合，避免了单一巨型 prompt 的维护困难。

### 7.5 生产级的重试策略

最大 160 次重试、指数退避、cooldown 机制、context overflow 自动压缩，体现了在真实生产环境中处理 LLM API 不稳定性的深厚经验。

---

## 八、核心流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     agentCommand()                          │
│  加载配置 → 解析 session → 选择模型 → 构建 skills snapshot  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              runWithModelFallback()                          │
│  主模型 → [FailoverError] → 备选模型1 → 备选模型2 → ...    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           runEmbeddedPiAgent() [while(true)]                │
│                                                             │
│  ┌─ Auth Profile 选择 ──────────────────────────────────┐   │
│  │  profile[0] → cooldown → profile[1] → ... → 轮换    │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│  ┌─ runEmbeddedAttempt() ───────────────────────────────┐   │
│  │  ① 创建工具集（createOpenClawCodingTools）           │   │
│  │  ② 工具策略过滤（7层 pipeline）                      │   │
│  │  ③ 构建 System Prompt（buildEmbeddedSystemPrompt）   │   │
│  │  ④ 创建 Agent Session                                │   │
│  │  ⑤ 订阅流式响应（subscribeEmbeddedPiSession）       │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │                                   │
│                    错误处理                                  │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │ ContextOverflow → compaction (最多3次)              │    │
│  │ Timeout → timeout compaction (最多2次)              │    │
│  │ OversizedToolResult → truncate result               │    │
│  │ ThinkingError → 降级 thinking level                 │    │
│  │ AuthError/RateLimit → advanceAuthProfile            │    │
│  │ Overload → 指数退避                                 │    │
│  │ ModelNotFound → 抛出 FailoverError (外层处理)       │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```
