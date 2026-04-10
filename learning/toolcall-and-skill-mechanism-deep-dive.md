# OpenClaw Tool Call 与 Skill 机制深度解读

## 一、总览

OpenClaw 的 Tool Call 和 Skill 是两套互补的能力扩展机制：

- **Tool Call**：Agent 在推理过程中直接调用的**结构化函数**，每个 tool 有 name、description、parameters（JSON Schema）和 execute 实现
- **Skill**：以 `SKILL.md` 文件形式存在的**知识注入**，通过 system prompt 告知 Agent 有哪些可用技能，Agent 在需要时用 `read` 工具读取 SKILL.md 获取详细指令

两者的关系：**Tool 是"手"，Skill 是"脑中的知识索引"**。Tool 通过 LLM 的 function calling 协议执行；Skill 通过 prompt 注入 + 文件读取实现。

---

## 二、Tool Call 机制

### 2.1 工具类型体系

OpenClaw 的工具分为四大类来源，最终统一为 `AnyAgentTool` 接口：

```typescript
// src/agents/tools/common.ts + src/agents/pi-tools.types.ts
type AnyAgentTool = AgentTool<any, unknown> & {
  ownerOnly?: boolean;       // 是否仅限 owner 使用
  displaySummary?: string;   // 工具摘要（用于 system prompt）
};

// 底层接口来自 @mariozechner/pi-agent-core
interface AgentTool {
  name: string;
  label?: string;
  description: string;
  parameters: JSONSchema;     // JSON Schema 定义参数
  execute: (toolCallId, params, signal, onUpdate) => Promise<AgentToolResult>;
}
```

### 2.2 工具创建：四层来源

核心入口函数 `createOpenClawCodingTools()`（`src/agents/pi-tools.ts`）从四个来源汇聚工具：

```
┌─────────────────────────────────────────────────────────────┐
│               createOpenClawCodingTools()                    │
│                                                             │
│  ① 基础编码工具 (来自 pi-coding-agent SDK)                   │
│     codingTools → 选择性替换 read/write/edit/bash            │
│                                                             │
│  ② exec 和 process 工具                                     │
│     createExecTool() + createProcessTool()                  │
│                                                             │
│  ③ OpenClaw 专有工具 (约 20 个)                              │
│     createOpenClawTools() → browser/canvas/nodes/cron/      │
│     message/tts/image_generate/gateway/sessions_*/          │
│     web_search/web_fetch/image/pdf/subagents                │
│                                                             │
│  ④ 插件工具 (extensions 提供)                                │
│     resolvePluginTools() → memory_search/memory_get/        │
│     diffs/firecrawl/exa/tavily/...                          │
└─────────────────────────────────────────────────────────────┘
```

#### ① 基础编码工具的替换策略

OpenClaw 不是简单使用 Pi SDK 的工具，而是**选择性替换**：

```typescript
// src/agents/pi-tools.ts 第 376-418 行
const base = (codingTools as AnyAgentTool[]).flatMap((tool) => {
  if (tool.name === readTool.name) {
    // → 沙箱环境用 createSandboxedReadTool
    // → 普通环境用 createOpenClawReadTool（加入图像清理、上下文窗口限制）
    if (sandboxRoot) {
      return [createSandboxedReadTool({ root: sandboxRoot, bridge: sandboxFsBridge })];
    }
    return [createOpenClawReadTool(freshReadTool, { modelContextWindowTokens })];
  }
  if (tool.name === "bash" || tool.name === "exec") {
    return [];  // 完全移除，后面用 OpenClaw 自己的 createExecTool 替代
  }
  if (tool.name === "write") {
    if (sandboxRoot) return [];  // 沙箱环境另建
    return [createHostWorkspaceWriteTool(workspaceRoot, { workspaceOnly })];
  }
  if (tool.name === "edit") {
    if (sandboxRoot) return [];
    return [createHostWorkspaceEditTool(workspaceRoot, { workspaceOnly })];
  }
  return [tool];  // 其他工具保留
});
```

#### ② Exec 工具的安全架构

