# OpenClaw Routing 系统深度解读

> 面向软件开发人员，从设计初衷、架构设计、关键实现细节等角度解读，提炼对 Agent 开发的借鉴参考。

## 一、设计初衷：解决什么问题？

OpenClaw 是一个 **多渠道（Multi-Channel）、多 Agent、多账户** 的 AI 网关。它需要同时对接 Discord、Telegram、WhatsApp、Slack、Signal、iMessage、Web 等众多消息渠道。核心问题是：

> **当一条消息从某个渠道的某个对话中抵达时，应该由哪个 Agent 处理？消息历史应该存在哪个 session 里？**

这看起来简单，实际面临多重复杂性：

1. **多 Agent 并存**：同一个 gateway 可以运行多个 Agent（main、research、code-review...），不同对话/频道/服务器需要路由到不同 Agent
2. **多账户**：同一渠道可能有多个 bot 账户（如两个 Telegram bot），每个账户可能绑定不同 Agent
3. **多层级组织结构**：Discord 有 Guild → Channel → Thread 三级，Slack 有 Team → Channel → Thread
4. **跨平台身份统一**：同一用户在 Telegram 是 `123456`，在 Discord 是 `222222222`，需要统一会话
5. **DM 与群聊的会话隔离策略**：有的场景需要所有私聊共享一个 session（连续性），有的需要每个对话方独立 session（隔离性）
6. **线程继承**：Discord/Slack 线程应该继承父频道的 Agent 绑定

---

## 二、架构设计

### 2.1 整体架构

```
┌────────────────────────────────────────────────────────────────┐
│                        Config Layer                            │
│  cfg.bindings: AgentBinding[]    ← 声明式路由规则              │
│  cfg.agents.list: AgentConfig[]  ← Agent 定义                 │
│  cfg.session.dmScope             ← DM 隔离策略                │
│  cfg.session.identityLinks       ← 跨平台身份映射             │
└───────────────────┬────────────────────────────────────────────┘
                    │
    ┌───────────────▼───────────────────────────────┐
    │        src/routing/ — 核心路由层（纯函数）     │
    │                                                │
    │  resolve-route.ts   路由解析引擎              │
    │  session-key.ts     会话密钥构建              │
    │  bindings.ts        绑定查询                  │
    │  account-id.ts      账户 ID 规范化            │
    │  account-lookup.ts  账户条目查找              │
    └──────┬──────────────────┬──────────────────────┘
           │                  │
  ┌────────▼──────┐  ┌───────▼──────────────────┐
  │  Plugin SDK   │  │    Channel Plugins        │
  │  routing.ts   │  │  binding-routing.ts       │
  │  (re-export)  │  │  (配置绑定覆盖层)         │
  └───────────────┘  └──────────────────────────┘
           │                  │
  ┌────────▼──────────────────▼──────────────────┐
  │            Gateway / Runtime                  │
  │  session-key 工具函数的最大消费者             │
  │  (不直接调 resolveAgentRoute)                 │
  └──────────────────────────────────────────────┘
```

**关键设计决策**：

- 路由层是 **纯函数 + 缓存**，不做 I/O，不依赖异步
- Gateway 不直接调用 `resolveAgentRoute()`——调用发生在 channel plugin 的运行时（`runtime-channel.ts`），Gateway 主要消费 session-key 工具函数
- Plugin SDK 通过 `src/plugin-sdk/routing.ts` 统一 re-export，严格控制 API 边界

### 2.2 模块职责

| 文件 | 职责 |
|------|------|
| `resolve-route.ts` | **核心路由解析引擎**：9 级优先级匹配，索引化查找，多级缓存 |
| `session-key.ts` | **会话密钥构建与规范化**：编码式命名空间，dmScope 隔离，identityLinks 身份映射 |
| `bindings.ts` | **绑定管理**：从配置提取绑定列表，查询 channel 下绑定的账户 |
| `account-id.ts` | **账户 ID 规范化**：防原型链污染，LRU 缓存 |
| `account-lookup.ts` | **账户条目查找**：大小写不敏感匹配 |
| `default-account-warnings.ts` | **配置警告**：缺少默认账户时的提示信息生成 |

### 2.3 核心数据模型

**输入 — `ResolveAgentRouteInput`**：

```typescript
type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;        // 完整配置对象
  channel: string;            // "discord" | "telegram" | "whatsapp" | ...
  accountId?: string | null;  // bot 账户 ID
  peer?: RoutePeer | null;    // { kind: "direct"|"group"|"channel", id: string }
  parentPeer?: RoutePeer;     // 线程的父级 peer（用于绑定继承）
  guildId?: string;           // Discord Guild ID
  teamId?: string;            // Slack Team ID
  memberRoleIds?: string[];   // Discord 成员角色 ID 列表
};
```

