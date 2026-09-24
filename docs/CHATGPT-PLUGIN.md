# ChatGPT 自定义插件 / MCP 接入设计

> 本文是 `docs/AI-MCP-ROADMAP.zh-CN.md` 与 `CODEX_IMPLEMENTATION.md` 的强制补充规范。  
> 目标：让 CF Manager 的 Remote MCP 不仅能被通用 MCP Client 使用，还能被 ChatGPT 作为个人/自定义插件直接连接，并保留后续打包发布为标准 Plugin 的能力。

---

## 1. 目标体验

### 自用

部署完成后，在 ChatGPT Developer Mode 中直接连接：

```text
https://<cf-manager-domain>/mcp
```

ChatGPT 将该 Remote MCP 建立为个人插件。

之后可以在 ChatGPT / Work 中：

```text
@CF Manager
检查所有 Cloudflare 账号的 Workers
```

或：

```text
@CF Manager
在 dev 账号的 example.com 新增 TXT 记录
```

仍然必须经过 CF Manager 自己的权限、Risk、Approval、Idempotency 与 Audit。

---

## 2. 关键架构

```text
                         CF Manager
                             │
                    https://.../mcp
                             │
          ┌──────────────────┴──────────────────┐
          │                                     │
   Generic MCP Clients                      ChatGPT
 Hermes / custom agent                Personal / Custom Plugin
          │                                     │
   Static bearer or OAuth                    OAuth 2.1
          │                                     │
          └──────────────────┬──────────────────┘
                             │
                      MCP Identity Layer
                             │
                 Permission / Risk / Audit
                             │
                    Cloudflare Gateway
                             │
               multiple Cloudflare accounts
```

MCP Tool 层只有一套。

**禁止为 ChatGPT 单独复制第二套 Cloudflare API 路由或权限逻辑。**

---

## 3. 鉴权策略必须升级为 Dual Auth

原设计中的：

```text
MCP_AUTH_SECRET
```

继续保留，但只作为通用/可信 MCP Client 的一种连接方式。

ChatGPT Plugin 路径需要标准 OAuth 2.1。

原因：

- ChatGPT MCP 集成不会依赖用户自定义 API Key header 作为标准插件身份方案。
- 私有数据和写操作必须有明确用户身份与授权。
- ChatGPT 通过 OAuth access token 调用 MCP。

目标：

```text
MCP auth modes:
- static_bearer
- oauth2
```

服务端统一解析成：

```ts
interface McpPrincipal {
  id: string;
  authType: 'static_bearer' | 'oauth2';
  scopes: string[];
  clientType?: 'chatgpt' | 'generic';
}
```

后续 Permission Engine 只能接受 `McpPrincipal`，不能直接自己重新解析 header。

---

## 4. ChatGPT OAuth 2.1

必须符合 MCP Authorization 规范。

至少实现：

### Protected Resource Metadata

```text
GET /.well-known/oauth-protected-resource
```

返回资源服务器信息，包括：

- resource
- authorization_servers
- scopes_supported
- resource_documentation

### Authorization Server Metadata

根据实际 issuer 暴露标准 metadata，例如：

```text
/.well-known/oauth-authorization-server
```

至少提供：

- issuer
- authorization_endpoint
- token_endpoint
- code_challenge_methods_supported: ["S256"]
- scopes_supported
- token_endpoint_auth_methods_supported

### Authorization Code + PKCE

必须：

- authorization_code flow
- PKCE
- S256
- state
- exact redirect URI validation
- short-lived authorization code
- authorization code 单次消费
- access token expiry
- refresh token 是否实现由实际需要决定

### resource 参数

OAuth flow 必须正确绑定 MCP resource，防止 token 被用于错误 resource。

---

## 5. ChatGPT Client Identification

优先支持 CIMD。

当协议与 OpenAI 当前要求兼容时，允许 ChatGPT 使用稳定 Client ID Metadata Document：

```text
https://chatgpt.com/oauth/client.json
```

不要写死为唯一合法客户端，除非部署模式明确配置：

```text
CHATGPT_ONLY=true
```

默认保留以后接入其他标准 MCP OAuth client 的可能性。

如采用 DCR，也必须：

- 严格校验 redirect URIs
- 限制 registration
- 防止 DCR 被当成公开垃圾 client 注册接口滥用

对于个人 CF Manager，优先采用 CIMD，减少 DCR 复杂度。

---

## 6. Single-user OAuth 模式

本项目是个人自托管管理后台，因此第一版可以实现安全边界清晰的 single-user OAuth authorization server。

但必须满足协议，不允许用“输入固定 Token”伪装成 OAuth。

建议：

```text
Admin login / explicit consent
            │
            ▼
     OAuth authorize
            │
       PKCE code
            │
            ▼
       token endpoint
            │
            ▼
     short-lived token
```

Access Token：

- 建议短生命周期。
- 使用不可预测 opaque token + D1 hash，或经过严格签名验证的 JWT。
- 不能把 Cloudflare Token 当 OAuth Token。
- 不能直接复用 API_SECRET 作为 access token。

