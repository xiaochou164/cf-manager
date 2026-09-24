# CODEX_IMPLEMENTATION.md

# CF Manager AI / Remote MCP — Codex 交钥匙实施指令

> 本文件是实现指令，不是愿景文档。  
> 设计依据：`docs/AI-MCP-ROADMAP.zh-CN.md`。  
> ChatGPT 插件兼容规范：`docs/CHATGPT-PLUGIN.md`（强制）。  
> 目标：在 **不破坏现有 CF Manager Web 管理后台** 的前提下，把本 Fork 实现为可部署、可测试、可审计的 **Human + AI Cloudflare 多账号 Operations Control Plane**。

---

## 0. Codex 的任务

你需要在当前仓库中持续完成实现，而不是只做分析、方案、伪代码或 TODO。

最终必须交付：

1. 原有 Web Admin 继续可用。
2. Cloudflare Worker / Pages 版本拥有标准 Remote MCP Endpoint。
3. MCP 可管理多个彼此独立的 Cloudflare 账号。
4. Web 与 MCP 共用同一套账号、凭据、Cloudflare API 服务层和审计数据。
5. 每个账号可以单独控制 AI 是否可访问及 AI 权限。
6. AI 支持跨账号资源搜索。
7. AI 支持受控 Cloudflare API 查询与写操作。
8. 高风险操作必须经过 Web Admin 人工批准。
9. 写操作具备幂等保护。
10. MCP 的所有敏感操作具备结构化审计。
11. 凭据不会通过 MCP、日志、错误信息泄露。
12. Cloudflare Worker / Pages + D1 + KV 仍是第一优先部署方式，并以个人 Free Plan 可运行作为约束。
13. Docker 版本不能被破坏；Worker MVP 稳定后再补齐 Docker parity。
14. CI 必须通过。
15. 不允许以“后续再实现”为理由留下 MVP 必需功能的空壳。
16. ChatGPT 个人/自定义插件必须可通过标准 Remote MCP + OAuth 2.1 接入；静态 MCP Secret 仅作为通用客户端兼容方式。
17. 最终必须生成并验证可移植 Plugin package（plugin.json + mcp.json + skills）。

实现过程中以本文件为执行约束，以 `docs/AI-MCP-ROADMAP.zh-CN.md` 为设计说明。若两者有冲突，以本文件中的安全边界和验收条件优先。

---

# 1. 开始工作前必须做的事情

先检查仓库，不要直接写代码。

执行：

```bash
git status
git branch --show-current
git rev-parse HEAD
git log -10 --oneline
```

阅读至少以下内容：

```text
README.zh-CN.md
docs/AI-MCP-ROADMAP.zh-CN.md
docs/account-auth.md
docs/deploy.md

worker/package.json
worker/src/index.ts
worker/src/types.ts
worker/src/db/schema.sql
worker/src/db/models.ts
worker/src/middleware/auth.ts
worker/src/services/cfApi.ts
worker/src/services/encryption.ts
worker/src/routes/accounts.ts
worker/src/routes/dns.ts
worker/src/routes/workers.ts
worker/src/routes/storage.ts
worker/src/routes/tunnels.ts

backend/package.json
backend/src/index.ts
backend/src/db.ts

frontend/package.json
frontend/src/router/*
frontend/src/views/*
frontend/src/api/*

scripts/check-db-schema.mjs
.github/workflows/ci.yml
```

再搜索：

```bash
rg "audit_log|addAuditLog|cfFetch|getAuthHeaders|ENCRYPTION_KEY|API_SECRET" worker backend frontend
rg "CREATE TABLE|ALTER TABLE|migration|schema" worker backend scripts
rg "accounts|Account" frontend/src worker/src backend/src
```

先理解现有架构、数据库迁移方式、测试组织方式、共享代码同步机制，再开始修改。

---

# 2. Git / 分支规则

从当前 `master` 建立实施分支：

```text
feat/ai-mcp-control-plane
```

不要重写 master 历史。

每个 Work Package 独立 commit，commit message 建议：

```text
WP0: lock baseline and add MCP regression coverage
WP1: extract shared Cloudflare service layer
WP2: add AI account metadata and migrations
WP3: add permission risk and endpoint policy engines
WP4: add read-only remote MCP server
WP5: add cross-account resource search
WP6: add controlled Cloudflare API read execution
WP7: add write approvals
WP8: add idempotent AI operations and structured audit
WP9: add typed high-frequency MCP tools
WP10: harden MCP security boundaries
WP11: add Docker MCP parity
WP12: finalize docs and release validation
```

不要把多个 WP 混成一个巨大 commit。

如果当前环境无法提交 commit，可以完成全部代码，但最终报告必须列出建议 commit 切分；有 Git 写权限时应实际提交。