`createExecTool()`（`src/agents/bash-tools.exec.ts`）是最复杂的单个工具，649 行代码，因为 shell 执行是最大的安全面：

**三种执行主机 (host)**：
- `sandbox`：Docker 容器内执行（最安全）
- `gateway`：网关主机本地执行（需审批）
- `node`：远程 Node 主机执行

**四级安全等级 (security)**：
- `deny`：默认拒绝所有命令
- `allowlist`：仅允许安全二进制（safeBins）列表中的命令
- `ask`：需要人工审批
- `full`：完全信任（仅 elevated 模式）

**关键安全机制**：
1. **Shell 变量注入检测**：`validateScriptFileForShellBleed()` 在执行前检查 Python/JS 脚本中是否有 shell 变量泄漏（如 `$HOME` 出现在 .py 文件中）
2. **环境变量沙箱化**：`sanitizeHostExecEnvWithDiagnostics()` 阻止 PATH 等危险环境变量被覆盖
3. **安全二进制白名单**：`safeBins` + `safeBinProfiles` 精确控制允许执行的程序
4. **后台执行管理**：支持 `yieldMs`/`background` 异步执行，带超时和 abort signal

#### ③ 插件工具注册

插件通过 `openclaw.plugin.json` 声明式注册工具，运行时通过 `resolvePluginTools()` 加载：

```typescript
// src/plugins/tools.ts
export function resolvePluginTools(params) {
  const registry = loadOpenClawPlugins({ config, workspaceDir });
  for (const entry of registry.tools) {
    // entry.factory(context) 返回 AnyAgentTool | AnyAgentTool[]
    let resolved = entry.factory(params.context);
    
    // 可选工具需要经过 allowlist 检查
    if (entry.optional) {
      resolved = resolved.filter(tool => 
        isOptionalToolAllowed({ toolName: tool.name, pluginId: entry.pluginId, allowlist })
      );
    }
    
    // 通过 WeakMap 标记插件元数据（pluginId + optional 状态）
    pluginToolMeta.set(tool, { pluginId: entry.pluginId, optional: entry.optional });
    tools.push(tool);
  }
}
```

### 2.3 工具策略过滤管线（Tool Policy Pipeline）

工具创建后，必须经过**多层安全过滤**才能最终暴露给 LLM。这是 OpenClaw 安全架构的核心：

```
原始工具集（30+ 工具）
  │
  ├── Memory Flush 模式 → 仅保留 read + write
  │
  ├── Message Provider 策略 → voice provider 排除 tts
  │
  ├── Model Provider 策略 → xAI 有原生 web_search 时排除冲突
  │
  ├── Owner-Only 策略 → 非 owner 移除 gateway/cron/nodes/whatsapp_login
  │
  ├── 七层策略管线 (applyToolPolicyPipeline)
  │   ├── ① Profile 策略      → minimal/coding/messaging/full 预设
  │   ├── ② Provider Profile   → 按模型提供商的策略
  │   ├── ③ 全局策略           → config.tools.allow / deny
  │   ├── ④ 全局 Provider 策略 → config.tools.byProvider
  │   ├── ⑤ Agent 策略         → agents.<id>.tools.allow / deny
  │   ├── ⑥ Agent Provider 策略→ agents.<id>.tools.byProvider
  │   └── ⑦ 群组策略           → 按消息来源群组/频道的策略
  │
  ├── Sandbox 策略 → config.tools.sandbox.tools.allow / deny
  │
  └── Subagent 策略 → 子代理的工具限制（深度相关）
  │
  ├── JSON Schema 规范化 → Gemini 清理 / OpenAI 强制 type:object / xAI 去约束关键词
  │
  ├── Before-Tool-Call Hook → 插件钩子 + 循环检测
  │
  └── Abort Signal → 中止信号注入
  │
  ▼
最终工具集（暴露给 LLM）
```

#### 策略匹配算法