**输出 — `ResolvedAgentRoute`**：

```typescript
type ResolvedAgentRoute = {
  agentId: string;            // 路由到的 Agent
  sessionKey: string;         // 会话持久化密钥
  mainSessionKey: string;     // 主会话密钥（DM 折叠用）
  lastRoutePolicy: "main" | "session";  // last-route 更新目标
  matchedBy: "binding.peer" | "binding.peer.parent" | ... | "default";
  channel: string;
  accountId: string;
};
```

**声明式绑定规则 — `AgentBinding`**：

```typescript
type AgentBindingMatch = {
  channel: string;              // 必选
  accountId?: string;           // "*" = 任意账户
  peer?: { kind, id: string };  // id "*" = 通配符
  guildId?: string;             // Discord Guild
  teamId?: string;              // Slack Team
  roles?: string[];             // Discord 角色
};

type AgentBinding = AgentRouteBinding | AgentAcpBinding;
```

---

## 三、关键实现细节

### 3.1 九级优先级路由匹配

这是 routing 最核心的设计——**分层优先级匹配**：

| 优先级 | matchedBy | 含义 | 典型场景 |
|--------|-----------|------|---------|
| 1 | `binding.peer` | 精确 peer 匹配 | WhatsApp 号码 → Agent A |
| 2 | `binding.peer.parent` | 父级 peer 继承 | Discord 线程继承父频道绑定 |
| 3 | `binding.peer.wildcard` | 通配符 peer（`id="*"`） | "所有 DM 走 Agent B" |
| 4 | `binding.guild+roles` | Guild + 角色匹配 | 有 admin 角色的用户走 Opus |
| 5 | `binding.guild` | 仅 Guild 匹配 | 整个 Discord 服务器走一个 Agent |
| 6 | `binding.team` | Team 匹配 | Slack workspace 级别 |
| 7 | `binding.account` | 账户匹配 | 特定 bot 账户走特定 Agent |
| 8 | `binding.channel` | 频道级（`accountId="*"`） | 所有 WhatsApp 账户的兜底 |
| 9 | `default` | 全局默认 | 无任何匹配时 |

实现上使用 **索引结构 `EvaluatedBindingsIndex`** 加速查找：

```typescript
type EvaluatedBindingsIndex = {
  byPeer: Map<string, EvaluatedBinding[]>;       // peer 精确匹配索引
  byPeerWildcard: EvaluatedBinding[];             // 通配符 peer
  byGuildWithRoles: Map<string, EvaluatedBinding[]>;
  byGuild: Map<string, EvaluatedBinding[]>;
  byTeam: Map<string, EvaluatedBinding[]>;
  byAccount: EvaluatedBinding[];
  byChannel: EvaluatedBinding[];
};
```

匹配过程定义为 **tiers 数组**，逐层扫描：

```typescript
const tiers = [
  { matchedBy: "binding.peer",          candidates: collectPeerIndexedBindings(index, peer), ... },
  { matchedBy: "binding.peer.parent",   candidates: collectPeerIndexedBindings(index, parentPeer), ... },
  { matchedBy: "binding.peer.wildcard", candidates: index.byPeerWildcard, ... },
  { matchedBy: "binding.guild+roles",   candidates: index.byGuildWithRoles.get(guildId), ... },
  // ... 继续
];
for (const tier of tiers) {
  if (!tier.enabled) continue;
  const matched = tier.candidates.find(
    c => tier.predicate(c) && matchesBindingScope(c.match, scope)
  );
  if (matched) return choose(matched.binding.agentId, tier.matchedBy);
}
return choose(defaultAgentId, "default");
```

### 3.2 Session Key 设计

Session Key 是会话持久化和并发控制的关键标识，格式：`agent:{agentId}:{rest}`

```
agent:main:main                                    ← DM, dmScope="main"
agent:main:direct:alice                            ← DM, dmScope="per-peer"
agent:main:telegram:direct:123456                  ← DM, dmScope="per-channel-peer"
agent:main:telegram:tasks:direct:7550356539        ← DM, dmScope="per-account-channel-peer"
agent:main:discord:channel:c1                      ← 群聊/频道
agent:main:discord:channel:c1:thread:t-123         ← 线程
plugin-binding:codex:a1b2c3d4e5f6...               ← 插件绑定（独立命名空间）
```

**DM 隔离策略（dmScope）** 是一个精妙的设计维度：

| dmScope | Session Key 示例 | 语义 |
|---------|------------------|------|
| `main` | `agent:main:main` | 所有 DM 共享一个 session——连续性最好 |
| `per-peer` | `agent:main:direct:alice` | 每个对话方独立 session——跨渠道统一 |
| `per-channel-peer` | `agent:main:telegram:direct:alice` | 按渠道+对话方隔离 |
| `per-account-channel-peer` | `agent:main:telegram:tasks:direct:alice` | 最强隔离 |

