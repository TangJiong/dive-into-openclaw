# OpenClaw 安全保障机制深度解读

## 一句话定位

OpenClaw 作为一个让 AI Agent 直接操作文件系统、执行 Shell 命令、访问网络的系统，其**安全保障是整个架构的生命线**。项目在 `src/security/`、`src/infra/`、`src/gateway/`、`src/agents/` 等模块中构建了一套多层纵深防御体系，覆盖了从工具调用拦截、沙箱隔离、密钥保护到网络防御的完整安全链路。

---

## 二、安全架构全景图

```
用户消息入站
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  第一层：Gateway 认证与访问控制                       │
│  - Token/Password/Tailscale/Trusted-Proxy 四种模式  │
│  - 速率限制（滑动窗口 + 锁定）                       │
│  - 设备认证（Device Auth）                           │
│  - WebSocket 未授权洪泛守卫                          │
│  - CORS Origin 严格校验                              │
└───────────┬─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│  第二层：Channel 消息来源鉴权                         │
│  - Allow-From 白名单策略                             │
│  - DM Policy / Group Policy                         │
│  - Owner-Only 命令限制                               │
│  - Elevated Mode 授权控制                            │
└───────────┬─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│  第三层：工具调用拦截与审批                           │
│  - 危险工具名单（DANGEROUS_ACP_TOOLS）               │
│  - 执行审批系统（Human-in-the-Loop）                 │
│  - 安全二进制白名单 + argv 约束                      │
│  - 七层工具策略过滤管道                              │
│  - Gateway HTTP 工具调用拒绝列表                     │
└───────────┬─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│  第四层：执行环境隔离                                 │
│  - Docker Sandbox 容器隔离                           │
│  - 宿主路径黑名单（/etc、/proc、Docker socket 等）   │
│  - 网络模式限制（禁 host 模式）                      │
│  - seccomp / AppArmor 强制启用                       │
│  - 环境变量净化（危险 key 过滤）                     │
└───────────┬─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│  第五层：密钥与凭证保护                               │
│  - SecretRef 间接引用（不在配置中存明文）             │
│  - 密钥文件 0o600 权限 + 符号链接拒绝                │
│  - 时序安全比较（防 Timing Attack）                  │
│  - 日志/输出自动脱敏（20+ 正则模式）                 │
│  - Payload 诊断凭证字段移除                          │
└───────────┬─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│  第六层：网络安全防护                                 │
│  - SSRF 防护（DNS 重绑定 + 私网 IP 阻断）           │
│  - 浏览器导航守卫（SSRF + 协议限制）                 │
│  - Fetch Guard（DNS pinning + 重定向限制）           │
│  - Webhook 请求校验                                  │
└───────────┬─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│  第七层：安全审计与自检                               │
│  - `openclaw security audit` 命令（30+ 检查项）     │
│  - 文件权限审计（world-writable 检测）               │
│  - 危险配置标志检测                                  │
│  - 深度探测（Gateway 连通性验证）                    │
│  - 插件代码安全扫描                                  │
└─────────────────────────────────────────────────────┘
```

---

## 三、危险工具调用拦截（核心安全机制）

### 3.1 危险工具定义

文件 `src/security/dangerous-tools.ts` 集中定义了两类危险工具：

```typescript
// Gateway HTTP 调用默认拒绝的工具 —— 这些工具通过非交互的 HTTP 调用非常危险
export const DEFAULT_GATEWAY_HTTP_TOOL_DENY = [
  "sessions_spawn",    // 远程 spawn Agent = RCE
  "sessions_send",     // 跨 Session 消息注入
  "cron",              // 定时任务控制面
  "gateway",           // 网关重配置
  "whatsapp_login",    // 交互式登录（HTTP 无法完成）
] as const;

// ACP 协议中必须人工审批的工具 —— 自动化场景绝不允许"静默执行"
export const DANGEROUS_ACP_TOOL_NAMES = [
  "exec", "spawn", "shell",           // 命令执行
  "sessions_spawn", "sessions_send",   // Session 操控
  "gateway",                           // 网关操控
  "fs_write", "fs_delete", "fs_move",  // 文件系统写/删
  "apply_patch",                       // 代码补丁
] as const;
```