```typescript
// src/agents/tool-policy-match.ts
function makeToolPolicyMatcher(policy: SandboxToolPolicy) {
  const deny = compileGlobPatterns(expandToolGroups(policy.deny ?? []));
  const allow = compileGlobPatterns(expandToolGroups(policy.allow ?? []));
  
  return (name: string) => {
    const normalized = normalizeToolName(name);
    // 1. deny 优先：命中 deny 则拒绝
    if (matchesAnyGlobPattern(normalized, deny)) return false;
    // 2. 无 allow 列表 → 允许
    if (allow.length === 0) return true;
    // 3. 命中 allow → 允许
    if (matchesAnyGlobPattern(normalized, allow)) return true;
    // 4. apply_patch 跟随 exec 的策略
    if (normalized === "apply_patch" && matchesAnyGlobPattern("exec", allow)) return true;
    // 5. 默认拒绝
    return false;
  };
}
```

#### 工具预设 Profile

```typescript
// src/agents/tool-catalog.ts
// 四种预设 profile 控制工具可见性
const PROFILES = {
  minimal:   { allow: ["session_status"] },
  coding:    { allow: ["read","write","edit","apply_patch","exec","process",
                       "web_search","web_fetch","memory_search","memory_get",
                       "sessions_*","subagents","cron","image","image_generate"] },
  messaging: { allow: ["sessions_list","sessions_history","sessions_send",
                       "session_status","message"] },
  full:      {},  // 无限制
};
```

#### 工具分组系统

支持通过组名引用多个工具：

```typescript
// 内置组
"group:fs"       → ["read", "write", "edit", "apply_patch"]
"group:runtime"  → ["exec", "process"]
"group:web"      → ["web_search", "web_fetch"]
"group:sessions" → ["sessions_list", "sessions_history", ...]
"group:openclaw" → [所有带 includeInOpenClawGroup 标记的核心工具]
"group:plugins"  → [所有已注册的插件工具]
```

### 2.4 工具执行生命周期

#### 格式转换：AgentTool → ToolDefinition

Pi SDK 内部使用 `ToolDefinition` 格式，OpenClaw 的 `AnyAgentTool` 需要经过适配器转换：

```typescript
// src/agents/pi-tool-definition-adapter.ts
export function toToolDefinitions(tools: AnyAgentTool[]): ToolDefinition[] {
  return tools.map((tool) => ({
    name: tool.name,
    label: tool.label ?? name,
    description: tool.description ?? "",
    parameters: tool.parameters,
    execute: async (...args) => {
      // 1. before-tool-call hook 检查（循环检测 + 插件钩子）
      const hookOutcome = await runBeforeToolCallHook({ toolName, params, toolCallId });
      if (hookOutcome.blocked) throw new Error(hookOutcome.reason);
      
      // 2. 调用 tool.execute()
      const rawResult = await tool.execute(toolCallId, hookOutcome.params, signal, onUpdate);
      
      // 3. 结果规范化
      return normalizeToolExecutionResult({ toolName, result: rawResult });
      
      // 4. 错误处理 → 返回 { status: "error", tool, error } 而非抛出异常
    },
  }));
}
```

#### Before-Tool-Call Hook

每次工具调用前都经过 hook 检查（`src/agents/pi-tools.before-tool-call.ts`）：

```typescript
async function runBeforeToolCallHook(args) {
  // 1. 循环检测（同一工具被反复调用相同参数）
  const loopResult = detectToolCallLoop(sessionState, toolName, params, config);
  if (loopResult.stuck && loopResult.level === "critical") {
    return { blocked: true, reason: loopResult.message };  // 阻止执行
  }
  
  // 2. 记录本次调用
  recordToolCall(sessionState, toolName, params, toolCallId);
  
  // 3. 运行插件注册的 before_tool_call 钩子
  const hookResult = await hookRunner.runBeforeToolCall({ toolName, params });
  if (hookResult?.block) {
    return { blocked: true, reason: hookResult.blockReason };
  }
  
  // 4. 钩子可以修改参数
  if (hookResult?.params) {
    return { blocked: false, params: { ...params, ...hookResult.params } };
  }
  
  return { blocked: false, params };
}
```

#### 工具结果类型