**身份链接（identityLinks）** 可以跨平台统一身份：

```yaml
session:
  identityLinks:
    alice: ["telegram:111111111", "discord:222222222222222222"]
```

这样 Alice 不管从 Telegram 还是 Discord 私聊，都会落到同一个 session（在 per-peer/per-channel-peer 模式下）。

### 3.3 多级缓存策略

路由解析在高流量场景下会被高频调用，因此设计了三级缓存：

```typescript
// 第一级：绑定评估缓存（按 config 对象 identity）
const evaluatedBindingsCacheByCfg = new WeakMap<OpenClawConfig, EvaluatedBindingsCache>();

// 第二级：channel+account 索引缓存（最多 2000 key）
byChannelAccount: Map<string, EvaluatedBinding[]>          // 合并后的绑定列表
byChannelAccountIndex: Map<string, EvaluatedBindingsIndex>  // 索引结构

// 第三级：最终路由结果缓存（最多 4000 key）
const resolvedRouteCacheByCfg = new WeakMap<
  OpenClawConfig,
  { byKey: Map<string, ResolvedAgentRoute> }
>();
```

关键细节：

- 使用 **`WeakMap<OpenClawConfig, ...>`**——配置对象一旦被 GC，缓存自动释放，无需手动清理
- 缓存 key 包含 `bindingsRef`、`agentsRef`、`sessionRef` 引用比较——配置热重载后自动失效
- 超过容量上限（4000）时**全量清空重建**（而非 LRU 淘汰），简化实现
- 测试验证了 2205+ 绑定场景下 `listBindings` 只被调用 1 次

### 3.4 跨平台兼容层

路由系统在多处做了 **兼容性规范化**：

**peer kind 兼容**：`dm` ↔ `direct`、`group` ↔ `channel` 互相匹配：

```typescript
function peerKindMatches(bindingKind: ChatType, scopeKind: ChatType): boolean {
  if (bindingKind === scopeKind) return true;
  const both = new Set([bindingKind, scopeKind]);
  return both.has("group") && both.has("channel");
}
```

**账户 ID 安全规范化**：防止原型链污染攻击（`__proto__`、`constructor`、`prototype`），限制 64 字符，有 LRU 缓存（512 上限）：

```typescript
function normalizeCanonicalAccountId(value: string): string | undefined {
  const canonical = canonicalizeAccountId(value);
  if (!canonical || isBlockedObjectKey(canonical)) return undefined;
  return canonical;
}
```

**Agent ID 规范化**：仅允许 `[a-z0-9_-]`，最长 64 字符，不合法字符折叠为 `-`。

### 3.5 两层路由：核心路由 + 配置绑定覆盖

核心路由 `resolveAgentRoute()` 完成后，Channel 层还有一个覆盖层——`binding-routing.ts` 中的 `resolveConfiguredBindingRoute()`：

```typescript
// 第一层：核心路由（纯函数，基于 cfg.bindings）
const route = resolveAgentRoute({ cfg, channel, accountId, peer, ... });

// 第二层：配置绑定覆盖（有状态，基于 conversation binding registry）
const { route: finalRoute } = resolveConfiguredBindingRoute({ cfg, route, conversation });
```

这种两层设计的目的：核心路由是**声明式静态匹配**，配置绑定覆盖是**运行时动态绑定**（如用户通过命令把某个对话绑定到特定 Agent/session）。

### 3.6 插件级路由——审批驱动的绑定

`src/plugins/conversation-binding.ts` 实现了一个独立于核心路由的 **插件级绑定系统**：

- 插件请求绑定 → 生成审批请求（交互式按钮：Allow once / Always allow / Deny）
- 审批结果持久化到 `~/.openclaw/plugin-binding-approvals.json`
- 生成独立的 session key：`plugin-binding:{pluginId}:{sha256-hash}`
- 支持 legacy 绑定迁移（旧格式 → 新格式）

这是一个完整的 **权限控制 + 状态机** 设计，插件不能随意劫持对话。

---

## 四、对 Agent 开发的借鉴

### 4.1 声明式路由 + 分层优先级

**核心思想**：把路由规则从代码中抽取到配置（`cfg.bindings`），用**明确的优先级层级**代替模糊的权重/打分。

**应用场景**：当你的 Agent 系统需要根据复杂条件（用户身份、所属组织、对话类型、权限角色）决定由哪个 Agent/模型/prompt 处理时，可以借鉴这种 tier-based matching：

```
精确匹配 > 通配符匹配 > 组织级匹配 > 角色匹配 > 账户级兜底 > 全局默认
```

**优势**：比 if-else 链或权重打分更可预测、更易调试、无歧义。

### 4.2 Session Key 作为编码式命名空间