**设计理念**：将危险工具定义集中在一处，避免 Gateway HTTP 限制、安全审计、ACP 提示三方的定义产生漂移（drift）。

### 3.2 执行审批系统（Human-in-the-Loop）

这是 OpenClaw 最重要的安全机制之一，位于 `src/infra/exec-approvals.ts` 及相关文件。整个审批系统围绕三个核心维度运作：

```
ExecSecurity（执行安全级别）     ExecAsk（用户询问策略）
┌──────────────┐               ┌──────────────┐
│ "deny"       │ ← 默认值      │ "off"        │ 不询问（直接按 security 决定）
│ "allowlist"  │               │ "on-miss"    │ ← 默认值：不在白名单才询问
│ "full"       │               │ "always"     │ 每次都询问
└──────────────┘               └──────────────┘
```

**审批判断核心逻辑**（`requiresExecApproval()`）：

```typescript
export function requiresExecApproval(params: {
  ask: ExecAsk;
  security: ExecSecurity;
  analysisOk: boolean;
  allowlistSatisfied: boolean;
}): boolean {
  return (
    params.ask === "always" ||           // 总是询问
    (params.ask === "on-miss" &&         // 不在白名单时询问
      params.security === "allowlist" &&
      (!params.analysisOk || !params.allowlistSatisfied))
  );
}
```

**审批文件存储**（安全设计亮点）：

```typescript
// 审批配置文件使用 0o600 权限（仅 owner 可读写）
export function saveExecApprovals(file: ExecApprovalsFile) {
  const filePath = resolveExecApprovalsPath();
  fs.writeFileSync(filePath, JSON.stringify(file, null, 2) + "\n", { mode: 0o600 });
  fs.chmodSync(filePath, 0o600);  // 双重保障
}
```

审批请求通过 **Unix Socket** 从 Agent 进程转发到 CLI 进程，用户在终端看到审批提示后做出决策（allow-once / allow-always / deny）。

### 3.3 Shell 命令静态分析引擎

位于 `src/infra/exec-approvals-analysis.ts`（22KB），这是一个完整的 Shell 命令解析器，能够：

1. **拆解管道链**：`cmd1 | cmd2 && cmd3` → 逐段分析
2. **解析重定向**：识别 `>`, `>>`, `<` 等操作
3. **识别 Shell 包装器**：`bash -c "..."`, `zsh -lc "..."` → 提取内部命令
4. **解析环境变量前缀**：`VAR=value command` → 分离出实际命令
5. **符号链接逃逸检测**：通过现有祖先路径解析 realpath，防止符号链接绕过

### 3.4 安全二进制白名单与 argv 约束

`src/infra/exec-safe-bin-policy.ts` 定义了一组"低风险"二进制程序白名单：

```typescript
export const DEFAULT_SAFE_BINS = [
  "cat", "head", "tail", "wc", "sort", "uniq",   // 文本流工具
  "grep", "awk", "sed", "cut", "tr",              // 文本处理
  "ls", "find", "which", "whoami", "date",        // 信息查询
  // ... 等
];
```

但光有白名单还不够——每个二进制还有 **argv 约束配置文件**（`exec-safe-bin-policy-profiles.ts`），限制允许的参数模式。例如，`grep` 不允许 `--exec` 参数（某些版本的 `grep` 可以执行外部命令）。

此外，安全二进制的**可信路径**也受限制：只信任 `/bin`、`/usr/bin` 等系统目录，不信任 `/usr/local/bin`（用户可写）或 `/tmp`（临时目录）。

### 3.5 七层工具策略过滤管道

在 Agents 层解读中提到的工具过滤，从安全角度更详细地展开：

```
原始工具集
  │
  ├── ① tools.profile Policy        → Profile 级别（如 "minimal" 只允许 read/search）
  ├── ② tools.byProvider.profile     → Provider 级别限制
  ├── ③ tools.allow / tools.deny     → 全局白/黑名单
  ├── ④ agents.*.tools.allow/deny    → 单个 Agent 的工具策略
  ├── ⑤ group tools.allow            → 群聊场景的额外限制
  ├── ⑥ Owner-Only Filter            → 仅 Owner 可用的高危工具
  └── ⑦ Sandbox Tool Policy          → 沙箱内可用工具（DEFAULT_TOOL_ALLOW / DENY）
  │
  ▼
最终暴露给 LLM 的工具集
```