---

# 3. 当前仓库已有 CI 契约

不得绕过、删除或弱化现有 CI。

当前 CI 的关键检查为：

### backend

```bash
cd backend
npm ci
npm run build
npm run typecheck
npm run lint
npm run test
```

### frontend

```bash
cd frontend
npm ci
npx vite build
npm run typecheck
npm run lint
```

### worker

```bash
cd worker
npm ci
node ../scripts/gen-version.js worker
node ../scripts/sync-shared.js
mkdir -p public
npm run build:worker
npm run typecheck
npm run lint
npm run test
```

### schema alignment

仓库根目录：

```bash
node scripts/check-db-schema.mjs
```

每个 WP 至少运行其受影响模块的 test/typecheck；涉及跨层、数据库、shared 或发布行为的 WP 必须运行完整 CI 等价命令。

**禁止通过删除测试、放宽 lint、添加无意义 ignore、跳过 schema check 来“修复”CI。**

---

# 4. 不允许破坏的现有接口

以下入口必须保持兼容：

```text
/admin/*
/api/*
/v1/*
/api/v1/*
```

现有：

- 多账号管理
- DNS
- Workers
- Pages
- KV / D1 / R2
- Tunnel
- Rules
- AI Workspace
- Browser Rendering
- OpenAI-compatible API
- Audit

不得因 MCP 改造而被重写成另一套产品。

特别注意：

- `/v1/*` 是 OpenAI-compatible API。
- `/mcp` 是 Remote MCP。
- 两者协议、鉴权、错误结构必须独立。
- 不允许用现有 `API_SECRET` 直接替代新的 MCP 凭证模型。
- 不允许因为 `API_SECRET` 未配置而让 `/mcp` 自动变成匿名访问。

---

# 5. 目标架构

目标结构：

```text
                        CF Manager
                            │
          ┌─────────────────┴─────────────────┐
          │                                   │
      Web Admin                           Remote MCP
      /admin/*                              /mcp
          │                                   │
          └─────────────────┬─────────────────┘
                            │
                  Unified Service Layer
                            │
     ┌──────────────────────┼────────────────────────┐
     │                      │                        │
Account Registry     Permission / Risk        Audit / Approval
     │                      │                        │
     └──────────────────────┼────────────────────────┘
                            │
                 Cloudflare API Gateway
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
      Account main      Account dev     Account backup
```

核心原则：

> Route handler 只负责协议适配和参数验证，业务规则放 Service 层；Web 与 MCP 最终调用相同的 Service / Gateway，不复制两套 Cloudflare 调用逻辑。

---

# 6. MCP 技术约束

## 6.1 Transport

实现标准 Remote MCP，优先采用当前官方 MCP SDK 对 **Streamable HTTP** 的实现。

在编码前确认项目可在 Cloudflare Workers runtime 运行。不要默认 Node-only transport 可以工作。

如果当前官方 SDK 的 API 与设计文档命名不同：

- 使用官方当前稳定 API。
- 保持协议兼容。
- 在 `docs/mcp.md` 记录实际 SDK 和 transport。
- 不要自己手写一个“看起来像 MCP”的私有 JSON API。

Endpoint：

```text
/mcp
```

至少支持 MCP：

- initialize
- tools/list
- tools/call

如 SDK 需要会话管理，应选择 Cloudflare Worker 可兼容的方案；个人低并发场景优先保持轻量，避免第一版强依赖付费资源。

---

# 7. MCP 鉴权必须 fail closed

新增 Worker Env：

```text
MCP_ENABLED
MCP_AUTH_SECRET
```

行为：

### MCP_ENABLED != true

```text
/mcp -> 404 或明确 MCP_DISABLED
```

不要“配置缺失就匿名开放”。

### MCP_ENABLED = true 但 MCP_AUTH_SECRET 缺失

启动/请求必须失败关闭：

```text
MCP_MISCONFIGURED
```

### 正常请求

要求：

```http
Authorization: Bearer <MCP_AUTH_SECRET>
```

要求：

- MCP Secret 与 `API_SECRET` 独立。
- 比较避免明显 timing leak。
- 日志不输出 Authorization header。
- 401/403 日志脱敏。
- 为失败认证增加轻量限流。
- CORS 不是安全边界。
- 不允许 query string 携带 secret。

后续 WP 可增加 `mcp_clients` 表和多 Client Token；MVP 至少先做到独立高熵 secret。

---

# 8. Account AI Metadata

扩展账号模型，至少包括：

```text
ai_alias
ai_enabled
ai_permissions
ai_risk_policy
last_verified_at
```

要求：

### ai_alias