```typescript
// src/agents/tools/common.ts
function textResult(text, details)    → { content: [{ type: "text", text }], details }
function jsonResult(payload)          → 同上，JSON 序列化
function imageResult(params)          → { content: [{ type: "image", data, mimeType }] }
function failedTextResult(text, details) → 带 status: "failed" 的结果

// 错误类
class ToolInputError (400)          → 参数验证错误
class ToolAuthorizationError (403)  → 权限不足
```

### 2.5 配置方式

在 `openclaw.yaml` 中配置工具策略：

```yaml
tools:
  # 全局工具策略
  profile: coding                    # minimal / coding / messaging / full
  allow: [exec, read, write, edit]   # 白名单（支持 glob：exec*）
  deny: [gateway]                    # 黑名单
  alsoAllow: [browser]               # 额外允许（不受 profile 限制）
  
  # 按模型提供商差异化
  byProvider:
    anthropic:
      allow: [exec, read, write, edit, web_search]
    openai:
      profile: full
  
  # Exec 工具详细配置
  exec:
    host: gateway                    # sandbox / gateway / node
    security: allowlist              # deny / allowlist / ask / full
    ask: on-miss                     # off / on-miss / always
    timeoutSec: 1800
    safeBins: [git, npm, pnpm]
    pathPrepend: [/usr/local/bin]
    workspaceOnly: true
  
  # 文件系统策略
  fs:
    workspaceOnly: true              # 限制 read/write/edit 在工作目录内
  
  # 沙箱工具策略
  sandbox:
    tools:
      allow: [exec, process, read, write, edit]
      deny: [browser, canvas, nodes, cron, gateway]
  
  # 子代理工具策略
  subagents:
    tools:
      allow: [read, write, edit, exec]
      deny: [gateway, cron]
  
  # 循环检测
  loopDetection:
    detectors:
      same-tool-same-params: { threshold: 5, level: critical }

# Agent 级别覆盖
agents:
  myAgent:
    tools:
      profile: full
      allow: [gateway]
      exec:
        host: sandbox
```

---

## 三、Sandbox（沙箱）机制

### 3.1 架构概览

Sandbox 是 OpenClaw 的**核心安全隔离层**，将 Agent 的代码执行限制在容器内：

```typescript
// src/agents/sandbox/types.ts
type SandboxConfig = {
  mode: "off" | "non-main" | "all";  // 关闭 / 仅非主Agent / 全部Agent
  backend: "docker" | "ssh" | "openshell";  // 后端类型
  scope: "session" | "agent" | "shared";    // 隔离粒度
  workspaceAccess: "none" | "ro" | "rw";    // 工作区访问级别
  docker: SandboxDockerConfig;               // Docker 配置
  tools: SandboxToolPolicy;                  // 工具策略
  browser: SandboxBrowserConfig;             // 浏览器隔离配置
  prune: SandboxPruneConfig;                 // 自动清理配置
};

type SandboxContext = {
  enabled: boolean;
  containerName: string;
  workspaceDir: string;                      // 沙箱内工作目录
  agentWorkspaceDir: string;                 // Agent 实际工作目录
  workspaceAccess: "none" | "ro" | "rw";
  containerWorkdir: string;                  // 容器内工作目录（/workspace）
  fsBridge?: SandboxFsBridge;                // 文件系统桥接
  backend?: SandboxBackendHandle;            // 后端句柄
};
```

### 3.2 沙箱对工具的影响

当沙箱启用时，工具行为发生根本性变化：

| 工具 | 无沙箱 | 有沙箱 |
|------|--------|--------|
| `read` | `createOpenClawReadTool` 直接读文件 | `createSandboxedReadTool` 通过 fsBridge 读取 |
| `write` | `createHostWorkspaceWriteTool` | `createSandboxedWriteTool`（workspaceAccess=rw）或**移除** |
| `edit` | `createHostWorkspaceEditTool` | `createSandboxedEditTool` 或**移除** |
| `exec` | 本地执行 | `docker exec` 容器内执行 |
| `browser` | 本地浏览器 | 通过 bridgeUrl 连接容器内浏览器 |

### 3.3 沙箱工具默认策略