**关键安全设计**：当 allowlist 仅包含插件工具（不含核心工具）时，系统会自动**剥离该 allowlist**（`stripPluginOnlyAllowlist()`），防止意外禁用所有核心工具。

---

## 四、用户 API Key 与密钥保护

### 4.1 SecretRef 间接引用机制

OpenClaw 不鼓励在配置文件中直接写入明文密钥，而是提供了 **SecretRef** 引用机制：

```yaml
# 不推荐：明文存储
gateway:
  auth:
    token: "my-secret-token-12345"

# 推荐：SecretRef 引用
gateway:
  auth:
    token:
      source: "file"           # 从文件读取
      id: "~/.openclaw/secrets/gateway-token"
    # 或
    token: "${OPENCLAW_GATEWAY_TOKEN}"  # 从环境变量读取
```

SecretRef 的解析逻辑在 `src/config/types.secrets.ts` 和 `src/secrets/` 模块中，支持多种后端：文件、环境变量、外部密钥管理器等。

### 4.2 密钥文件安全读取

`src/infra/secret-file.ts` 实现了密钥文件的安全读取，包含多重校验：

```typescript
export function loadSecretFileSync(filePath, label, options) {
  // ① 路径解析与规范化
  const resolvedPath = resolveUserPath(trimmedPath);

  // ② 符号链接检查（可选拒绝）
  if (options.rejectSymlink && previewStat.isSymbolicLink()) {
    return { ok: false, message: "must not be a symlink" };
  }

  // ③ 必须是常规文件
  if (!previewStat.isFile()) { ... }

  // ④ 文件大小限制（防止读取巨型文件导致 DoS）
  if (previewStat.size > maxBytes) { ... }  // 默认 16KB

  // ⑤ TOCTOU 安全：通过 openVerifiedFileSync 打开并验证
  //    (open → fstat → 验证一致性，防止 race condition)
  const opened = openVerifiedFileSync({ ... });

  // ⑥ 读取并返回
  const raw = fs.readFileSync(opened.fd, "utf8");
  return { ok: true, secret: raw.trim() };
}
```

其中 `openVerifiedFileSync`（`src/infra/safe-open-sync.ts`）采用 **open-then-verify** 模式：先用 `fs.openSync()` 获取文件描述符，再用 `fs.fstatSync(fd)` 验证文件属性，避免 **TOCTOU（Time-of-check to Time-of-use）** 竞态条件。

### 4.3 时序安全的密钥比较

`src/security/secret-equal.ts`：

```typescript
export function safeEqualSecret(provided, expected): boolean {
  if (typeof provided !== "string" || typeof expected !== "string") {
    return false;
  }
  // 先 SHA-256 哈希，再使用 timingSafeEqual
  // 这样即使字符串长度不同也不会泄露信息
  const hash = (s: string) => createHash("sha256").update(s).digest();
  return timingSafeEqual(hash(provided), hash(expected));
}
```

**为什么先哈希再比较？** `crypto.timingSafeEqual()` 要求两个 Buffer 长度相同，否则直接抛异常。通过先 SHA-256 哈希，确保两个比较对象始终是 32 字节，既满足了 `timingSafeEqual` 的要求，又防止了 **Timing Attack**（通过比较耗时推断密钥内容）。

### 4.4 Auth Profile 凭证存储

`src/agents/auth-profiles/store.ts` 管理多个 LLM API key 的存储：

- 存储位置：`~/.openclaw/auth-profiles.json`
- 文件权限：`0o600`（仅 owner 可读写）
- 每个 profile 有独立的 cooldown 机制，失败后短时间内不会被重用
- 支持 OAuth token 自动刷新（`oauth.ts`）

---

## 五、用户数据防泄露

### 5.1 日志敏感信息自动脱敏

`src/logging/redact.ts` 实现了一套全自动的敏感信息脱敏系统：