**核心思想**：把路由决策的关键维度（agentId、channel、accountId、peerKind、peerId）编码进一个字符串 key。

**好处**：

- 天然适配 KV 存储（Redis、LevelDB、文件系统）
- 解析和构建都是纯字符串操作，极快
- 便于调试——看到 `agent:main:discord:channel:c1:thread:t-123` 就知道完整上下文
- 可以用前缀查询实现范围操作

### 4.3 配置驱动 + WeakMap 缓存

**核心思想**：用 `WeakMap<Config, Cache>` 实现"配置变更即缓存失效"。

**应用场景**：当你的 Agent 系统有热重载配置的需求时，不需要显式 invalidation：

```typescript
const cacheByCfg = new WeakMap<Config, ComputedResult>();
function getResult(cfg: Config) {
  if (cacheByCfg.has(cfg)) return cacheByCfg.get(cfg)!;
  const result = expensive_compute(cfg);
  cacheByCfg.set(cfg, result);
  return result;
}
// 配置对象换了引用 → 缓存自动失效
```

### 4.4 DM Scope 的四级隔离策略

**核心思想**：把"会话隔离度"作为一个可配置的维度，从最宽松到最严格提供 4 个级别。

**应用场景**：多租户 Agent 系统中，不同客户可能需要不同的隔离级别。有的客户希望跨渠道连续对话（`main`），有的希望严格隔离（`per-account-channel-peer`）。把这个决策抽取为配置选项而非硬编码。

### 4.5 跨平台身份映射（Identity Links）

**核心思想**：用一个简单的 `{canonical: [platform:id, ...]}` 映射表，实现多平台用户身份统一。

**应用场景**：当你的 Agent 需要"记住"同一用户跨渠道的偏好/上下文时，Identity Links 的设计既简洁又灵活——不需要数据库，一个 JSON map 即可。

### 4.6 两层路由分离（静态 + 动态）

**核心思想**：核心路由是纯函数、无副作用、可缓存的静态匹配；运行时绑定是有状态的动态覆盖。两者分层组合。

**应用场景**：Agent 开发中，"默认该由哪个 Agent 处理"是配置问题，"用户临时把这个对话切换到另一个 Agent"是运行时问题。分层处理可以保证核心路由的高性能和可预测性，同时支持灵活的动态覆盖。

### 4.7 插件路由的审批机制

**核心思想**：第三方插件不能随意接管对话，必须经过用户授权（allow-once / allow-always / deny），审批结果持久化。

**应用场景**：当你的 Agent 平台允许第三方扩展时，路由权限控制是安全红线。OpenClaw 的设计给出了一个完整的参考：请求 → 审批 → 持久化 → 绑定/拒绝的完整生命周期。

---

## 五、总结

OpenClaw 的 routing 系统展示了一个 **生产级、多渠道 Agent 路由引擎** 的完整设计：

| 维度 | 设计选择 | 价值 |
|------|---------|------|
| 匹配策略 | 9 级优先级 tier-based | 可预测、可调试、无歧义 |
| Session 标识 | 编码式 key + dmScope | 灵活隔离 + 天然 KV 友好 |
| 缓存 | WeakMap + 引用相等性 | 配置热重载自动失效 |
| 兼容 | peer kind 互匹配 + 原型链防护 | 跨平台鲁棒 |
| 扩展 | Plugin SDK re-export + 审批驱动绑定 | 安全的第三方扩展 |
| 分层 | 静态路由 + 动态绑定覆盖 | 高性能 + 灵活性兼得 |

---

## 附录：关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/routing/resolve-route.ts` | 核心路由解析引擎（833 行） |
| `src/routing/session-key.ts` | 会话密钥构建与规范化（255 行） |
| `src/routing/bindings.ts` | 绑定查询（116 行） |
| `src/routing/account-id.ts` | 账户 ID 规范化（72 行） |
| `src/routing/account-lookup.ts` | 账户条目查找（35 行） |
| `src/routing/default-account-warnings.ts` | 配置警告格式化（18 行） |
| `src/channels/plugins/binding-routing.ts` | Channel 层配置绑定覆盖（92 行） |
| `src/channels/plugins/configured-binding-match.ts` | 配置绑定匹配逻辑（121 行） |
| `src/channels/thread-bindings-policy.ts` | 线程绑定策略（270 行） |
| `src/plugins/conversation-binding.ts` | 插件级审批驱动绑定（994 行） |
| `src/plugin-sdk/routing.ts` | Plugin SDK 路由 API 统一出口（35 行） |
| `src/plugins/runtime/runtime-channel.ts` | 运行时路由能力暴露 |
| `src/config/types.agents.ts` | 绑定配置类型定义 |
| `src/config/bindings.ts` | 配置层绑定过滤 |
| `src/config/types.base.ts` | DmScope / SessionConfig 类型定义 |
