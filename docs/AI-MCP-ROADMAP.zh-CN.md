# CF Manager — AI / Remote MCP 多账号统一运维改造方案

> 状态：Design / Implementation Plan  
> 执行入口：实际开发请同时读取仓库根目录 [`CODEX_IMPLEMENTATION.md`](../CODEX_IMPLEMENTATION.md)，该文件定义 WP0→WP12 的实现顺序、测试门槛、安全约束和最终验收标准。  
> 目标仓库：`xiaochou164/cf-manager`  
> 基线：保留现有 CF Manager Web 管理后台与 Cloudflare Pages/Worker 部署能力，在同一套账号库和业务层上增加 Remote MCP，使人工操作与 AI 运维共享账号、权限、审计和 Cloudflare API 路由。

---

## 1. 改造目标

本项目不改造成“只有 MCP 的工具”，而是形成一个双入口的 Cloudflare 运维中心：

- **Web Admin**：本人日常查看、手工维护和高风险操作确认。
- **Remote MCP**：供 ChatGPT / Codex / Claude / Hermes 等 AI Agent 统一调用。
- **Shared Core**：Web 和 MCP 必须共用同一套 Account Registry、Credential Vault、Cloudflare API Client、Permission Engine、Risk Engine 和 Audit Log。
- **Multi Account**：支持多个彼此独立的 Cloudflare 登录账号，不要求它们属于同一个 Cloudflare User。
- **Zero-server-first**：优先维持 Cloudflare Pages/Workers + D1 + KV 架构，个人使用场景以 Cloudflare Free Plan 可运行作为设计目标。
- **Backward compatible**：现有管理后台、Docker 版和现有 API 尽量不破坏。

最终使用体验：

```text
                         ┌──────────────────────────┐
                         │        CF Manager        │
                         └────────────┬─────────────┘
                                      │
                   ┌──────────────────┴──────────────────┐
                   │                                     │
              /admin/*                                /mcp
            Web Admin UI                         Remote MCP Server
                   │                                     │
                   └──────────────────┬──────────────────┘
                                      │
                         Unified Service Layer
                                      │
                ┌─────────────────────┼─────────────────────┐
                │                     │                     │
        Account Registry      Permission/Risk       Audit / Approval
                │                     │                     │
                └─────────────────────┼─────────────────────┘
                                      │
                          Cloudflare API Router
                                      │
              ┌───────────────────────┼────────────────────────┐
              ▼                       ▼                        ▼
          CF Account A            CF Account B             CF Account C
        DNS/Workers/R2          Pages/D1/KV              Tunnel/Rules
```

---

## 2. 现有代码可直接复用的能力

当前 Worker 版已经具备较好的改造基础：

- `accounts` D1 表：多账号、API Token / Global API Key、Account ID、启用能力、账号状态。
- `audit_log` D1 表：已有基础审计。
- API Token / API Key 已加密保存。
- `cfFetch` / Cloudflare REST API 服务层可继续作为统一请求基础。
- `/api/accounts`、`/api/dns`、`/api/workers`、`/api/storage`、`/api/tunnels` 等已有业务路由。
- `/admin/*` 已作为独立 Web UI。
- `/v1/*` 为现有 OpenAI-compatible AI 接口，应继续保留，**不要与 MCP 混用**。
- D1 + KV + Hono 适合直接新增 `/mcp`。

因此本次原则是：**抽公共服务层，不重写现有 Web 功能。**

---

## 3. 明确的非目标

第一阶段不要做：

1. 不把所有现有 REST Route 直接机械映射成几百个 MCP Tool。
2. 不允许 AI 任意访问任意 URL。
3. 不允许 AI 直接读取、返回或导出 Cloudflare API Token / Global API Key 明文。
4. 不默认开放删除 Zone、删除 D1、清空 R2、删除生产 Worker 等破坏性操作。
5. 不为了多账号切换自动分摊 Cloudflare AI 等产品配额。
6. 不把 MCP 与现有 `/v1` OpenAI API 合并。
7. 不要求第一阶段接入外部数据库、VPS、Redis 或付费服务。

---

## 4. 总体架构

新增以下逻辑层：