```typescript
const DEFAULT_REDACT_PATTERNS: string[] = [
  // ENV-style 赋值：API_KEY=sk-xxx
  `\\b[A-Z0-9_]*(?:KEY|TOKEN|SECRET|PASSWORD|PASSWD)\\b\\s*[=:]\\s*...`,

  // JSON 字段："apiKey": "sk-xxx"
  `"(?:apiKey|token|secret|password|...)"\s*:\s*"([^"]+)"`,

  // CLI 参数：--api-key sk-xxx
  `--(?:api[-_]?key|token|secret|password)\\s+...`,

  // Authorization 头：Bearer eyJxx...
  `Authorization\\s*[:=]\\s*Bearer\\s+...`,

  // PEM 私钥块
  `-----BEGIN [A-Z ]*PRIVATE KEY-----...-----END...`,

  // 常见 Token 前缀
  `\\b(sk-[A-Za-z0-9_-]{8,})\\b`,           // OpenAI
  `\\b(ghp_[A-Za-z0-9]{20,})\\b`,           // GitHub PAT
  `\\b(xox[baprs]-[A-Za-z0-9-]{10,})\\b`,   // Slack
  `\\b(AIza[0-9A-Za-z\\-_]{20,})\\b`,        // Google API
  `\\b(pplx-[A-Za-z0-9_-]{10,})\\b`,         // Perplexity
  // Telegram Bot Token
  `\\bbot(\\d{6,}:[A-Za-z0-9_-]{20,})\\b`,
];
```

脱敏规则：
- 短于 18 字符的 token → 替换为 `***`
- 长 token → 保留前 6 后 4 字符：`sk-abc1…9xyz`
- PEM 私钥 → 仅保留首尾行：`-----BEGIN...` + `…redacted…` + `-----END...`

### 5.2 配置快照脱敏

`src/config/redact-snapshot.ts`（22KB）实现了配置导出时的深度脱敏：

```
配置快照导出流程：
  ① 遍历所有配置路径
  ② 对标记为 sensitive 的路径（由 schema.hints.ts 定义），替换值为 "***"
  ③ SecretRef 对象 → 保留 source/provider 字段，脱敏 id 字段
  ④ URL 中的 userinfo 部分（user:password@host）→ 自动剥离
  ⑤ 环境变量占位符（${VAR}）→ 保留原样（不是真实值）
  ⑥ 对原始 YAML/JSON 文本进行二次脱敏（防遗漏）
```

### 5.3 Payload 诊断信息净化

`src/agents/payload-redaction.ts` 在持久化诊断日志时移除敏感字段：

```typescript
// 被识别为凭证字段的名称模式（会被自动移除）
function isCredentialFieldName(key: string): boolean {
  const normalized = normalizeFieldName(key);
  return (
    normalized.endsWith("apikey") ||
    normalized.endsWith("password") ||
    normalized.endsWith("secret") ||
    normalized.endsWith("secretkey") ||
    normalized.endsWith("token") ||
    normalized === "authorization" ||
    normalized === "proxyauthorization"
  );
}