API_SECRET 可以作为后台用户身份验证的一部分，但 OAuth access token 必须独立生成。

---

## 7. OAuth Scopes

至少定义：

```text
cf.read
cf.write
cf.deploy
cf.dns.write
cf.storage.write
cf.delete
cf.dangerous
```

OAuth Scope 是第一层。

Account 的 `ai_permissions` 是第二层。

实际权限：

```text
effective permission
=
OAuth scopes
∩ MCP client policy
∩ account.ai_permissions
∩ endpoint policy
∩ risk/approval policy
```

不能因为 OAuth 有 `cf.write` 就绕过 Account AI Permission。

---

## 8. MCP Tool securitySchemes

ChatGPT 连接使用的 MCP tools 必须正确声明 security schemes。

例如只读工具：

```text
oauth2: cf.read
```

DNS 写：

```text
oauth2: cf.dns.write
```

Deploy：

```text
oauth2: cf.deploy
```

如果某个 health/profile 工具确实可以匿名，才允许 `noauth`。

不要为了连接方便把资源管理工具标成 noauth。

---

## 9. OAuth Challenge

当 Tool 缺少或 scope 不足时：

- 返回标准 OAuth challenge。
- 包含 MCP `mcp/www_authenticate` metadata。
- 让 ChatGPT 能触发账号连接/重新授权 UI。

禁止仅返回：

```json
{"error":"Unauthorized"}
```

而导致 ChatGPT 无法自动发起授权。

---

## 10. Tool annotations

所有 MCP Tool 按真实行为设置 annotation。

例如：

### accounts_list

```text
readOnlyHint = true
destructiveHint = false
```

### resources_search

```text
readOnlyHint = true
destructiveHint = false
```

### cloudflare_api_execute

动态操作本身难以用一个固定 annotation 精确描述，因此：

- 通用 execute 的描述必须明确它可写。
- 对高频写操作提供 typed tools。
- destructive typed tools 标注 destructive。
- Host confirmation 不能替代服务端 Risk/Approval。

### dns_record_upsert

```text
readOnlyHint = false
destructiveHint = false
```

### resource delete

```text
readOnlyHint = false
destructiveHint = true
```

---

## 11. CF Manager Profile Tool

增加一个轻量只读 Tool：

```text
cf_manager_profile
```

用途：

- 返回 CF Manager 实例名称。
- 返回登录主体/实例标识。
- 不返回 Cloudflare credential。
- 不把内部 Cloudflare 多账号误表示成 ChatGPT 的 OAuth 用户账号。

可按 OpenAI MCP profile metadata 规范标记，用于用户连接多个 CF Manager 实例时区分。

示例：

```json
{
  "instance": "xiaochou-cf-manager",
  "user": "owner",
  "cloudflare_accounts": 5
}
```

---

## 12. ChatGPT Plugin 两种接入模式

### Mode A — Direct Personal Plugin

这是本项目 MVP 必须支持的方式。

用户：

1. 部署 CF Manager。
2. 配好 OAuth。
3. 在 ChatGPT 开启 Developer Mode。
4. 添加 MCP Server URL：

```text
https://<domain>/mcp
```

5. 完成 OAuth。
6. ChatGPT 中安装/调用个人插件。

这种模式不需要用户先制作 plugin package。

---

### Mode B — Portable Plugin Package

作为正式交付的一部分提供可生成的插件包。

建议仓库：

```text
plugins/
└── cf-manager/
    ├── plugin.json
    ├── mcp.json
    ├── skills/
    │   └── cloudflare-ops/
    │       └── SKILL.md
    └── assets/
```

但由于每个自托管实例的 URL 不同，不应把某个固定生产 URL 硬编码提交到仓库。

实现：

```text
scripts/generate-chatgpt-plugin.mjs
```

输入：

```bash
node scripts/generate-chatgpt-plugin.mjs \
  --url https://cf.example.com \
  --output dist/cf-manager-plugin
```

生成：

```text
plugin.json
mcp.json
skills/...
```

---

## 13. Portable plugin.json

使用当前标准 Agent Plugins schema。

最低包含：

- name
- version
- description
- author
- repository
- license
- keywords

OpenAI 展示 metadata 放：

```text
extensions.com.openai.interface
```

建议：

```text
displayName: CF Manager
category: Developer Tools
capabilities:
- Read
- Write
```

建议 default prompts：

- 检查所有 Cloudflare 账号的 Workers 状态。
- 搜索所有账号中名为 xxx 的资源。
- 对 dev 账号执行一次 DNS 变更预检。
- 查看过去 24 小时 AI 运维审计。

---

## 14. Portable mcp.json

生成的配置使用标准 Agent Plugins MCP schema。