```typescript
// src/agents/sandbox/constants.ts
const DEFAULT_TOOL_ALLOW = [
  "exec", "process", "read", "write", "edit", "apply_patch",
  "image", "sessions_list", "sessions_history", "sessions_send",
  "sessions_spawn", "sessions_yield", "subagents", "session_status",
];

const DEFAULT_TOOL_DENY = [
  "browser", "canvas", "nodes", "cron", "gateway",
  ...CHANNEL_IDS,  // 所有消息渠道工具
];
```

### 3.4 沙箱安全验证

`src/agents/sandbox/validate-sandbox-security.ts` 在创建容器时进行安全校验：

```typescript
// 阻止危险的主机路径挂载
const BLOCKED_HOST_PATHS = [
  "/etc", "/private/etc", "/proc", "/sys", "/dev", "/root", "/boot",
  "/run", "/var/run", "/var/run/docker.sock",  // Docker socket!
];

// 阻止不安全的 seccomp/apparmor 配置
const BLOCKED_SECCOMP_PROFILES = new Set(["unconfined"]);
const BLOCKED_APPARMOR_PROFILES = new Set(["unconfined"]);

// 保留的容器内目标路径
const RESERVED_CONTAINER_TARGET_PATHS = ["/workspace", "/agent"];
```

### 3.5 子代理工具限制

子代理（subagent）有额外的工具限制，按深度递增：

```typescript
// src/agents/pi-tools.policy.ts
// 始终禁止的工具（所有深度）
const SUBAGENT_TOOL_DENY_ALWAYS = [
  "gateway", "agents_list",     // 系统管理
  "whatsapp_login",             // 交互式设置
  "session_status", "cron",     // 调度
  "memory_search", "memory_get",// 记忆（应通过 spawn prompt 传递）
  "sessions_send",              // 直接发送
];

// 叶子节点额外禁止的工具
const SUBAGENT_TOOL_DENY_LEAF = [
  "subagents", "sessions_list", "sessions_history", "sessions_spawn",
];
```

---

## 四、Skill 机制

### 4.1 Skill 的定义

每个 Skill 是一个目录，包含一个 `SKILL.md` 文件。典型结构：

```
skills/
  weather/
    SKILL.md          # 必需：技能定义文件
    tools/            # 可选：关联工具脚本
```

`SKILL.md` 的 frontmatter 格式：

```markdown
---
name: weather
description: "Get current weather via wttr.in. Use when user asks about weather."
homepage: https://wttr.in/:help
metadata:
  {
    "openclaw": {
      "emoji": "☔",
      "requires": { "bins": ["curl"] },          # 所需二进制
      "install": [                                 # 安装方式
        { "kind": "brew", "formula": "curl", "bins": ["curl"] }
      ]
    }
  }
---

# Weather Skill
（详细使用说明…）
```

### 4.2 关键 Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Skill 名称（唯一标识） |
| `description` | string | 简短描述（LLM 用此判断是否匹配） |
| `metadata.openclaw.always` | boolean | 是否始终包含（无需满足 requires） |
| `metadata.openclaw.requires.bins` | string[] | 所需本地二进制（全部满足） |
| `metadata.openclaw.requires.anyBins` | string[] | 所需二进制（任一满足） |
| `metadata.openclaw.requires.env` | string[] | 所需环境变量 |
| `metadata.openclaw.requires.config` | string[] | 所需配置路径（如 `browser.enabled`） |
| `metadata.openclaw.primaryEnv` | string | 主要环境变量名（用于 apiKey 注入） |
| `metadata.openclaw.os` | string[] | 支持的操作系统（`darwin`/`linux`/`win32`） |
| `metadata.openclaw.install` | InstallSpec[] | 安装规格（brew/node/go/uv/download） |
| `user-invocable` | boolean | 是否允许用户直接 `/skillname` 调用（默认 true） |
| `disable-model-invocation` | boolean | 是否禁止模型自动调用（默认 false） |
| `command-dispatch` | "tool" | 命令分发方式 |
| `command-tool` | string | 分发到的工具名 |

### 4.3 Skill 加载：六层来源合并

`loadSkillEntries()`（`src/agents/skills/workspace.ts`）从六个目录加载，按优先级从低到高：