// 图像 base64 数据 → 替换为 <redacted>，保留 SHA-256 摘要和字节数
function shouldRedactImageData(record) {
  return type === "image" || hasImageMime(record);
}
```

### 5.4 宿主环境变量净化

`src/infra/host-env-security.ts` 在 Agent 执行 Shell 命令前，**过滤掉所有危险的环境变量**：

```json
// src/infra/host-env-security-policy.json
{
  "blockedKeys": [
    "NODE_OPTIONS",        // 可注入 Node.js 启动参数
    "NODE_PATH",           // 可劫持模块解析
    "PYTHONPATH",          // Python 模块注入
    "BASH_ENV",            // Bash 启动脚本注入
    "SSLKEYLOGFILE",       // TLS 密钥泄露
    "LD_PRELOAD",          // 动态库劫持（通过 blockedPrefixes: "LD_"）
    "DYLD_INSERT_LIBRARIES" // macOS 动态库劫持（通过 blockedPrefixes: "DYLD_"）
    // ... 31 个被阻断的 key
  ],
  "blockedOverrideKeys": [
    "HOME", "GIT_SSH_COMMAND", "EDITOR", "PROMPT_COMMAND",
    "HISTFILE", "OPENSSL_CONF", "XDG_CONFIG_HOME"
    // ... 34 个禁止覆盖的 key
  ],
  "blockedOverridePrefixes": ["GIT_CONFIG_", "NPM_CONFIG_"],
  "blockedPrefixes": ["DYLD_", "LD_", "BASH_FUNC_"]
}
```

**特别注意**：`PATH` 变量被**硬编码禁止覆盖**（不在 JSON 中，而是在代码中单独处理），因为它是命令解析和安全二进制检查的安全边界。

---

## 六、沙箱执行环境安全

### 6.1 Docker 容器安全验证

`src/agents/sandbox/validate-sandbox-security.ts` 在创建沙箱容器前执行全面的安全验证：

**宿主路径黑名单**：

```typescript
export const BLOCKED_HOST_PATHS = [
  "/etc",                          // 系统配置
  "/private/etc",                  // macOS 系统配置
  "/proc", "/sys", "/dev",         // 内核接口
  "/root",                         // root 用户目录
  "/boot",                         // 引导文件
  "/run", "/var/run",              // 运行时目录
  "/var/run/docker.sock",          // Docker socket（容器逃逸）
  "/private/var/run/docker.sock",  // macOS Docker socket
];
```

**挂载验证逻辑**：

```
对每个 bind mount：
  ① 非绝对路径 → 拒绝（相对路径难以安全验证）
  ② 挂载 "/" → 拒绝（不允许挂载系统根目录）
  ③ 命中 BLOCKED_HOST_PATHS → 拒绝
  ④ 不在 allowedSourceRoots 内 → 拒绝
  ⑤ 目标路径命中保留路径（/workspace 等） → 拒绝
  ⑥ **符号链接逃逸检测**：resolve realpath 后重新验证
```

**网络模式限制**：

```typescript
// 禁止 host 网络模式（绕过容器网络隔离）
if (blockedReason === "host") {
  throw new Error("Network 'host' mode bypasses container network isolation.");
}
// 禁止 container:* 模式（共享其他容器网络命名空间）
if (blockedReason === "container_namespace_join") {
  throw new Error("Network 'container:*' bypasses sandbox network isolation.");
}
```

**安全配置文件强制**：

```typescript
// 禁止 seccomp=unconfined（移除系统调用过滤）
validateSeccompProfile(cfg.seccompProfile);
// 禁止 apparmor=unconfined（移除强制访问控制）
validateApparmorProfile(cfg.apparmorProfile);
```

### 6.2 沙箱工具策略

`src/agents/sandbox/tool-policy.ts` 为沙箱环境定义了**独立的工具策略**：

```
DEFAULT_TOOL_ALLOW（沙箱默认允许的工具）
  → 通常是受限子集，例如只允许文件读写、搜索

DEFAULT_TOOL_DENY（沙箱默认拒绝的工具）
  → 例如禁止在沙箱内 spawn 新 session、操控 gateway
```

策略支持三级继承：`agent 级 > global 级 > 默认值`。

---

## 七、网络安全防护

### 7.1 SSRF 防护体系

`src/infra/net/ssrf.ts`（13.7KB）实现了全面的 SSRF（Server-Side Request Forgery）防护：

```typescript
// 被阻断的主机名
const BLOCKED_HOSTNAMES = new Set([
  "localhost",
  "localhost.localdomain",
  "metadata.google.internal",     // GCP 元数据服务（云环境 SSRF 经典目标）
]);

// 私网 IP 检查（默认阻断）
function isPrivateIpAddress(address: string): boolean {
  // 检查 IPv4: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8
  // 检查 IPv6: ::1, fe80::/10, fc00::/7
  // 检查 IPv4-mapped IPv6: ::ffff:10.0.0.1
}
```

**DNS Pinning 防护**：

```
标准 fetch 的问题：
  ① DNS 解析 example.com → 1.2.3.4（公网 IP，通过检查）
  ② 实际请求时重新解析 → 10.0.0.1（私网 IP，绕过检查）
  这就是 DNS 重绑定攻击（DNS Rebinding）

OpenClaw 的 Fetch Guard 解决方案：
  ① DNS 解析并 pin 住 IP 地址
  ② 创建 Pinned Dispatcher（锁定到已验证的 IP）
  ③ 所有后续请求直接使用 pinned IP
  ④ 手动处理重定向（每跳都重新验证 SSRF）
  ⑤ 跨域重定向自动清理敏感 Header