- 唯一。
- 稳定。
- 不区分大小写时必须明确规范化。
- 建议规则：`^[a-z0-9][a-z0-9_-]{0,62}$`。
- 不允许 `all`、`system` 等保留字与实际账号 alias 冲突。
- Web UI 必须能编辑。
- AI 返回账号时同时提供 `name` 与 `ai_alias`，但模型操作优先使用 alias。

### ai_enabled

旧账号 migration 后：

```text
ai_enabled = false
```

必须由用户主动开启。

### ai_permissions

第一版至少支持：

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

解析失败时：

> deny，不能 fallback 到全权限。

---

# 9. 数据库迁移规则

不要只修改 `schema.sql` 然后假设已有部署会自动升级。

需要：

1. 更新全新安装 schema。
2. 新增可重复执行/可检测状态的 migration。
3. 同步 Worker D1 和 backend SQLite schema。
4. 保持 `scripts/check-db-schema.mjs` 通过。
5. migration 失败不能留下部分安全策略已启用、部分字段缺失的模糊状态。
6. 为 migration 增加测试。

至少新增：

- account AI metadata
- `ai_approvals`
- `ai_operations`
- audit 扩展字段

如 SQLite 与 D1 在 ALTER/index 能力上存在差异，为两个 adapter 正确处理，不要用假兼容 SQL。

---

# 10. Shared Service Layer

在 WP1 完成之前不要直接从 MCP route 到处调用现有 REST route。

至少抽象：

```ts
AccountRegistry
CloudflareGateway
AuditService
PermissionEngine
RiskEngine
ApprovalService
OperationService
ResourceSearchService
```

当前 `worker/src/services/cfApi.ts` 可以演进为 `CloudflareGateway` 的底层能力。

要求：

- 凭据解密只发生在 Gateway 边界附近。
- 上层永远拿不到解密后的 Token。
- Account A 的 auth header 不可能被复用到 Account B。
- Cloudflare error 在返回 Agent 前必须经过 sanitizer。
- Web Route 与 MCP Tool 尽量复用 Gateway / Service。

不要为了“架构漂亮”一次性重写全部旧 route。采用渐进式抽取，并用 regression tests 锁行为。

---

# 11. MCP Tool 契约

MVP 的 MCP 工具数量保持少。

必须实现：

```text
accounts_list
resources_search
cloudflare_api_search
cloudflare_api_execute
audit_search
approval_status
```

---

## 11.1 accounts_list

返回 AI 有权限看到的账号。

不得返回：

- api_token
- api_key
- Global API Key
- Authorization header
- encryption ciphertext
- MCP secret

允许返回：

```json
{
  "name": "个人主账号",
  "alias": "main",
  "cloudflare_account_id": "...",
  "active": true,
  "features": ["workers", "dns"],
  "permissions": {
    "read": true,
    "deploy": true
  }
}
```

默认排除：

- ai_enabled=false
- disabled account

---

## 11.2 resources_search

必须支持跨账号。

输入语义至少：

```json
{
  "query": "obsidian-universal-sync",
  "account": "all",
  "types": ["workers", "pages", "dns", "d1", "kv", "r2", "tunnels"]
}
```

account 可为：

- 具体 alias
- all

但 `all` 只能搜索 ai_enabled 且拥有 read 权限的账号。

要求：

- 并发上限。
- timeout。
- partial failure。
- 每个结果带 account alias。
- 不能因为一个账号 403/429 导致其他账号结果丢失。
- 限制单次返回数量。
- 可用 KV 做短时 cache，但不能缓存 secret。

统一返回：

```json
{
  "results": [],
  "partial": false,
  "errors": []
}
```

---

## 11.3 cloudflare_api_search

不要把 Cloudflare 数千 API endpoints 都注册为 MCP tools。

建立受控 policy catalog。

搜索返回：

```json
{
  "method": "GET",
  "path_template": "/accounts/{account_id}/workers/scripts",
  "permission": "read",
  "risk": "LOW",
  "allowed": true
}
```

MVP policy catalog 先覆盖当前 CF Manager 已实现产品使用到的 endpoints。

未收录 endpoint：

```text
allowed = false
```

默认 deny。

---

## 11.4 cloudflare_api_execute

这是核心安全边界。

输入：

```json
{
  "account": "dev",
  "method": "GET",
  "path": "/accounts/{account_id}/workers/scripts",
  "query": {},
  "body": null,
  "reason": "List workers in dev",
  "idempotency_key": null,
  "approval_id": null
}
```

### 绝对禁止

模型传入：

```text
https://...
http://...
//host/...
任意 upstream host
```

服务端 origin 必须硬编码：

```text
https://api.cloudflare.com/client/v4
```

### path 规则