目标形态：

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "cf-manager": {
      "type": "streamable-http",
      "url": "https://<deployment>/mcp"
    }
  }
}
```

生成脚本负责替换 deployment URL。

---

## 15. Skills

增加：

```text
skills/cloudflare-ops/SKILL.md
```

只写稳定工作流，不写 credential。

建议包含：

### 跨账号资源查找

```text
accounts_list
→ resources_search
→ summarize by account
```

### Worker 迁移

```text
source inspect
→ target inspect
→ preflight
→ present plan
→ execute
→ verify
```

### DNS 修改

```text
read existing
→ calculate diff
→ write
→ verify
```

### 风险规则

- Delete 前先说明影响。
- HIGH/CRITICAL 遵守 Approval。
- 不尝试绕过 CF Manager 的权限错误。
- 不索取 Cloudflare Token 明文。

Skills 不能替代服务器侧安全策略。

---

## 16. ChatGPT UI（可选）

MVP 不要求 UI。

后续可使用 MCP Apps 标准为以下 Tool 返回 UI resource：

- resources_search：资源表格。
- deployment preflight：差异预览。
- audit_search：审计表格。
- approval_status：审批状态。

即使 UI 存在：

> Tool 在没有 UI 的情况下也必须完整可用。

高风险操作的最终服务端审批不能仅依赖 ChatGPT UI。

---

## 17. ChatGPT Host Confirmation 与内部 Approval

两层不要混淆：

```text
ChatGPT confirmation
        +
CF Manager Risk/Approval
```

ChatGPT 可能对 write/destructive Tool 展示确认。

但是：

- HIGH/CRITICAL 的 CF Manager Approval 仍然保留。
- Host confirmation 不是 authorization。
- Agent 不能通过“用户已经在 ChatGPT 点确认”绕过 Web Admin approval。

以后如果确认 ChatGPT 提供可验证、可绑定 request hash 的 approval primitive，可以另行评估整合。

---

## 18. Web Admin 增加 ChatGPT 状态

Settings → MCP 增加：

```text
Generic MCP
- Enabled
- Static bearer configured

ChatGPT Plugin
- OAuth enabled
- Issuer
- Resource URL
- OAuth metadata health
- ChatGPT connection readiness
```

增加：

```text
Test ChatGPT compatibility
```

检查：

- HTTPS
- /mcp
- protected resource metadata
- authorization metadata
- PKCE S256
- required scopes
- OAuth challenge
- tools/list

不能需要用户把 ChatGPT access token 粘贴进后台。

---

## 19. ChatGPT Acceptance

新增 acceptance suite：

### GPT1

Protected resource metadata 合法。

### GPT2

Authorization server metadata 合法并包含 PKCE S256。

### GPT3

Invalid redirect URI 拒绝。

### GPT4

PKCE verifier 错误拒绝。

### GPT5

Authorization code 只能使用一次。

### GPT6

Expired code/token 拒绝。

### GPT7

Missing scope 对 Tool 拒绝并返回可发现 OAuth challenge。

### GPT8

OAuth token 不能访问未授权 CF account permission。

### GPT9

ChatGPT 连接后可以 list tools。

### GPT10

accounts_list 正常。

### GPT11

resources_search 正常。

### GPT12

write Tool 仍经过 Risk/Approval。

### GPT13

Plugin package generator 输出的 plugin.json / mcp.json 通过 schema validation。

### GPT14

Skills 中不存在 secret 或生产 credential。

---

## 20. Codex 工作包调整

原 WP0 → WP12 保留。

调整：

### WP4

除 Remote MCP Skeleton 外，同时建立：

- Auth abstraction
- static bearer principal
- OAuth principal interface

不要把 static bearer 写死进 Tool。

### WP7

Write Operations + Approval 时，Tool 同时声明正确 securitySchemes / annotations。

### WP10

增加 OAuth threat model：

- code interception
- PKCE bypass
- redirect URI injection
- token replay
- resource confusion
- scope escalation
- CSRF/state
- metadata spoofing

### WP12

必须增加：

- ChatGPT Direct Personal Plugin acceptance
- `docs/CHATGPT-PLUGIN.md`
- portable plugin generator
- plugin.json/mcp.json schema validation
- skills
- ChatGPT connection docs

---

## 21. MVP Definition of Done 增补

除原有 MCP DoD 外，必须满足：

- [ ] ChatGPT 可通过标准 Remote MCP 连接。
- [ ] ChatGPT 使用 OAuth 2.1，而非自定义 API Key。
- [ ] OAuth 使用 Authorization Code + PKCE S256。
- [ ] Protected Resource Metadata 可发现。
- [ ] OAuth scopes 与 CF Manager Account AI Permissions 同时生效。
- [ ] Tool securitySchemes 正确。
- [ ] Tool annotations 与真实副作用一致。
- [ ] ChatGPT OAuth 不能绕过 Risk / Approval。
- [ ] 支持 Direct Personal Plugin。
- [ ] 可以生成 portable plugin package。
- [ ] plugin.json / mcp.json schema validation PASS。
- [ ] ChatGPT 接入文档完成。

---

## 22. 最终建议

优先顺序：

```text
Remote MCP protocol correctness
        ↓
Permission / Risk / Audit
        ↓
OAuth 2.1 for ChatGPT
        ↓
Direct Personal Plugin
        ↓
Portable plugin package
        ↓
Optional MCP Apps UI
```

不要为了先看到 ChatGPT 插件图标，而跳过服务器侧 OAuth、权限和审计。