```text
worker/src/
├── mcp/
│   ├── server.ts
│   ├── protocol.ts
│   ├── tools/
│   │   ├── accounts.ts
│   │   ├── search.ts
│   │   ├── resources.ts
│   │   ├── execute.ts
│   │   ├── audit.ts
│   │   └── approvals.ts
│   └── schemas/
│
├── services/
│   ├── accountRegistry.ts
│   ├── cloudflareGateway.ts
│   ├── permissionEngine.ts
│   ├── riskEngine.ts
│   ├── approvalService.ts
│   ├── auditService.ts
│   └── resourceSearch.ts
│
└── routes/
    └── mcp.ts
```

后续 Docker 版尽量复用同样的 domain/service contract；具体存储适配由 D1 / SQLite Repository 分别实现。

---

## 5. 账号模型改造

### 5.1 AI Alias

给每个账号增加唯一、易读的 AI 别名：

示例：

```text
name: 个人主账号
ai_alias: main

name: 开发测试账号
ai_alias: dev

name: 备用账号
ai_alias: backup
```

之后自然语言可以稳定映射：

- “查看 main 账号的 Workers”
- “把 dev 的 DNS 配置同步到 backup”
- “搜索所有账号里的 obsidian-universal-sync”

约束：

- 小写。
- 建议仅允许 `a-z0-9-_ ` 中去除空格后的安全字符集。
- D1 唯一索引。
- 修改 alias 必须写审计日志。
- AI 默认优先使用 alias，不在日志/UI 中依赖邮箱作为唯一标识。

### 5.2 Accounts 表建议扩展

建议 migration，不直接破坏旧 schema：

```sql
ALTER TABLE accounts ADD COLUMN ai_alias TEXT;
ALTER TABLE accounts ADD COLUMN ai_enabled INTEGER DEFAULT 0;
ALTER TABLE accounts ADD COLUMN ai_permissions TEXT DEFAULT 'read';
ALTER TABLE accounts ADD COLUMN ai_risk_policy TEXT DEFAULT 'confirm_high';
ALTER TABLE accounts ADD COLUMN credential_version INTEGER DEFAULT 1;
ALTER TABLE accounts ADD COLUMN last_verified_at DATETIME;

CREATE UNIQUE INDEX IF NOT EXISTS idx_accounts_ai_alias
ON accounts(ai_alias)
WHERE ai_alias IS NOT NULL;
```

`ai_permissions` 第一版可以 JSON 字符串保存，后续若复杂化再正规化。

建议结构：

```json
{
  "read": true,
  "write": false,
  "deploy": false,
  "dns_write": false,
  "storage_write": false,
  "delete": false,
  "dangerous": false
}
```

默认新账号：

- Web 人工权限：维持现有逻辑。
- AI：默认关闭。
- 用户必须在后台显式勾选“允许 AI 管理”。

---

## 6. MCP 接口设计

### 6.1 Transport

优先实现标准 **Remote MCP + Streamable HTTP**，入口：

```text
POST /mcp
GET  /mcp
```

兼容 MCP Inspector，并设计为可直接被支持 Remote MCP 的 Agent 客户端添加。

不要复用 `API_SECRET` 作为唯一长期 MCP 凭证。新增独立 MCP 认证配置。

建议 Env：

```text
MCP_ENABLED=true
MCP_AUTH_SECRET=<high-entropy-secret>
MCP_REQUIRE_APPROVAL=true
```

未来可以扩展 OAuth，但第一阶段使用高强度 Bearer Secret 即可。

### 6.2 MCP Tool 数量保持少而稳定

第一阶段只提供以下核心工具：

#### 1. `accounts_list`

用途：

- 列出 AI 被授权访问的账号。
- 返回 `id / name / ai_alias / cloudflare_account_id / available_features / ai_permissions`。
- 永远不返回 credential。

支持：

```json
{
  "include_inactive": false
}
```

#### 2. `resources_search`

跨账号资源搜索。

输入：

```json
{
  "query": "obsidian-universal-sync",
  "account": "all",
  "types": ["workers", "pages", "dns", "d1", "kv", "r2"]
}
```

输出统一 Resource Descriptor：

```json
{
  "account_alias": "main",
  "type": "worker",
  "id": "...",
  "name": "obsidian-universal-sync",
  "zone": null,
  "summary": {},
  "console_hint": {}
}
```

第一阶段至少支持：

- zones
- dns
- workers
- pages
- kv
- d1
- r2
- tunnels

#### 3. `cloudflare_api_search`

用于搜索支持的 Cloudflare API 能力，而不是把几千个 Endpoint 全展开为 MCP tools。