- normalize 后必须以单个 `/` 开头。
- reject backslash confusion。
- reject control chars。
- reject encoded host confusion。
- reject path traversal。
- endpoint 必须匹配 policy。
- path template 中 account_id / zone_id 必须经过 server-side resource resolution 或明确验证。
- 不允许通过 URL trick 绕过固定 host。

### Redirect

Cloudflare Gateway 对敏感请求不要无条件跟随跨 host redirect。

如 fetch runtime 默认 follow，应采用能确保最终 host 仍是 Cloudflare API 的安全策略，或禁止 redirect 并显式处理。

### Header

客户端不能覆盖：

- Authorization
- X-Auth-Key
- X-Auth-Email
- Host
- Cookie

凭据只能由 Gateway 注入。

### Response

执行 response 经过 redaction：

- auth token
- api key
- secret binding value
- cookie
- known credential fields

---

# 12. Endpoint Policy Engine

创建 version-controlled policy catalog，例如：

```text
shared/cloudflare-api-policy.json
```

或类型安全的 TS 数据结构。

每条至少：

```json
{
  "id": "dns.records.create",
  "method": "POST",
  "pathPattern": "/zones/:zone_id/dns_records",
  "permission": "dns_write",
  "risk": "MEDIUM",
  "allowed": true
}
```

风险等级：

```text
LOW
MEDIUM
HIGH
CRITICAL
```

默认建议：

- 读查询：LOW
- 一般新增/修改：MEDIUM
- deploy / purge / 有明显影响的写：HIGH
- delete / destructive：CRITICAL

**所有未知 endpoint = DENY。**

---

# 13. Permission Engine

服务端必须同时检查：

1. MCP 身份合法。
2. account 存在。
3. account active。
4. `ai_enabled=true`。
5. 请求动作需要的 permission 已授权。
6. endpoint policy allowed。
7. risk policy 允许。
8. approval 状态满足要求。

任何一层失败都不得继续到 Cloudflare API。

不要相信模型传入的：

- risk
- permission
- account_id

这些必须由服务端计算/解析。

---

# 14. Approval 模型

HIGH / CRITICAL 操作走审批。

默认：

- LOW：直接执行。
- MEDIUM：直接执行 + 强审计。
- HIGH：需要 approval，除非账号 risk policy 明确允许。
- CRITICAL：始终需要 Web Approval。

MVP 不允许 CRITICAL 全局免确认。

## 14.1 首次请求

Agent 调用 `cloudflare_api_execute`：

系统 canonicalize 请求，计算 request hash。

返回：

```json
{
  "code": "APPROVAL_REQUIRED",
  "approval_id": "...",
  "request_id": "...",
  "risk": "HIGH",
  "expires_at": "..."
}
```

## 14.2 Web Admin

新增：

```text
AI Operations
└── Pending Approvals
```

显示：

- account alias
- operation
- method/path
- target
- reason
- risk
- payload summary
- created at
- expires at

按钮：

- Approve
- Reject

**默认 Approve 只授权，不在浏览器里偷偷替 AI 改写 payload。**

## 14.3 重试执行

AI 重新调用 execute，并携带：

```text
approval_id
```

服务端重新 canonicalize 请求，并校验：

```text
hash(current_request) == approved_request_hash
```

一致才允许执行。

执行前必须原子化 claim：

```text
approved -> executing
```

避免 approval 被并发重放。

成功：

```text
executed
```

失败：

记录失败原因；是否允许同一 approval 重试必须设计明确。默认对于不确定是否已提交的 destructive/write 请求，不自动重试。

---

# 15. Idempotency

所有 MCP 写操作必须支持 `idempotency_key`。

表：

```text
ai_operations
```

至少字段：

- id
- idempotency_key
- account_id
- tool_name
- request_hash
- state
- result_json
- error_json
- created_at
- updated_at

状态：

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

规则：

- 相同 key + 相同 request hash：返回已有状态/结果。
- 相同 key + 不同 hash：`IDEMPOTENCY_CONFLICT`。
- 不能因为 HTTP retry 重复创建 DNS、重复 deploy 或重复 delete。
- 对 Cloudflare 429 / 5xx 的重试必须考虑“服务端是否已经执行”的不确定性。

---

# 16. Audit

扩展现有 audit，不另起一套互不关联日志。

至少支持：

```text
actor_type: human | mcp | system
actor_id
request_id
tool_name
risk_level
duration_ms
metadata
```

每次 MCP 调用至少记录：

- actor
- account
- tool
- action
- target
- reason
- risk
- approval_id
- idempotency key 摘要
- status
- error code
- duration
- request id

禁止记录：

- Token
- API Key
- Authorization
- Secret binding plaintext
- MCP Secret

为日志 sanitizer 写单测。

---

# 17. Web Admin UI

不要给 Web Admin 增加“聊天机器人”。这里做的是运维控制台。

至少实现：