```
优先级（低 → 高）：
  ① extra          → config.skills.load.extraDirs + 插件 skills 目录
  ② bundled        → 随安装包分发（<packageRoot>/skills/）
  ③ managed        → ~/.openclaw/skills（通过 openclaw skills install）
  ④ personal agents → ~/.agents/skills（个人全局级）
  ⑤ project agents  → <workspace>/.agents/skills（项目级）
  ⑥ workspace      → <workspace>/skills/（工作区级，最高优先级）
```

**同名 Skill 按优先级覆盖**（使用 `Map<name, Skill>` 合并）。

#### 安全措施

1. **路径遍历防护**：`resolveContainedSkillPath()` 确保 skill 目录不会通过符号链接逃逸出配置的根目录
2. **文件大小限制**：单个 `SKILL.md` 最大 256KB
3. **数量限制**：每个来源最多 200 个 skill，每个根目录最多扫描 300 个子目录
4. **插件 skill 隔离**：`isPathInsideWithRealpath()` 确保插件 skill 不会逃逸出插件根目录

### 4.4 Skill 过滤：运行时资格评估

加载后经过 `filterSkillEntries()` → `shouldIncludeSkill()` 多层过滤：

```typescript
// src/agents/skills/config.ts
export function shouldIncludeSkill(params) {
  // 1. 配置中 enabled: false → 排除
  if (skillConfig?.enabled === false) return false;
  
  // 2. 内置 skill 的白名单检查
  if (!isBundledSkillAllowed(entry, allowBundled)) return false;
  
  // 3. 运行时资格评估
  return evaluateRuntimeEligibility({
    os: entry.metadata?.os,            // 操作系统匹配
    requires: entry.metadata?.requires, // 依赖检查
    hasBin: hasBinary,                  // 本地二进制检测
    hasEnv: (envName) => Boolean(       // 环境变量检测
      process.env[envName] ||
      skillConfig?.env?.[envName] ||
      (skillConfig?.apiKey && entry.metadata?.primaryEnv === envName)
    ),
    isConfigPathTruthy: (path) => isConfigPathTruthy(config, path),
  });
}
```

### 4.5 Skill Prompt 格式化：三层降级

```
Tier 1 (Full)     → name + description + location（来自 pi-coding-agent SDK）
  ↓ 超出 30,000 字符预算
Tier 2 (Compact)  → 仅 name + location，无 description
  ↓ 仍然超出
Tier 3 (Truncated) → 二分搜索最大可容纳子集（compact 格式）
```

最终注入到 system prompt 的结构：

```
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.
- When a skill drives external API writes, assume rate limits...

<available_skills>
  <skill>
    <name>weather</name>
    <description>Get current weather...</description>
    <location>~/.openclaw/skills/weather/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

**路径压缩**：`compactSkillPaths()` 将 `/Users/alice/...` 替换为 `~/...`，节省 ~400-600 token。

### 4.6 Skill 环境变量注入

`src/agents/skills/env-overrides.ts` 在 Agent 运行时注入 skill 需要的环境变量：

```typescript
// 在 attempt.ts 中调用
restoreSkillEnv = params.skillsSnapshot
  ? applySkillEnvOverridesFromSnapshot({ snapshot, config })
  : applySkillEnvOverrides({ skills: skillEntries, config });