输入：

```json
{
  "query": "list worker deployments"
}
```

输出：

- 推荐 method
- canonical endpoint template
- 所需 scope
- 风险等级
- 是否允许 MCP execute

API catalog 可以从项目内静态 allowlist/catalog 开始，后续再自动生成。

#### 4. `cloudflare_api_execute`

通用受控 Cloudflare API 执行入口。

输入建议：

```json
{
  "account": "dev",
  "method": "GET",
  "path": "/accounts/{account_id}/workers/scripts",
  "params": {},
  "body": null,
  "reason": "List workers in dev account"
}
```

关键规则：

- `account` 必须解析为系统已有账号。
- `path` 只能是 Cloudflare API **相对路径**。
- 服务端固定 origin：`https://api.cloudflare.com/client/v4`。
- 禁止客户端传完整 URL。
- 禁止 `http://`、`https://`、userinfo、重定向到任意 Host。
- Account ID / Zone ID 尽量由服务端解析，不信任模型任意替换。
- Endpoint 必须命中 allowlist / policy catalog。
- 权限不足直接拒绝。
- 高风险操作先走 approval。
- credential 仅服务端解密并注入请求头。
- 返回结果时对敏感字段做 redaction。

#### 5. `audit_search`

让 AI 能回答：

- “昨天 AI 改过哪些 DNS？”
- “最近谁部署过这个 Worker？”
- “main 账号最近 20 次写操作是什么？”

#### 6. `approval_status`

用于查询待确认高风险操作。

第一阶段不建议让 MCP 自己批准自己。批准动作由 Web Admin 完成。

---

## 7. 高频操作的 Typed Tools

在通用 `execute` 之外，可以提供少量 typed tools，目的不是扩大工具数量，而是把最常用、最容易误操作的场景做强校验。

建议第二阶段加入：

- `dns_record_list`
- `dns_record_upsert`
- `worker_get`
- `worker_deploy`
- `worker_logs_query`（若现有能力允许）
- `pages_project_list`
- `storage_list`

不要第一版就全面铺开。

---

## 8. Permission Engine

权限判断必须在服务端执行，不能依赖 Agent 提示词。

建议动作域：

```text
read
write
deploy
dns_write
storage_write
delete
dangerous
```

示例策略：

| 操作 | 权限 | 默认风险 |
|---|---|---|
| List Workers | read | LOW |
| Read DNS records | read | LOW |
| Create DNS TXT | dns_write | MEDIUM |
| Update Worker | deploy | HIGH |
| Create D1 | storage_write | HIGH |
| Delete Worker | delete | CRITICAL |
| Delete Zone | dangerous | CRITICAL |
| Delete D1 | dangerous | CRITICAL |
| Delete R2 Bucket | dangerous | CRITICAL |
| Purge all cache | write | HIGH |

策略顺序：

```text
MCP authentication
    ↓
Account exists + ai_enabled
    ↓
Permission check
    ↓
Endpoint policy
    ↓
Risk classification
    ↓
Approval if required
    ↓
Cloudflare call
    ↓
Audit
```

---

## 9. Risk / Approval Engine

风险等级：

```text
LOW
MEDIUM
HIGH
CRITICAL
```

建议默认：

- LOW：直接执行。
- MEDIUM：允许执行，但详细审计。
- HIGH：默认需要人工确认，可在账号策略里改为允许。
- CRITICAL：始终需要 Web Admin 确认；第一阶段不允许永久关闭确认。

新增表：

```sql
CREATE TABLE IF NOT EXISTS ai_approvals (
  id TEXT PRIMARY KEY,
  account_id INTEGER REFERENCES accounts(id) ON DELETE CASCADE,
  request_id TEXT NOT NULL,
  tool_name TEXT NOT NULL,
  action TEXT NOT NULL,
  target TEXT,
  request_json TEXT NOT NULL,
  risk_level TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN ('pending','approved','rejected','expired','executed')),
  expires_at DATETIME NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  decided_at DATETIME,
  executed_at DATETIME
);

CREATE INDEX IF NOT EXISTS idx_ai_approvals_status_created
ON ai_approvals(status, created_at);
```

批准流程：

```text
AI request
   ↓
HIGH / CRITICAL
   ↓
create approval
   ↓
MCP 返回 APPROVAL_REQUIRED + approval_id
   ↓
Web Admin -> AI Operations -> Pending Approvals
   ↓
Approve / Reject
   ↓
approval becomes one-shot execution ticket
   ↓
execute exactly once
```