## Account 编辑

新增：

- Enable AI Management
- AI Alias
- AI Permissions
- Risk Policy
- Last Verified

权限建议使用 checkbox / grouped permissions。

## AI Operations

新增页面：

```text
AI Operations
├── Overview
├── Pending Approvals
└── Audit
```

Overview 至少：

- 24h MCP calls
- success/error
- pending approvals
- high/critical count
- recent activity

## Settings / MCP

显示：

- MCP enabled
- Endpoint URL
- Auth configured / not configured
- health
- 安全提示

如果 secret 存 Env，Web 不应展示其明文。

---

# 18. Resource Search 第一版覆盖

至少：

### Workers

- script name
- account alias
- modified/deployment info（API 可得时）

### Pages

- project name
- account alias
- domains / deployment summary（API 可得时）

### DNS

- zone
- record name
- type
- content 可按敏感策略决定是否完整返回；普通 DNS 内容可返回

### D1

- database name / UUID

### KV

- namespace title / ID

### R2

- bucket name

### Tunnel

- tunnel name / ID / status

不要把 R2 objects 全量扫描作为默认 search；那可能成本高。资源搜索优先搜索 bucket / namespace / DB / script / project 等控制面对象。

---

# 19. Typed Tools

WP9 才增加 typed tools。

第一批建议：

```text
dns_record_list
dns_record_upsert
worker_get
worker_deploy
pages_project_list
storage_list
```

Typed Tool 的价值是强 schema / 强 validation，不是重复包装所有 API。

任何 typed write tool 最终仍要走：

```text
Permission -> Risk -> Approval -> Idempotency -> Gateway -> Audit
```

禁止 typed tool 绕过通用安全层。

---

# 20. Worker Secret / Binding 迁移规则

跨账号 Worker 迁移是重要用例，但要尊重 Cloudflare API 能力。

对源 Worker：

- 可读取的普通配置可以迁移。
- Secret binding 的明文如果 Cloudflare API 不提供，就不能伪造“已复制”。

Preflight 必须明确输出：

```text
secret X: VALUE_UNAVAILABLE
requires target secret input
```

不得在日志或 UI 中把 secret value 伪装成可读。

跨账号迁移默认流程：

```text
read source
→ normalize
→ inspect target
→ dependency diff
→ plan
→ approval
→ create dependencies
→ deploy
→ verify
→ audit
```

---

# 21. Security Test Matrix

WP10 前必须补齐。

至少覆盖：

## Auth

- MCP disabled。
- missing secret。
- wrong bearer。
- empty bearer。
- malformed header。
- API_SECRET 不能当 MCP secret 使用。

## Account isolation

- A token 永不请求 B account。
- alias 不可解析时拒绝。
- ai_enabled=false 拒绝。
- disabled account 拒绝。

## URL / SSRF

拒绝：

```text
https://evil.com/
http://evil.com/
//evil.com
/%2f%2fevil.com
\evil.com
/../
encoded traversal
control characters
userinfo-host confusion
```

## Header injection

请求 body/query/header 不能覆盖 auth headers。

## Policy

- unknown endpoint 拒绝。
- method mismatch 拒绝。
- GET policy 不能用 DELETE。
- zone/account ownership mismatch 拒绝或重新解析。

## Approval

- pending 不能执行。
- rejected 不能执行。
- expired 不能执行。
- approval A 不能用于请求 B。
- approval payload 改 1 个字段即不能复用。
- approval 不能执行两次。
- 并发两次只有一次 claim 成功。

## Idempotency

- same key/same payload。
- same key/different payload conflict。
- concurrent duplicate。
- 429。
- upstream timeout。

## Redaction

伪造错误响应包含：

- Bearer token
- X-Auth-Key
- email/key
- secret

最终 Agent/Audit 都不得出现敏感值。

---

# 22. WP0 — Baseline & Guardrails

目标：建立修改前基线。

完成：

1. 记录当前 HEAD。
2. 跑完整 CI 等价检查。
3. 记录已有失败，不要把已有失败误算成新回归。
4. 为关键已有路由增加必要 regression tests。
5. 不修改产品行为。

输出：

```text
docs/implementation/WP0-BASELINE.md
```

记录：

- HEAD
- Node 版本
- commands
- tests count
- failures
- known issues

只有 baseline 明确后进入 WP1。

---

# 23. WP1 — Shared Service Refactor

目标：为 Web/MCP 共用业务层准备。

不要改 UI 功能。

实现：

- AccountRegistry
- CloudflareGateway
- AuditService

把新增 MCP 会用到的底层能力先从 route 中抽离。

要求：

- 现有 REST response contract 不变。
- 现有 account credentials 行为不变。
- cfFetch 现有调用可以渐进迁移。
- 增加账号隔离单测。