// 运行结束后恢复
restoreSkillEnv?.();
```

**安全机制**：
- 引用计数管理（多个 skill 注入同一 key 时正确处理）
- 危险环境变量阻止（`OPENSSL_CONF`、PATH 类变量）
- `getActiveSkillEnvKeys()` 暴露当前被注入的 key，防止泄漏到子进程

### 4.7 Skill 的沙箱处理

当沙箱 `workspaceAccess` 为 `none` 或 `ro` 时，skills 需要同步到沙箱工作区：

```typescript
// src/agents/skills/workspace.ts
export async function syncSkillsToWorkspace(params) {
  // 1. 从源工作区加载所有 skill entries
  const entries = loadSkillEntries(sourceDir, { config });
  
  // 2. 清空目标沙箱的 skills/ 目录
  await fsp.rm(targetSkillsDir, { recursive: true, force: true });
  await fsp.mkdir(targetSkillsDir, { recursive: true });
  
  // 3. 复制所有 skill 目录到沙箱
  for (const entry of entries) {
    const dest = resolveSyncedSkillDestinationPath({ targetSkillsDir, entry, usedDirNames });
    await fsp.cp(entry.skill.baseDir, dest, { recursive: true, force: true });
  }
}
```

### 4.8 Skill 文件变更监听

`src/agents/skills/refresh.ts` 使用 chokidar 监听所有 skill 目录：

```typescript
// 监听目标：所有可能的 SKILL.md 文件
// <workspace>/skills/*/SKILL.md
// ~/.agents/skills/*/SKILL.md
// ~/.openclaw/skills/*/SKILL.md
// + 额外目录 + 插件目录

// 变更后触发版本号 bump
watcher.on("change", (path) => {
  bumpSkillsSnapshotVersion({ workspaceDir, reason: "watch", changedPath: path });
});
```

Cron 任务使用版本号判断是否需要重建快照：

```typescript
// src/cron/isolated-agent/skills-snapshot.ts
const shouldRefresh =
  !existingSnapshot ||
  existingSnapshot.version !== snapshotVersion ||
  !matchesSkillFilter(existingSnapshot.skillFilter, skillFilter);
```

### 4.9 配置方式

```yaml
skills:
  # 内置 skill 白名单（仅影响 bundled skills）
  allowBundled: [weather, github, coding-agent]
  
  load:
    extraDirs: [/path/to/custom/skills]  # 额外加载目录
    watch: true                           # 文件监听
    watchDebounceMs: 250                  # 监听防抖
  
  install:
    preferBrew: true
    nodeManager: npm    # npm / pnpm / yarn / bun
  
  limits:
    maxCandidatesPerRoot: 300
    maxSkillsLoadedPerSource: 200
    maxSkillsInPrompt: 150
    maxSkillsPromptChars: 30000
    maxSkillFileBytes: 256000
  
  # 单个 skill 配置
  entries:
    weather:
      enabled: true
    github:
      apiKey: { env: GITHUB_TOKEN }
      env:
        GH_TOKEN: my-token
    tavily:
      enabled: false