```

核心实现在 `src/infra/net/fetch-guard.ts`：

```typescript
export async function fetchWithSsrFGuard(params: GuardedFetchOptions) {
  while (true) {
    // ① 解析并验证 URL（只允许 http/https）
    const parsedUrl = new URL(currentUrl);

    // ② DNS 解析 + SSRF 策略检查
    const pinned = await resolvePinnedHostnameWithPolicy(parsedUrl.hostname, {
      policy: params.policy,
    });

    // ③ 创建 pinned dispatcher（锁定到已验证 IP）
    const dispatcher = createPinnedDispatcher(pinned, ...);

    // ④ 发送请求（redirect: "manual" 手动处理重定向）
    const response = await fetcher(parsedUrl.toString(), {
      redirect: "manual",
      dispatcher,
    });

    // ⑤ 处理重定向：验证新 URL、限制次数、检测循环
    if (isRedirectStatus(response.status)) {
      if (redirectCount > maxRedirects) throw new Error("Too many redirects");
      if (visited.has(nextUrl)) throw new Error("Redirect loop detected");
      // 跨域重定向：剥离敏感 Header
      if (nextParsedUrl.origin !== parsedUrl.origin) {
        currentInit = retainSafeHeadersForCrossOriginRedirect(currentInit);
      }
      continue;
    }
    return { response, finalUrl: currentUrl };
  }
}
```

### 7.2 浏览器导航守卫

`src/browser/navigation-guard.ts` 为 Agent 的浏览器操作提供 SSRF 防护：

```typescript
export async function assertBrowserNavigationAllowed(opts) {
  // ① 只允许 http/https 协议（阻止 file://, data:, javascript: 等）
  if (!NETWORK_NAVIGATION_PROTOCOLS.has(parsed.protocol)) {
    // 仅允许 about:blank
    if (!isAllowedNonNetworkNavigationUrl(parsed)) throw ...;
  }

  // ② SSRF 策略检查（与 fetch guard 共享基础设施）
  await resolvePinnedHostnameWithPolicy(parsed.hostname, {
    policy: opts.ssrfPolicy,
  });
}
```

### 7.3 Plugin SDK 的 SSRF 策略

`src/plugin-sdk/ssrf-policy.ts` 为插件提供了 SSRF 策略工具：

- **主机名后缀白名单**：`normalizeHostnameSuffixAllowlist()` — 例如只允许 `*.example.com`
- **私网访问控制**：`assertHttpUrlTargetsPrivateNetwork()` — HTTP URL 必须指向私网可信主机
- **构建 SSRF 策略**：`buildHostnameAllowlistPolicyFromSuffixAllowlist()` — 从配置生成策略

---

## 八、Gateway 认证与访问控制

### 8.1 四种认证模式

`src/gateway/auth.ts` 支持四种认证模式：

| 模式 | 适用场景 | 安全等级 |
|------|---------|---------|
| `token` | 推荐默认，共享密钥认证 | ★★★★★ |
| `password` | 简单密码认证 | ★★★★ |
| `trusted-proxy` | 反向代理委托认证 | ★★★★★（依赖代理配置） |
| `none` | 仅限 loopback | ★★ |

Token 认证使用前述的 `safeEqualSecret()` 进行时序安全比较。

### 8.2 认证速率限制

`src/gateway/auth-rate-limit.ts` 实现了**滑动窗口 + 锁定**的速率限制：

```
默认配置：
  maxAttempts:  10 次     ← 窗口内最大失败次数
  windowMs:     60,000ms  ← 滑动窗口 1 分钟
  lockoutMs:    300,000ms ← 超限后锁定 5 分钟

特殊规则：
  - Loopback 地址（127.0.0.1 / ::1）默认豁免，防止本地 CLI 被锁
  - 多 scope 支持：shared-secret、device-token、hook-auth 独立计数
  - 成功认证后自动重置计数器