验收：

完整 worker test/typecheck/lint/build。

---

# 24. WP2 — Account AI Metadata + UI

实现数据库 migration、model、API、frontend。

默认：

```text
ai_enabled=false
```

新增 UI 后，老用户不配置 AI 时行为完全不变。

验收：

- 新安装。
- 老 schema migration。
- alias duplicate。
- invalid alias。
- malformed permissions。
- frontend build/typecheck。
- schema alignment。

---

# 25. WP3 — Permission / Risk / Endpoint Policy

实现纯逻辑优先，先不要开放写 MCP。

要求高测试覆盖。

核心函数尽量可无 Cloudflare 环境单测：

```ts
resolveAccount()
resolvePolicy()
requiredPermission()
classifyRisk()
authorizeAction()
canonicalizeRequest()
```

验收：

所有未知 API deny by default。

---

# 26. WP4 — Read-only Remote MCP

实现：

- `/mcp`
- MCP authentication
- initialize
- tools/list
- `accounts_list`
- `audit_search`

MCP 在本 WP 强制只读。

即便 Agent 构造写方法，也必须拒绝。

新增：

```text
docs/mcp.md
```

写清客户端配置和 curl/Inspector 健康检查方法，但文档中使用占位 secret。

---

# 27. WP5 — Cross-account Resource Search

实现 `resources_search`。

要求：

- bounded concurrency
- timeout
- partial failures
- per-account permission
- normalized result
- result limit
- optional KV cache

测试至少模拟 3 个账号：

```text
main -> success
dev -> 403
backup -> success
```

结果必须返回 main + backup，并显式标记 dev error。

---

# 28. WP6 — Generic API Search + Read Execute

实现：

- `cloudflare_api_search`
- `cloudflare_api_execute`

此 WP 只允许安全 read endpoint。

即使 policy 里未来存在 write 定义，也先通过全局 feature gate 阻止写执行。

目标是先验证：

> 少量 MCP tools + API catalog + controlled execute

是否稳定。

---

# 29. WP7 — Write + Approval

再开放写能力。

先选择少量典型 endpoint：

- DNS create/update
- Worker deploy/config update

Delete 类即便实现 policy，也先保持 CRITICAL + approval。

实现：

- `ai_approvals`
- Web approval UI
- `approval_status`
- request hash
- one-shot claim

验收必须包含真实生命周期测试。

---

# 30. WP8 — Idempotency + Structured Audit

写操作没有 WP8 就不算可发布。

实现：

- `ai_operations`
- idempotency key
- state machine
- expanded audit
- error sanitization
- timing/duration

写 state transition tests。

禁止用“先查有没有资源”代替严格的 idempotency ledger。

---

# 31. WP9 — Typed High-frequency Tools

只根据实际已有 Cloudflare Service 能力增加。

不要一次性生成上百 MCP tools。

typed tool 与 generic execute 必须共享 policy/service，不复制安全逻辑。

---

# 32. WP10 — Security Hardening

执行 threat model。

新增：

```text
docs/mcp-security.md
```

文档至少说明：

- trust boundaries
- credential storage
- MCP authentication
- permissions
- endpoint allowlist
- approval
- idempotency
- SSRF
- log redaction
- known limitations

跑安全矩阵全部测试。

做 dependency audit，但不要为追求 0 warning 盲目 major upgrade 整个项目。

---

# 33. WP11 — Docker Parity

Worker 版本通过以后，再给 backend 增加相同 MCP 能力。

要求：

- Service contract 尽量共享。
- SQLite migrations 对齐。
- Node 与 Worker 的行为和 error codes 尽量一致。
- Docker 不强依赖 D1/KV。

如果某 Cloudflare runtime feature 无法在 Docker 完全等价，文档明确差异，而不是 silent fallback。

---

# 34. WP12 — Release / Deployment

完成：

```text
docs/mcp.md
docs/mcp-security.md
docs/mcp-clients.md
docs/implementation/*
```

更新：

- README.zh-CN.md
- README.md（至少增加 MCP feature/文档入口）
- docs/deploy.md
- .env.example / Worker env docs
- deployment workflows（仅在确有必要时）

部署文档必须说明新增：

```text
MCP_ENABLED
MCP_AUTH_SECRET
```

并强调：

> MCP_ENABLED=true 时 MCP_AUTH_SECRET 必须设置。

---

# 35. Cloudflare Staging 验收

如果 Codex 环境已有合法 Cloudflare 测试凭据，可部署到**隔离 staging**。

不得默认操作用户生产资源。

建议 staging：

```text
cf-manager-ai-staging
cf-manager-ai-staging-db
cf-manager-ai-staging-kv
```

真实 Cloudflare smoke 只允许：