```

---

## 五、Tool Call 与 Skill 的交互关系

### 5.1 Skill 如何使用 Tool

Skill 本身不是 tool，但 Skill 的使用依赖 tool：

1. **Agent 通过 `read` 工具读取 `SKILL.md`**：skill 信息在 prompt 中只有名称和路径，Agent 需要先 read 获取完整内容
2. **SKILL.md 中的指令通常引导 Agent 使用 `exec` 工具**：如 weather skill 指导 Agent 执行 `curl wttr.in/London`
3. **部分 Skill 有 `command-dispatch` 机制**：直接分发到指定工具

### 5.2 Skill 命令分发

```markdown
---
command-dispatch: tool
command-tool: browser
command-arg-mode: raw
---
```

当用户使用 `/skillname args` 时，可以直接分发到 `browser` 工具执行，跳过 LLM 推理。

### 5.3 插件同时提供 Tool 和 Skill

例如 `extensions/tavily` 插件：
- 通过 `openclaw.plugin.json` 注册 `tavily_search` / `tavily_extract` 工具
- 通过 `skills/tavily/SKILL.md` 提供使用指南

---

## 六、安全保证总结

### 6.1 工具级安全

| 机制 | 实现位置 | 说明 |
|------|---------|------|
| 七层策略管线 | `tool-policy-pipeline.ts` | 多维度过滤工具可见性 |
| Owner-Only 限制 | `tool-policy.ts` | gateway/cron/nodes 仅 owner 可用 |
| Exec 安全等级 | `bash-tools.exec.ts` | deny → allowlist → ask → full |
| Shell 注入检测 | `bash-tools.exec.ts` | 预检 Python/JS 脚本中的 shell 变量泄漏 |
| 环境变量沙箱化 | `host-env-security.ts` | 阻止 PATH 等危险变量覆盖 |
| 循环检测 | `pi-tools.before-tool-call.ts` | 检测并阻止工具调用死循环 |
| Before-Tool-Call Hook | `pi-tools.before-tool-call.ts` | 插件可阻止或修改任何工具调用 |
| 文件系统边界 | `tool-fs-policy.ts` | `workspaceOnly` 限制文件操作范围 |
| Abort Signal | `pi-tools.abort.ts` | 超时/取消信号传播到所有工具 |

### 6.2 沙箱级安全

| 机制 | 实现位置 | 说明 |
|------|---------|------|
| Docker 容器隔离 | `sandbox/docker.ts` | exec/write/read 在容器内执行 |
| 挂载路径阻止 | `validate-sandbox-security.ts` | 阻止 /etc、/proc、Docker socket 挂载 |
| Seccomp/AppArmor | `validate-sandbox-security.ts` | 阻止 "unconfined" 配置 |
| 工作区访问控制 | `sandbox/types.ts` | none/ro/rw 三级工作区访问 |
| 工具默认策略 | `sandbox/constants.ts` | 沙箱内默认禁止 browser/canvas/cron 等 |
| 浏览器隔离 | `sandbox/browser.ts` | 独立容器运行浏览器，通过 CDP 桥接 |
| 文件系统桥接 | `sandbox/fs-bridge.ts` | 沙箱内工具通过桥接访问文件 |

### 6.3 Skill 级安全

| 机制 | 实现位置 | 说明 |
|------|---------|------|
| 路径遍历防护 | `skills/workspace.ts` | 符号链接解析 + 根目录边界检查 |
| 文件大小限制 | `skills/workspace.ts` | 单个 SKILL.md ≤ 256KB |
| 数量限制 | `skills/workspace.ts` | 每源 ≤ 200 个，prompt 中 ≤ 150 个 |
| 环境变量安全 | `skills/env-overrides.ts` | 阻止危险 env、引用计数管理 |
| 安装规格验证 | `skills/frontmatter.ts` | brew formula/npm spec/Go module 白名单校验 |
| 插件 skill 隔离 | `skills/plugin-skills.ts` | 插件 skill 不能逃逸出插件根目录 |

---

## 七、完整数据流

```
消息到达
    │
    ▼
agentCommand() → runWithModelFallback() → runEmbeddedPiAgent()
    │
    ▼
runEmbeddedAttempt()
    │
    ├── resolveEmbeddedRunSkillEntries()     # 加载 skills
    ├── applySkillEnvOverrides()              # 注入 skill 环境变量
    ├── resolveSkillsPromptForRun()           # 生成 skills prompt
    │
    ├── createOpenClawCodingTools()           # 创建所有工具
    │   ├── 基础编码工具（read/write/edit）
    │   ├── exec + process 工具
    │   ├── OpenClaw 专有工具（20+）
    │   └── 插件工具（memory_search 等）
    │
    ├── [七层策略过滤管线]                     # 过滤工具可见性
    ├── normalizeToolParameters()              # Schema 规范化
    ├── wrapToolWithBeforeToolCallHook()       # 注入前置钩子
    ├── wrapToolWithAbortSignal()              # 注入中止信号
    │
    ├── toToolDefinitions()                    # AgentTool → ToolDefinition
    │
    ├── buildEmbeddedSystemPrompt()            # 组装 system prompt
    │   └── buildSkillsSection()               #   └── 注入 skills 段落
    │
    ├── createAgentSession()                   # 创建 Pi Agent Session
    ├── applySystemPromptOverrideToSession()   # 覆盖 system prompt
    │
    └── session.prompt(userMessage)            # 启动 Agent 循环
        │
        ├── LLM 推理 → 决定调用哪个 tool
        │
        ├── tool_execution_start → beforeToolCallHook → tool.execute()
        │   ├── 循环检测
        │   ├── 参数校验
        │   ├── 执行（可能在沙箱容器内）
        │   └── 结果规范化
        │
        ├── 如果 LLM 选择了某个 skill
        │   └── read tool → 读取 SKILL.md → LLM 按指令操作
        │
        └── 生成最终回复 → 发回消息平台
```