必须防止：

- 同一个 approval 重放。
- approval 修改参数后复用。
- 过期 approval 执行。
- 批准 A 账号操作后换成 B 账号执行。

可对 request canonical JSON 做 SHA-256，批准票据绑定 hash。

---

## 10. Audit Log 升级

现有 `audit_log` 继续保留，但扩展以下字段：

```sql
ALTER TABLE audit_log ADD COLUMN actor_type TEXT DEFAULT 'human';
ALTER TABLE audit_log ADD COLUMN actor_id TEXT;
ALTER TABLE audit_log ADD COLUMN request_id TEXT;
ALTER TABLE audit_log ADD COLUMN risk_level TEXT;
ALTER TABLE audit_log ADD COLUMN tool_name TEXT;
ALTER TABLE audit_log ADD COLUMN duration_ms INTEGER;
ALTER TABLE audit_log ADD COLUMN metadata TEXT;
```

`actor_type`：

- human
- mcp
- system

MCP 每次调用至少记录：

- request_id
- MCP client（若可识别）
- account
- tool
- operation
- target
- risk
- approval_id
- start/end
- success/error
- Cloudflare error code
- 参数摘要
- **绝不记录明文 token / Global API Key / secret value**

Web Admin 新增：

```text
AI Operations
├── Overview
├── Pending Approvals
├── AI Permissions
├── MCP Clients
└── Audit
```

---

## 11. Credential 安全

### 强制规则

1. MCP tool 永不返回 `api_token`、`api_key`。
2. 所有凭据只在执行 Cloudflare 请求的最内层短暂解密。
3. 禁止把完整 Cloudflare headers 写入日志。
4. 错误堆栈必须 redact authorization header。
5. Global API Key 继续兼容，但 UI 中显示“AI 管理不推荐”。
6. AI 启用时优先要求最小权限 API Token。
7. 新增 Token 健康检查与 `last_verified_at`。
8. 账号删除/停用后立刻使 MCP 路由失效。

### 未来增强

可考虑 credential key rotation：

- `credential_version`
- 新旧 ENCRYPTION_KEY 迁移
- 不要求第一阶段完成。

---

## 12. Cloudflare API Gateway 安全边界

这是整个改造最重要的部分。

### 12.1 固定 Host

所有通用 execute 请求：

```text
https://api.cloudflare.com/client/v4
```

Host 在服务端硬编码。

模型只能传：

```text
/accounts/...
/zones/...
/user/...
```

### 12.2 Endpoint Policy Catalog

建议建立：

```text
worker/src/data/cloudflare-api-policy.json
```

每条包含：

```json
{
  "method": "POST",
  "pathPattern": "/zones/:zone_id/dns_records",
  "permission": "dns_write",
  "risk": "MEDIUM",
  "allowed": true
}
```

未命中策略：

```text
DENY BY DEFAULT
```

第一阶段先覆盖 CF Manager 当前已有能力对应 endpoint；后续按需扩展。

### 12.3 SSRF

禁止：

- 完整 URL。
- 协议相对 URL。
- path 中 `@host` 欺骗。
- 非 Cloudflare redirect。
- 任意自定义 upstream。

对于现有 Browser Rendering / 外部 URL 功能，继续走它自己的 SSRF 策略，不与 MCP Cloudflare API execute 混用。

---

## 13. MCP Authentication

第一阶段：

```http
Authorization: Bearer <MCP_AUTH_SECRET>
```

要求：

- `MCP_AUTH_SECRET` 和 `API_SECRET` 独立。
- 未配置 secret 时，生产环境下 MCP 默认关闭。
- `MCP_ENABLED` 必须显式开启。
- Bearer 比较使用 constant-time 思路。
- 登录失败限流。
- 不把 secret 写入 D1。

后续 V2 可增加：

- 多 MCP Client。
- 每 Client 独立 token。
- Client-specific scopes。
- OAuth 2.1 / Dynamic Client Registration（依据目标 MCP 客户端兼容性决定）。

建议预留表：

```sql
CREATE TABLE IF NOT EXISTS mcp_clients (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  token_hash TEXT,
  enabled INTEGER DEFAULT 1,
  permissions TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_used_at DATETIME
);
```