- 测试账号。
- 新建的测试 Zone/Worker/资源。
- 或用户明确授权的 staging 资源。

没有凭据时：

- 完成本地/mock/integration。
- 不伪造“Cloudflare staging PASS”。

---

# 36. MCP Acceptance Suite

至少完成以下 acceptance：

### A1 Initialization

MCP client 可以 initialize。

### A2 Auth

错误 secret 被拒绝。

### A3 Accounts

`accounts_list` 只返回 AI-enabled 账号。

### A4 Cross-account search

跨 3 个账号查询成功并正确处理 partial error。

### A5 Read execute

允许 policy 内 GET。

### A6 Unknown endpoint

明确拒绝。

### A7 Unauthorized write

账号无 write 权限时拒绝。

### A8 Approval

HIGH write 返回 approval id。

### A9 Approval tamper

approve 后修改 body，执行被拒绝。

### A10 Approval replay

同一个 approval 第二次执行被拒绝。

### A11 Idempotency

网络重试不会重复写。

### A12 Audit

每个操作可通过 audit_search 查到，且无 secret。

### A13 Admin

Pending Approval 可以在 Web 查看、批准、拒绝。

### A14 Regression

Admin / existing APIs 继续工作。

---

# 37. 最终完整检查命令

交付前至少执行：

```bash
# Backend
cd backend
npm ci
npm run build
npm run typecheck
npm run lint
npm run test
cd ..

# Frontend
cd frontend
npm ci
npx vite build
npm run typecheck
npm run lint
cd ..

# Worker
cd worker
npm ci
node ../scripts/gen-version.js worker
node ../scripts/sync-shared.js
mkdir -p public
npm run build:worker
npm run typecheck
npm run lint
npm run test
cd ..

# Schema
node scripts/check-db-schema.mjs
```

另外运行项目新增的 MCP acceptance/integration 测试。

不要只报告“应该通过”。

必须保存实际结果。

---

# 38. 最终交付报告

创建：

```text
docs/implementation/FINAL-ACCEPTANCE.md
```

包含：

## Candidate

- branch
- HEAD
- base commit
- changed file count

## Implemented

逐项列出 WP0-WP12：

```text
PASS / PARTIAL / NOT APPLICABLE
```

PARTIAL 必须解释。

## Tests

列出实际：

- suites
- tests
- pass/fail
- commands

## Security

列出：

- SSRF
- auth
- account isolation
- approval replay
- idempotency
- redaction

测试结果。

## Deployment

- Worker build
- local test
- staging（若实际执行）
- production 未执行则明确写未执行

## Known limitations

只允许留下非 MVP 阻断项。

## Rollback

给出：

- code rollback
- D1 migration compatibility
- MCP disable switch

---

# 39. Stop Conditions

不要因为普通实现困难停下来向用户询问。

以下情况才允许停：

1. 需要用户提供真实 Cloudflare secret 才能做真实 staging。
2. 第三方服务不可用，且 mock/local 已无法进一步验证。
3. 发现会造成生产数据破坏，且无法通过 staging / mock 安全验证。
4. Roadmap 与现有项目许可/架构存在无法兼容的硬冲突。

即使触发 Stop Condition，也要：

- 完成所有不依赖该阻塞项的工作。
- 提供精确 blocker。
- 提供已经完成的 commit。
- 不把“需要真实 secret”当成不写测试的理由。

---

# 40. 禁止行为

Codex 不得：

- 删除/绕过现有 CI。
- 把 Token 写进 repo。
- 把生产 Cloudflare credential 放进 test fixture。
- 关闭 TypeScript 检查来通过构建。
- 用 `any` 大面积绕过核心安全类型。
- 让 MCP 在无 secret 时开放。
- 允许 arbitrary URL fetch。
- 允许 unknown Cloudflare endpoint 默认通过。
- 允许 Agent 自己批准自己的 CRITICAL 操作。
- 允许 approval 脱离 request hash。
- 把 Global API Key 输出给 Agent。
- 假装复制不可读取的 Worker secrets。
- 自动对生产资源跑 destructive smoke。
- 为了实现 MCP 大规模删除现有功能。
- 把所有 Cloudflare API 生成为上千 MCP tools。
- 只写 TODO / placeholder 然后宣称 WP 完成。

---

# 41. 质量优先级

发生取舍时，按以下顺序：

```text
1. Credential / account isolation safety
2. Destructive-operation safety
3. Existing feature compatibility
4. Protocol correctness
5. Data consistency / idempotency
6. Auditability
7. Cloudflare Free-tier efficiency
8. UI polish
```

不要为了 UI 或代码“漂亮”牺牲 1–6。

---

# 42. 实施完成的最终定义

只有以下全部成立，才可以说“交钥匙版本完成”：