```

### 8.3 WebSocket 未授权洪泛守卫

`src/gateway/server/ws-connection/unauthorized-flood-guard.ts` 防止攻击者通过大量未授权 WebSocket 消息消耗服务器资源：

```typescript
class UnauthorizedFloodGuard {
  // 超过 10 次未授权请求 → 强制关闭连接
  // 每 100 次记录一条日志（防日志洪泛）
  registerUnauthorized(): { shouldClose: boolean; shouldLog: boolean } { ... }
}
```

### 8.4 设备认证（Device Auth）

`src/gateway/device-auth.ts` + `src/shared/device-auth-store.ts` 实现了针对 Control UI 的设备级认证：

- 设备首次连接需要通过**配对流程**（Pairing）
- 配对后颁发 Device Token
- Device Token 支持**定期轮换**
- 可通过 `dangerouslyDisableDeviceAuth` 临时禁用（安全审计会标记为 **critical**）

---

## 九、安全审计体系

### 9.1 `openclaw security audit` 命令

`src/security/audit.ts`（1502 行）实现了一个全面的安全审计引擎，覆盖 **30+ 检查项**：

| 检查类别 | 关键检查项 | 严重等级 |
|---------|-----------|---------|
| **Gateway 认证** | 非 loopback 绑定无认证 | critical |
| | Token 长度不足 24 字符 | warn |
| | 无速率限制 | warn |
| **Control UI** | 设备认证被禁用 | critical |
| | Origin 白名单含通配符 | critical |
| | Host-header 回退已启用 | critical |
| **文件权限** | 状态目录 world-writable | critical |
| | 配置文件 world-readable | critical |
| | 配置文件是符号链接 | warn |
| **执行安全** | exec security=full 已配置 | warn/critical |
| | 开放渠道可触达 exec Agent | critical |
| | safeBins 包含解释器（无 profile） | warn |
| **日志安全** | 脱敏功能已关闭 | warn |
| **网络暴露** | Tailscale Funnel 已启用 | critical |
| | mDNS full 模式泄露元数据 | warn |
| **沙箱** | exec host=sandbox 但 sandbox=off | warn |
| | Docker 容器标签异常 | warn |
| **插件信任** | 第三方插件代码安全扫描 | warn |
| **危险配置** | `dangerously*` / `allowInsecure*` 标志 | warn |
| **凭证卫生** | 配置中存在明文密钥 | warn |

### 9.2 危险配置标志检测

`src/security/dangerous-config-flags.ts` 集中检测所有 `dangerously*` 和 `allowInsecure*` 前缀的配置项：

```typescript
export function collectEnabledInsecureOrDangerousFlags(cfg): string[] {
  const flags = [];
  if (cfg.gateway?.controlUi?.allowInsecureAuth === true)
    flags.push("gateway.controlUi.allowInsecureAuth=true");
  if (cfg.gateway?.controlUi?.dangerouslyAllowHostHeaderOriginFallback === true)
    flags.push("...");
  if (cfg.gateway?.controlUi?.dangerouslyDisableDeviceAuth === true)
    flags.push("...");
  if (cfg.hooks?.gmail?.allowUnsafeExternalContent === true)
    flags.push("...");
  if (cfg.tools?.exec?.applyPatch?.workspaceOnly === false)
    flags.push("tools.exec.applyPatch.workspaceOnly=false");
  return flags;
}
```

### 9.3 文件系统权限审计

`src/security/audit-fs.ts` 检查关键路径的文件权限：

```
检查 ~/.openclaw/（状态目录）:
  - world-writable → critical（其他用户可篡改状态）
  - group-writable → warn（组内用户可篡改）
  - world/group-readable → warn（可能泄露密钥）
  - 推荐权限：0o700（drwx------）

检查 openclaw.yaml（配置文件）:
  - world/group-writable → critical（可被篡改）
  - world-readable → critical（配置含 token/密钥）
  - 推荐权限：0o600（-rw-------）
  - 符号链接 → warn（需信任链接目标）
```

---

## 十、ACP 协议的安全约束

在 ACP（Agent Cooperation Protocol）场景中，安全约束更为严格：

```typescript
// src/acp/client.ts
const SAFE_AUTO_APPROVE_TOOL_IDS = new Set([
  "read", "search", "web_search", "memory_search"
]);
// 只有只读工具可以自动批准