明文 client token 仅创建时显示一次，只存 hash。

---

## 14. Web Admin 改造

### Account 页面

新增：

- AI 管理：On / Off
- AI Alias
- AI 权限
- 高风险策略
- 最近验证时间
- “测试 AI 权限”按钮

### System Settings

新增 MCP 设置：

- MCP 启用状态
- Endpoint
- Authentication 状态
- Client 管理
- 一键复制 MCP URL
- Rotate Client Secret
- MCP health check

### AI Operations

Dashboard 卡片：

- 24h MCP 调用数
- 成功率
- 高风险操作数
- 待审批数
- 最近调用
- 各账号 AI 操作分布

### Audit

增加过滤：

- Actor：Human / MCP / System
- Account
- Tool
- Risk
- Status
- Date
- Request ID

---

## 15. Cross-account Operations

跨账号操作不要由一个“超级 API”无约束执行。

正确方式：

```text
Plan
 ↓
Resolve source + target accounts
 ↓
Read source resources
 ↓
Normalize
 ↓
Preflight target
 ↓
Generate operation list
 ↓
Risk check per operation
 ↓
Approval if required
 ↓
Idempotent execution
 ↓
Verify
 ↓
Audit
```

例如：

> 把 main 上的 worker X 部署到 backup

系统应先生成计划：

1. 读取 main/X。
2. 获取 bindings / routes / compatibility settings。
3. 标记 secret bindings 为“不可读取，需要人工提供/目标已有”。
4. 检查目标 D1/R2/KV 是否存在。
5. 计算需创建资源。
6. 输出预检结果。
7. 获取批准。
8. 再执行。

**绝不能假装可以从 Cloudflare API 读取 Worker Secret 的明文。**

---

## 16. Idempotency

所有 MCP 写操作建议支持：

```text
request_id
idempotency_key
```

D1 表：

```sql
CREATE TABLE IF NOT EXISTS ai_operations (
  id TEXT PRIMARY KEY,
  idempotency_key TEXT UNIQUE,
  account_id INTEGER,
  tool_name TEXT NOT NULL,
  request_hash TEXT NOT NULL,
  state TEXT NOT NULL,
  result_json TEXT,
  error_json TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

状态建议：

```text
PREPARED
APPROVAL_REQUIRED
APPROVED
EXECUTING
SUCCEEDED
FAILED
REJECTED
EXPIRED
```

同一个 idempotency key：

- 相同 request hash：返回已有结果。
- 不同 request hash：返回 conflict。

---

## 17. MCP 返回错误规范

统一 machine-readable error：

```json
{
  "code": "APPROVAL_REQUIRED",
  "message": "This operation requires approval",
  "request_id": "...",
  "approval_id": "...",
  "risk": "HIGH"
}
```

建议错误码：

- UNAUTHORIZED
- MCP_DISABLED
- ACCOUNT_NOT_FOUND
- ACCOUNT_AI_DISABLED
- PERMISSION_DENIED
- ENDPOINT_NOT_ALLOWED
- VALIDATION_ERROR
- APPROVAL_REQUIRED
- APPROVAL_EXPIRED
- IDEMPOTENCY_CONFLICT
- CLOUDFLARE_API_ERROR
- RATE_LIMITED
- INTERNAL_ERROR

不要只返回自然语言字符串。

---

## 18. Rate Limit

至少做三层：

1. MCP Client 总体调用频率。
2. 单账号写操作频率。
3. HIGH / CRITICAL 操作频率。

KV 可做快速计数，D1 做兜底审计。

对于 Cloudflare 返回 429：

- 读取 Retry-After（如有）。
- 不无限自动重试。
- 写操作重试必须结合 idempotency。
- MCP 返回明确的 retryable 状态。

---

## 19. Free-tier 设计原则

目标是个人使用时无需额外服务器：

```text
Cloudflare Pages / Workers
        +
       D1
        +
       KV