- [ ] 原 Admin 可用
- [ ] 多账号可用
- [ ] AI Alias 可配置
- [ ] 每账号 AI 独立开关
- [ ] AI 权限生效
- [ ] Remote MCP 可连接
- [ ] MCP 单独鉴权且 fail closed
- [ ] accounts_list 可用
- [ ] resources_search 跨账号可用
- [ ] cloudflare_api_search 可用
- [ ] read execute 可用
- [ ] write execute 受权限控制
- [ ] HIGH / CRITICAL Approval 可用
- [ ] approval 防篡改
- [ ] approval 防 replay
- [ ] idempotency 可用
- [ ] Audit 可查
- [ ] Secret redaction 有测试
- [ ] arbitrary upstream / SSRF 被拒绝
- [ ] Worker tests PASS
- [ ] Worker typecheck PASS
- [ ] Worker lint PASS
- [ ] Worker build PASS
- [ ] Frontend build/typecheck/lint PASS
- [ ] Backend build/typecheck/lint/test PASS
- [ ] Schema check PASS
- [ ] MCP acceptance PASS
- [ ] 文档完成
- [ ] rollback 方法明确

如果其中任何 MVP 项未完成，不得用“基本完成”“核心完成”模糊代替，应明确列出缺失项。

---

# 43. 给 Codex 的启动指令

在 Codex 中可以直接使用：

```text
读取仓库根目录 CODEX_IMPLEMENTATION.md 和 docs/AI-MCP-ROADMAP.zh-CN.md。

你现在负责将这个 Fork 实现为可交付的 Cloudflare 多账号 Web Admin + Remote MCP AI Operations Control Plane。

严格按 CODEX_IMPLEMENTATION.md 的 WP0 → WP12 顺序执行。先完成 baseline、测试和架构检查，然后持续实现，不要停留在方案讨论，不要只输出代码建议。

要求：
1. 保留现有 CF Manager 功能和 API 兼容性。
2. Worker/Pages + D1 + KV 为第一优先实现。
3. 每个 WP 完成后运行对应 test/typecheck/lint/build，并记录实际结果。
4. 每个 WP 独立 commit。
5. 对 Cloudflare API 采用 deny-by-default endpoint policy。
6. MCP 必须独立鉴权、fail closed。
7. 所有写操作必须经过 Permission/Risk；高风险操作走人工 Approval；写操作具备 Idempotency。
8. 不得泄露任何 Cloudflare credential 或 MCP secret。
9. 不允许 arbitrary URL fetch。
10. 不要询问普通实现细节；按仓库现状自行做最安全、最兼容的工程判断。
11. 只有需要真实 Cloudflare staging secret 或存在明确生产破坏风险时才停下说明 blocker。
12. 最终生成 docs/implementation/FINAL-ACCEPTANCE.md，并给出实际测试结果、commit 列表、已知限制和部署/回滚说明。

现在从 WP0 开始，先读取代码、建立 baseline、运行现有 CI 等价检查，然后继续后续 WP。
```

---

# 44. 与 Roadmap 的关系

`docs/AI-MCP-ROADMAP.zh-CN.md` 回答：

> 我们要做什么、为什么这么设计。

本文件回答：

> Codex 具体怎么做、按什么顺序做、怎么证明做完了。

实施中必须同时保留两份文档。


---

# 45. ChatGPT Plugin 兼容要求（强制）

实现时同时读取 `docs/CHATGPT-PLUGIN.md`。

本节修正原文中“仅以 MCP_AUTH_SECRET 作为 MVP 鉴权”的假设：

- `MCP_AUTH_SECRET` 继续支持 Generic MCP Client。
- ChatGPT Plugin 必须支持符合 MCP Authorization 规范的 OAuth 2.1。
- ChatGPT OAuth 使用 Authorization Code + PKCE S256。
- 暴露 Protected Resource Metadata 与 Authorization Server Metadata。
- Tool 根据真实权限声明 securitySchemes。
- Tool annotation 必须与副作用一致。
- OAuth Scope 与 Account AI Permission 取交集，不能互相替代。
- ChatGPT Host confirmation 不能替代 CF Manager 的 HIGH/CRITICAL Approval。
- MVP 支持 ChatGPT Developer Mode 直接以 `https://<domain>/mcp` 创建个人插件。
- WP12 必须增加 portable Plugin package generator：
  - `plugin.json`
  - `mcp.json`
  - `skills/cloudflare-ops/SKILL.md`
- 部署 URL 不得硬编码进仓库；由生成脚本注入。
- 增加 ChatGPT OAuth / package schema acceptance tests。

最终验收中，如果 Generic MCP PASS 但 ChatGPT OAuth / Plugin 连接未完成，不得标记“交钥匙完成”。