// 危险工具必须人工审批
import { DANGEROUS_ACP_TOOLS } from "../security/dangerous-tools.js";
// exec, spawn, shell, fs_write, fs_delete, fs_move, apply_patch...
```

ACP 客户端还会：
1. **过滤 Provider 认证环境变量**：`omitEnvKeysCaseInsensitive()` 确保 spawn 的子进程不继承父进程的 API key
2. **工具名称验证**：`normalizeToolName()` 使用严格正则 `/^[a-z0-9._-]+$/` 防止注入
3. **沙箱会话禁止 spawn ACP**：防止沙箱内的 Agent 通过 ACP 逃逸到宿主

---

## 十一、安全设计亮点总结

### 11.1 纵深防御（Defense in Depth）

从 Gateway 认证 → Channel 鉴权 → 工具策略 → 执行审批 → 沙箱隔离 → 环境变量净化，**每一层都独立生效**，即使某层被突破，下一层仍能提供保护。

### 11.2 默认安全（Secure by Default）

- 执行安全默认 `"deny"`
- 询问策略默认 `"on-miss"`
- 审批文件权限默认 `0o600`
- SSRF 默认阻断私网访问
- 日志脱敏默认开启
- 沙箱 seccomp/apparmor 默认不允许 "unconfined"

### 11.3 显式危险标记（Explicit Danger Marking）

所有安全降级操作都以 `dangerously*` 或 `allowInsecure*` 命名，确保：
- 代码审查时一目了然
- 安全审计自动检测并标记
- 开发者不会"无意间"降低安全等级

### 11.4 集中定义 + 分散执行

危险工具列表、环境变量黑名单、SSRF 策略等安全定义**集中在少数文件中**，但在 Gateway、Agent、Sandbox、ACP 等多个执行点分散检查，避免因定义漂移导致的安全缺口。

### 11.5 安全审计即代码

`openclaw security audit` 不是事后附加的工具，而是与核心代码紧密集成的**自检系统**，能自动检测 30+ 种已知风险模式，将安全知识固化为可执行的检查逻辑。

---

## 十二、安全模块文件地图

| 模块目录 | 核心文件 | 职责 |
|---------|---------|------|
| `src/security/` | `audit.ts` (1502行) | 安全审计引擎 |
| | `dangerous-tools.ts` | 危险工具名单定义 |
| | `dangerous-config-flags.ts` | 危险配置标志检测 |
| | `secret-equal.ts` | 时序安全密钥比较 |
| | `audit-fs.ts` | 文件系统权限审计 |
| | `dm-policy-shared.ts` | DM 策略安全检查 |
| `src/infra/` | `exec-approvals.ts` (590行) | 执行审批核心 |
| | `exec-approvals-allowlist.ts` (670行) | 白名单评估引擎 |
| | `exec-approvals-analysis.ts` (22KB) | Shell 命令静态分析 |
| | `exec-safe-bin-policy.ts` | 安全二进制策略 |
| | `host-env-security.ts` | 环境变量净化 |
| | `secret-file.ts` | 密钥文件安全读取 |
| | `safe-open-sync.ts` | TOCTOU 安全文件打开 |
| | `net/ssrf.ts` (13.7KB) | SSRF 防护核心 |
| | `net/fetch-guard.ts` | Fetch 守卫（DNS pinning） |
| | `path-guards.ts` | 路径安全守卫 |
| | `hardlink-guards.ts` | 硬链接安全守卫 |
| `src/gateway/` | `auth.ts` (15KB) | Gateway 认证 |
| | `auth-rate-limit.ts` | 认证速率限制 |
| | `device-auth.ts` | 设备认证 |
| | `security-path.ts` | 安全路径检查 |
| `src/agents/` | `tool-policy.ts` | 工具策略 |
| | `tool-policy-pipeline.ts` | 七层过滤管道 |
| | `path-policy.ts` | 路径边界检查 |
| | `sandbox/validate-sandbox-security.ts` (344行) | 沙箱安全验证 |
| | `payload-redaction.ts` | 诊断数据脱敏 |
| `src/logging/` | `redact.ts` | 日志自动脱敏 |
| `src/config/` | `redact-snapshot.ts` (22KB) | 配置快照脱敏 |
| `src/plugin-sdk/` | `ssrf-policy.ts` | 插件 SSRF 策略 |
| | `webhook-request-guards.ts` | Webhook 请求校验 |
| `src/browser/` | `navigation-guard.ts` | 浏览器导航守卫 |