```

原则：

- 不引入长期运行进程。
- 不引入外部队列作为第一阶段硬依赖。
- 避免一次 MCP 请求做大规模 CPU 密集型转换。
- 大批量跨账号操作拆成受控小批次。
- 对资源搜索结果做短时 KV cache。
- 审计分页，禁止无限查询。
- 默认限制单次跨账号 fan-out 数。

如后续批量任务复杂，再评估 Queues / Durable Objects，但不要第一阶段提前引入。

---

## 20. Docker 兼容策略

Cloudflare Worker 版优先完成。

随后把核心逻辑抽成无平台依赖的 Service Contract：

```ts
interface AccountRepository {}
interface AuditRepository {}
interface ApprovalRepository {}
interface OperationRepository {}
interface CredentialService {}
interface CloudflareGateway {}
```

Worker：

- D1 repository
- Web Crypto
- KV rate limit/cache

Docker：

- SQLite repository
- Node crypto
- memory / SQLite rate limit

MCP protocol/tool 层尽量共用。

---

## 21. 测试要求

### Unit

必须覆盖：

- alias normalization
- permission matrix
- risk classification
- endpoint allowlist
- path normalization
- SSRF rejection
- secret redaction
- request canonicalization/hash
- approval lifecycle
- approval replay rejection
- idempotency conflict
- audit sanitization

### Integration

使用 fake Cloudflare upstream 或 fetch mock：

- 多账号 route 正确。
- A 账号凭据绝不发给 B。
- GET read 成功。
- 未授权 write 拒绝。
- HIGH 创建 approval。
- approved request 只能执行原始 payload。
- execute 后 audit 完整。
- Cloudflare 4xx/5xx 正确映射。
- 429 不造成写操作重复。

### Existing Regression

现有：

- `/admin/*`
- `/api/*`
- `/v1/*`
- DNS
- Workers
- Storage
- Tunnel
- AI

必须保持回归测试通过。

### MCP Acceptance

至少使用 MCP Inspector 做：

1. initialize
2. list tools
3. accounts_list
4. resources_search
5. read execute
6. forbidden write
7. approval-required write
8. approved execution
9. audit_search
10. invalid auth

---

## 22. 开发阶段 / Work Packages

### WP0 — Baseline & Guardrails

目标：

- 锁定当前 upstream baseline。
- 跑完现有 Worker tests/build。
- 补关键现有接口回归测试。
- 不改功能。

验收：

- `npm test`
- `npm run build`
- 现有前端构建通过。

### WP1 — Shared Service Refactor

目标：

- 把账号获取、Cloudflare API 调用、审计从 route handler 中进一步抽离。
- Web 功能行为不变。

新增：

- accountRegistry
- cloudflareGateway
- auditService

验收：

- 现有 API contract 不变化。
- 所有现有测试通过。

### WP2 — Account AI Metadata

实现：

- ai_alias
- ai_enabled
- ai_permissions
- ai_risk_policy
- migrations
- Account UI

验收：

- 旧数据库可无损升级。
- 旧账号默认 AI disabled。
- alias 唯一。

### WP3 — Permission + Risk Engine

实现：

- Permission Matrix
- Risk classification
- endpoint policy catalog

要求：

- deny by default。

### WP4 — Remote MCP Skeleton

实现：

- `/mcp`
- Streamable HTTP
- MCP auth
- initialize/list tools
- accounts_list
- audit_search

此阶段**只读**。

### WP5 — Cross-account Resource Search

实现：

- resources_search
- zones/dns/workers/pages/kv/d1/r2/tunnels

要求：

- 并发有上限。
- 单账号失败不导致全部失败。
- partial result 明确标记。

### WP6 — Generic Cloudflare API Search / Execute Read-only

实现：

- cloudflare_api_search
- cloudflare_api_execute
- GET only
- endpoint allowlist

这一阶段确认 Code Mode / 通用工具模型可稳定工作。

### WP7 — Write Operations + Approval

实现：

- POST/PUT/PATCH/DELETE policy
- risk engine
- ai_approvals
- Web approval UI
- one-shot approval

### WP8 — Idempotency + Durable Audit

实现：

- ai_operations
- request hash
- idempotency
- structured audit
- actor/request/tool/risk metadata

### WP9 — Typed High-frequency Tools

按实际使用增加：

- DNS upsert
- Worker deploy
- Pages
- Storage

不要为了“工具数量多”扩张。

### WP10 — Security Hardening

完成：

- threat model
- SSRF tests
- secret redaction tests
- auth brute-force throttling
- malformed path
- redirect handling
- approval replay
- account confused-deputy test
- dependency audit

### WP11 — Docker Parity

将成熟 MCP 功能适配 Docker/SQLite。

### WP12 — Release & Docs

输出：

- `docs/mcp.md`
- `docs/mcp-security.md`
- `docs/mcp-clients.md`
- D1 migration docs
- Cloudflare deployment guide
- upgrade / rollback
- release checklist

---

## 23. 第一版 Definition of Done

只有满足以下条件才算 MVP 完成：

- Web 管理后台原有主要功能可用。
- 至少支持 3 个独立 Cloudflare 账号。
- 每账号可配置唯一 AI Alias。
- 每账号独立 AI 开关和权限。
- Remote MCP 能通过公网 HTTPS 接入。
- MCP 与 Admin 使用独立 credential。
- AI 可跨账号搜索资源。
- AI 可执行受控只读 Cloudflare API。
- AI 可提交写操作。
- HIGH / CRITICAL 可进入 Web Approval。
- 批准后只能执行一次且 payload 不可变化。
- AI 操作全部可在 Web Audit 查看。
- 不存在凭据明文返回。
- 通用 execute 无法访问 Cloudflare API 之外的 Host。
- 回归测试通过。
- Worker build/deploy 通过。
- MCP Inspector acceptance 通过。

---

## 24. Codex 执行约束

Codex 开发时遵守：

1. **先读代码再改**，不要依据本文猜文件结构。
2. 每个 WP 单独 commit，commit message 标明 WP。
3. 每个 WP 完成后运行测试和 build。
4. 发现现有实现与本文假设冲突时，以“保持现有行为 + 满足安全边界”为优先原则。
5. 不为了实现 MCP 重写整个 CF Manager。
6. 不把 Secret 写入 fixture、日志、snapshot、Git history。
7. 数据库变化必须 migration-friendly。
8. 对危险操作采用 deny-by-default。
9. 如果 Cloudflare API 某 endpoint 权限/行为不明确，先加测试或文档，不默认放开。
10. 跨账号复制操作必须 preflight，禁止 silent destructive overwrite。
11. 不引入付费服务作为基础依赖。
12. 不移除 Docker 支持；第一阶段可以 Worker-first，但保留后续 parity 的接口边界。

---

## 25. 推荐最终目录形态

```text
cf-manager/
├── docs/
│   ├── AI-MCP-ROADMAP.zh-CN.md
│   ├── mcp.md
│   ├── mcp-security.md
│   └── mcp-clients.md
│
├── shared/
│   ├── cloudflare-api-policy.*
│   └── ...
│
├── worker/src/
│   ├── mcp/
│   ├── services/
│   ├── repositories/
│   ├── migrations/
│   └── ...
│
├── backend/src/
│   ├── mcp/
│   ├── services/
│   ├── repositories/
│   └── ...
│
└── frontend/src/
    ├── views/
    │   └── AiOperations/
    └── ...
```

具体目录最终以代码复用成本为准，不要求机械照搬。

---

## 26. 预期最终体验

### 人工

打开：

```text
https://<your-domain>/admin/
```

可查看和操作所有 Cloudflare 账号。

### AI

添加 MCP：

```text
https://<your-domain>/mcp
```

然后可以使用自然语言：

> 搜索所有账号，找出部署了 obsidian-universal-sync 的位置。

> 检查 dev 账号所有 Workers，告诉我最近部署状态。

> 给 main 账号的 example.com 新增一条 TXT 记录。

> 把 main 的某个 Worker 迁移到 backup，先给我做 preflight，不要直接改。

> 列出过去 24 小时所有 AI 发起的 Cloudflare 写操作。

系统负责把自然语言意图转换为受权限、风险、审批、幂等和审计约束的 Cloudflare API 调用。

---

## 27. 核心设计原则总结

这个改造的目标不是“给 CF Manager 加一个聊天框”，而是把 CF Manager 升级成：

> **Human + AI 共用的 Cloudflare 多账号 Operations Control Plane**

最重要的五条原则：

1. **一个账号库**：Web 与 AI 不维护两套 Cloudflare 凭据。
2. **一个业务层**：人工和 AI 最终走同样的 Cloudflare Gateway。
3. **AI 最小权限**：AI 权限独立于人工后台权限。
4. **危险操作可控**：Risk + Approval + Idempotency + Audit。
5. **MCP 少工具、广覆盖**：通过 Search + Controlled Execute 覆盖 Cloudflare API，而不是堆数千个 Tool。

后续实际开发以 WP0 → WP12 顺序推进，不建议跳过 WP1/WP3 直接实现可写 MCP。
