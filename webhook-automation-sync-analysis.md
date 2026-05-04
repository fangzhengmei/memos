# Webhook 自动化同步机制分析

本文档分析 Memos 系统中 Memo 创建、更新或资源变化后的事件发送机制，以及外部输入如何进入系统。

---

## 一、向外事件发送（系统 → 外部自动化）

### 1.1 核心发送机制

#### 1.1.1 触发点

当 Memo 发生变化时，系统通过以下关键函数触发 webhook：

| 操作 | 触发函数 | ActivityType | 代码位置 |
|------|----------|--------------|----------|
| **创建 Memo** | `DispatchMemoCreatedWebhook()` | `memos.memo.created` | `memo_service.go:136` |
| **更新 Memo** | `DispatchMemoUpdatedWebhook()` | `memos.memo.updated` | `memo_update_helpers.go:67` |
| **删除 Memo** | `DispatchMemoDeletedWebhook()` | `memos.memo.deleted` | `memo_service.go:587` |
| **创建评论** | `DispatchMemoCommentCreatedWebhook()` | `memos.memo.comment.created` | `memo_service.go:699` |
| **更新附件** | `dispatchMemoUpdatedSideEffects()` | `memos.memo.updated` | `memo_attachment_service.go:48` |
| **更新关系** | `dispatchMemoUpdatedSideEffects()` | `memos.memo.updated` | `memo_relation_service.go:48` |

#### 1.1.2 触发流程示例（创建 Memo）

```
CreateMemo()
    │
    ├── 1. 验证用户权限
    ├── 2. 构建 Memo 数据
    ├── 3. 解析 Markdown payload
    ├── 4. 写入数据库
    ├── 5. 设置附件/关系
    ├── 6. 转换为 API 响应格式
    │
    ├── 7. DispatchMemoCreatedWebhook() ← 触发 webhook
    │   │
    │   └── dispatchMemoRelatedWebhook()
    │       ├── 获取 Memo 创建者
    │       ├── 获取该用户的所有 webhooks
    │       ├── 遍历 webhooks，构建 payload
    │       └── webhook.PostAsync() ← 异步发送
    │
    ├── 8. SSEHub.Broadcast() ← 实时推送通知
    └── 9. 返回结果
```

### 1.2 异步发送实现

#### 1.2.1 异步队列机制

```go
// internal/webhook/webhook.go

// 异步队列，缓冲 128 个请求
asyncPostQueue = make(chan *WebhookRequestPayload, 128)

func init() {
    // 启动 4 个 worker goroutine 处理队列
    for range 4 {
        go func() {
            for payload := range asyncPostQueue {
                if err := Post(payload); err != nil {
                    slog.Warn("Failed to dispatch webhook asynchronously", ...)
                }
            }
        }()
    }
}
```

#### 1.2.2 投递失败处理

- **队列满时丢弃**：当队列满时，新的 webhook 请求会被丢弃并记录警告日志
- **单条失败不影响其他**：每个 webhook 独立处理，单个失败不会影响同批次的其他 webhook
- **仅记录日志**：失败后仅记录日志，没有重试机制

### 1.3 安全机制

#### 1.3.1 SSRF 防护

```go
// internal/webhook/validate.go

// 禁止访问的 IP 段
var reservedCIDRs = []string{
    "127.0.0.0/8",    // IPv4 回环
    "10.0.0.0/8",     // RFC-1918 A 类
    "172.16.0.0/12",  // RFC-1918 B 类
    "192.168.0.0/16", // RFC-1918 C 类
    "169.254.0.0/16", // 链路本地 / 云 IMDS
    "::1/128",        // IPv6 回环
    "fc00::/7",       // IPv6 唯一本地
    "fe80::/10",      // IPv6 链路本地
}

// 连接时验证 IP
func safeDialContext(ctx context.Context, network, addr string) (net.Conn, error) {
    // 解析 hostname
    ips, err := net.DefaultResolver.LookupHost(ctx, host)
    // 检查每个 IP 是否在保留范围内
    for _, ipStr := range ips {
        if ip := net.ParseIP(ipStr); ip != nil && isReservedIP(ip) {
            return nil, errors.Errorf("webhook: connection to reserved/private IP address is not allowed")
        }
    }
    // ...
}
```

#### 1.3.2 可配置的安全开关

```go
// 允许私有 IP（用于本地部署场景）
var AllowPrivateIPs bool
```

#### 1.3.3 URL 验证

```go
func ValidateURL(rawURL string) error {
    // 1. 必须是有效的绝对 URL
    u, err := url.ParseRequestURI(rawURL)
    
    // 2. 必须使用 http 或 https
    if u.Scheme != "http" && u.Scheme != "https" {
        return status.Errorf(codes.InvalidArgument, ...)
    }
    
    // 3. 主机名必须可解析
    ips, err := net.LookupHost(u.Hostname())
    
    // 4. 解析后的 IP 不能是保留 IP
    for _, ipStr := range ips {
        if ip := net.ParseIP(ipStr); ip != nil && isReservedIP(ip) {
            return status.Errorf(codes.InvalidArgument, ...)
        }
    }
    return nil
}
```

### 1.4 请求/响应格式

#### 1.4.1 Webhook Payload 结构

```go
// internal/webhook/webhook.go
type WebhookRequestPayload struct {
    URL          string       `json:"url"`           // 目标 URL（不在请求体中发送）
    ActivityType string       `json:"activityType"`  // 事件类型
    Creator      string       `json:"creator"`       // 创建者资源名，如 "users/steven"
    Memo         *v1pb.Memo   `json:"memo"`          // 完整的 Memo 对象
}
```

#### 1.4.2 发送格式

- **HTTP 方法**：POST
- **Content-Type**：application/json
- **超时**：30 秒
- **请求体**：`WebhookRequestPayload` 的 JSON 序列化（不包含 `url` 字段）

#### 1.4.3 响应验证

外部服务返回的响应必须满足：

1. **HTTP 状态码**：2xx
2. **响应体格式**：
```json
{
    "code": 0,
    "message": "success"
}
```
3. **code 必须为 0**，否则视为失败

### 1.5 Webhook 配置管理

#### 1.5.1 配置存储

Webhook 配置存储在 **用户设置** 中，通过 `UserSetting` 实体管理：

- **Key**：`WEBHOOKS`（`storepb.UserSetting_WEBHOOKS`）
- **存储位置**：用户设置表

#### 1.5.2 API 接口（来自 `user_service.proto`）

| RPC 方法 | HTTP 方法 | 路径 | 功能 |
|----------|-----------|------|------|
| `ListUserWebhooks` | GET | `/api/v1/{parent=users/*}/webhooks` | 列出用户的所有 webhook |
| `CreateUserWebhook` | POST | `/api/v1/{parent=users/*}/webhooks` | 创建新 webhook |
| `UpdateUserWebhook` | PATCH | `/api/v1/{webhook.name=users/*/webhooks/*}` | 更新 webhook |
| `DeleteUserWebhook` | DELETE | `/api/v1/{name=users/*/webhooks/*}` | 删除 webhook |

#### 1.5.3 数据结构

```protobuf
// proto/api/v1/user_service.proto
message UserWebhook {
    string name = 1;           // 资源名，格式: users/{user}/webhooks/{webhook}
    string url = 2;            // 目标 URL
    string display_name = 3;   // 可选：显示名称
    google.protobuf.Timestamp create_time = 4;
    google.protobuf.Timestamp update_time = 5;
}
```

---

## 二、外部输入机制（外部自动化 → 系统）

### 2.1 主要输入通道

Memos 系统提供三种主要的外部自动化输入方式：

1. **REST API（Connect RPC）**：标准 HTTP API
2. **MCP 协议**：AI 助手专用协议
3. **PAT 认证**：长期访问令牌

### 2.2 REST API（Connect RPC）

#### 2.2.1 核心 Memo 操作 API

| 操作 | RPC 方法 | HTTP | 路径 |
|------|----------|------|------|
| **创建** | `CreateMemo` | POST | `/api/v1/memos` |
| **更新** | `UpdateMemo` | PATCH | `/api/v1/memos/{memo.name}` |
| **删除** | `DeleteMemo` | DELETE | `/api/v1/memos/{name}` |
| **查询单条** | `GetMemo` | GET | `/api/v1/memos/{name}` |
| **查询列表** | `ListMemos` | GET | `/api/v1/memos` |
| **创建评论** | `CreateMemoComment` | POST | `/api/v1/memos/{name}:createComment` |

#### 2.2.2 API 请求示例

**创建 Memo：**
```bash
POST /api/v1/memos
Content-Type: application/json
Authorization: Bearer <token>

{
    "memo": {
        "content": "#work Meeting notes\n- Item 1\n- Item 2",
        "visibility": "PRIVATE"
    }
}
```

**更新 Memo：**
```bash
PATCH /api/v1/memos/abc123
Content-Type: application/json
Authorization: Bearer <token>

{
    "memo": {
        "name": "memos/abc123",
        "content": "Updated content"
    },
    "update_mask": {
        "paths": ["content"]
    }
}
```

### 2.3 MCP 协议（AI 助手集成）

#### 2.3.1 MCP 服务器概述

Memos 内置了完整的 **MCP（Model Context Protocol）** 服务器，允许 AI 助手（如 Claude Desktop）直接操作 Memos 数据。

- **端点**：`/mcp`
- **认证**：通过 `Authorization` 头（PAT 或 JWT）
- **协议**：HTTP + SSE（可流式）

#### 2.3.2 MCP 路由配置

```go
// server/router/mcp/mcp.go
func (s *MCPService) RegisterRoutes(echoServer *echo.Echo) {
    // ...
    mcpGroup.Any("/mcp", echo.WrapHandler(httpHandler))
    mcpGroup.Any("/mcp/readonly", echo.WrapHandler(httpHandler))
    mcpGroup.Any("/mcp/x/:toolsets", echo.WrapHandler(httpHandler))
    mcpGroup.Any("/mcp/x/:toolsets/readonly", echo.WrapHandler(httpHandler))
}
```

#### 2.3.3 MCP 工具列表

**Memo 操作工具**（`tools_memo.go`）：

| 工具名 | 功能 | 是否修改数据 |
|--------|------|--------------|
| `list_memos` | 列出可见的 memo | 否 |
| `get_memo` | 获取单个 memo | 否 |
| `create_memo` | 创建新 memo | **是** |
| `update_memo` | 更新 memo | **是** |
| `delete_memo` | 删除 memo | **是** |
| `search_memos` | 搜索 memo 内容 | 否 |
| `list_memo_comments` | 列出评论 | 否 |
| `create_memo_comment` | 创建评论 | **是** |

**其他工具**：
- `tools_tag.go`：标签操作
- `tools_attachment.go`：附件操作
- `tools_relation.go`：关系操作
- `tools_reaction.go`：反应操作
- `resources_memo.go`：Memo 资源
- `prompts.go`：内置提示词

#### 2.3.4 MCP 认证流程

```go
// server/router/mcp/mcp.go
mcpGroup.Use(func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c *echo.Context) error {
        // 从 Authorization 头获取 token
        authHeader := c.Request().Header.Get("Authorization")
        if authHeader != "" {
            // 使用 Authenticator 验证 token（支持 PAT 和 JWT）
            result := s.authenticator.Authenticate(c.Request().Context(), authHeader)
            if result == nil {
                return c.JSON(http.StatusUnauthorized, ...)
            }
            // 将用户信息注入 context
            ctx := auth.ApplyToContext(c.Request().Context(), result)
            c.SetRequest(c.Request().WithContext(ctx))
        }
        return next(c)
    }
})
```

#### 2.3.5 MCP 安全特性

1. **只读模式**：
   - 路径 `/mcp/readonly` 或 `X-MCP-Readonly: true` 头
   - 禁用所有修改类工具（`create_memo`, `update_memo`, `delete_memo` 等）

2. **工具集过滤**：
   - `X-MCP-Toolsets` 头：指定可用工具集
   - `X-MCP-Tools` 头：指定具体可用工具
   - `X-MCP-Exclude-Tools` 头：排除特定工具

3. **工具集定义**：
   - `memo`：Memo 读写
   - `tag`：标签操作
   - `attachment`：附件操作
   - `relation`：关系操作
   - `reaction`：反应操作

### 2.4 附件与资源变化

#### 2.4.1 附件更新触发

当 Memo 的附件发生变化时，会触发 `memos.memo.updated` webhook：

```go
// server/router/api/v1/memo_attachment_service.go:48
func (s *APIV1Service) SetMemoAttachments(...) {
    // ... 更新附件逻辑 ...
    
    // 触发更新副作用（包括 webhook）
    s.dispatchMemoUpdatedSideEffects(ctx, updatedMemo, parentMemo, memoMessage)
}
```

#### 2.4.2 关系更新触发

```go
// server/router/api/v1/memo_relation_service.go:48
func (s *APIV1Service) SetMemoRelations(...) {
    // ... 更新关系逻辑 ...
    
    // 触发更新副作用
    s.dispatchMemoUpdatedSideEffects(ctx, updatedMemo, parentMemo, memoMessage)
}
```

### 2.5 PAT（Personal Access Token）认证

#### 2.5.1 PAT 用途

PAT 是长期有效的访问令牌，用于：
- 脚本/自动化工具访问 API
- MCP 服务器认证
- 移动应用认证
- CLI 工具认证

#### 2.5.2 PAT 特性

- **格式**：`memos_pat_` 前缀的随机字符串
- **存储**：SHA-256 哈希存储在数据库
- **过期**：可选过期时间（或永不过期）
- **描述**：用户可添加描述用于标识

#### 2.5.3 PAT 管理 API

| 操作 | HTTP | 路径 |
|------|------|------|
| 列出 PAT | GET | `/api/v1/{parent=users/*}/personalAccessTokens` |
| 创建 PAT | POST | `/api/v1/{parent=users/*}/personalAccessTokens` |
| 删除 PAT | DELETE | `/api/v1/{name=users/*/personalAccessTokens/*}` |

#### 2.5.4 创建 PAT 响应

```json
{
    "personal_access_token": {
        "name": "users/steven/personalAccessTokens/abc123",
        "description": "Automation script",
        "expires_at": "2026-01-01T00:00:00Z",
        "created_at": "2025-05-05T00:00:00Z"
    },
    "token": "memos_pat_xxxxxxxxxxxx"  // 仅在创建时返回一次！
}
```

---

## 三、完整数据流图

### 3.1 向外事件流

```
┌─────────────────────────────────────────────────────────────────┐
│                        Memos 系统内部                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │ CreateMemo   │    │ UpdateMemo   │    │ DeleteMemo   │     │
│  │ SetAttachment│    │ SetRelation  │    │ CreateComment│     │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘     │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              DispatchMemo*Webhook()                      │    │
│  │  - dispatchMemoRelatedWebhook()                         │    │
│  │  - 获取用户的 webhooks 列表                              │    │
│  │  - 构建 WebhookRequestPayload                            │    │
│  └───────────────────────┬─────────────────────────────────┘    │
│                          │                                         │
│                          ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              webhook.PostAsync()                         │    │
│  │  - 写入 asyncPostQueue (缓冲 128)                       │    │
│  │  - 队列满时丢弃 + 警告日志                               │    │
│  └───────────────────────┬─────────────────────────────────┘    │
│                          │                                         │
│                          ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              4 个 Worker Goroutine                       │    │
│  │  - 从队列读取 payload                                    │    │
│  │  - 调用 Post() 发送 HTTP POST                            │    │
│  │  - 失败时记录警告日志                                    │    │
│  └───────────────────────┬─────────────────────────────────┘    │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      外部自动化服务                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  POST <webhook-url>                                             │
│  Content-Type: application/json                                 │
│                                                                  │
│  请求体:                                                         │
│  {                                                               │
│    "activityType": "memos.memo.created",                        │
│    "creator": "users/steven",                                   │
│    "memo": {                                                     │
│      "name": "memos/abc123",                                    │
│      "content": "...",                                           │
│      ...                                                         │
│    }                                                             │
│  }                                                               │
│                                                                  │
│  期望响应 (200 OK):                                             │
│  {"code": 0, "message": "success"}                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 向内输入流

```
┌─────────────────────────────────────────────────────────────────┐
│                      外部自动化/AI 助手                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐           ┌──────────────────┐           │
│  │   REST API       │           │    MCP 协议      │           │
│  │  (Connect RPC)   │           │  (AI 助手专用)   │           │
│  └────────┬─────────┘           └────────┬─────────┘           │
│           │                               │                       │
│           │ POST /api/v1/memos            │ POST /mcp            │
│           │ PATCH /api/v1/memos/*         │ SSE 流式             │
│           │ DELETE /api/v1/memos/*        │                      │
│           │                               │                       │
│           │ Authorization: Bearer <token> │ Authorization: Bearer│
│           │                               │                       │
│           └───────────────┬───────────────┘                       │
│                           │                                          │
│                           ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              认证层 (Authenticator)                       │     │
│  │  - 支持 JWT (短期会话令牌)                                │     │
│  │  - 支持 PAT (长期访问令牌)                                │     │
│  │  - 从 Authorization 头提取                               │     │
│  └───────────────────────┬──────────────────────────────────┘     │
└──────────────────────────┼───────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Memos 系统内部                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   API/MCP 处理层                          │   │
│  │                                                           │   │
│  │  REST API:                                                │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │   │
│  │  │CreateMemo   │ │UpdateMemo   │ │DeleteMemo   │        │   │
│  │  │SetAttachment│ │SetRelation  │ │CreateComment│        │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘        │   │
│  │                                                           │   │
│  │  MCP 工具:                                                │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │   │
│  │  │create_memo  │ │update_memo  │ │delete_memo  │        │   │
│  │  │create_comment││...          │ │...          │        │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘        │   │
│  │                                                           │   │
│  │  注意: MCP 工具内部调用的是与 REST API 相同的服务方法     │   │
│  │  (handleCreateMemo 调用 s.apiV1Service.CreateMemo)      │   │
│  └───────────────────────┬───────────────────────────────────┘   │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   副作用触发层                             │   │
│  │                                                           │   │
│  │  1. DispatchMemo*Webhook()  → 向外发送 webhook          │   │
│  │  2. SSEHub.Broadcast()       → 实时推送通知              │   │
│  │  3. dispatchMentionNotifications → 提及通知              │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、关键代码位置索引

### 4.1 向外事件发送

| 功能 | 文件路径 | 关键函数/行号 |
|------|----------|---------------|
| Webhook 核心发送 | `internal/webhook/webhook.go` | `Post()`, `PostAsync()` |
| URL/IP 安全验证 | `internal/webhook/validate.go` | `ValidateURL()`, `isReservedIP()` |
| Memo 创建触发 | `server/router/api/v1/memo_service.go:136` | `DispatchMemoCreatedWebhook()` |
| Memo 更新触发 | `server/router/api/v1/memo_update_helpers.go:66` | `dispatchMemoUpdatedSideEffects()` |
| Memo 删除触发 | `server/router/api/v1/memo_service.go:587` | `DispatchMemoDeletedWebhook()` |
| 评论创建触发 | `server/router/api/v1/memo_service.go:699` | `DispatchMemoCommentCreatedWebhook()` |
| 附件更新触发 | `server/router/api/v1/memo_attachment_service.go:48` | `dispatchMemoUpdatedSideEffects()` |
| 关系更新触发 | `server/router/api/v1/memo_relation_service.go:48` | `dispatchMemoUpdatedSideEffects()` |

### 4.2 向内输入处理

| 功能 | 文件路径 | 关键函数/行号 |
|------|----------|---------------|
| MCP 服务器核心 | `server/router/mcp/mcp.go` | `RegisterRoutes()`, `NewMCPService()` |
| MCP Memo 工具 | `server/router/mcp/tools_memo.go` | `handleCreateMemo()`, `handleUpdateMemo()` |
| Memo REST API | `server/router/api/v1/memo_service.go` | `CreateMemo()`, `UpdateMemo()`, `DeleteMemo()` |
| PAT 生成与验证 | `server/auth/` | `GeneratePersonalAccessToken()`, `HashPersonalAccessToken()` |
| PAT 管理 API | `server/router/api/v1/user_service.go:907` | `ListPersonalAccessTokens()`, `CreatePersonalAccessToken()` |
| Webhook 配置管理 | `server/router/api/v1/user_service.go:990` | `ListUserWebhooks()`, `CreateUserWebhook()` |

### 4.3 协议定义

| 功能 | 文件路径 |
|------|----------|
| Webhook 配置数据结构 | `proto/api/v1/user_service.proto:672` (UserWebhook) |
| Memo API 定义 | `proto/api/v1/memo_service.proto` |
| PAT API 定义 | `proto/api/v1/user_service.proto:612` (PersonalAccessToken) |

---

## 五、注意事项与限制

### 5.1 向外事件（Webhook）

1. **无重试机制**：webhook 发送失败后仅记录日志，不会自动重试
2. **队列可能溢出**：异步队列缓冲 128 个请求，高并发下可能丢弃
3. **无批量发送**：每个 webhook 独立发送，不支持批量
4. **无签名验证**：请求体不包含签名，外部服务无法验证请求来源的真实性
5. **SSRF 保护**：默认禁止向私有/保留 IP 发送，本地部署需设置 `AllowPrivateIPs = true`

### 5.2 向内输入

1. **MCP 认证**：MCP 端点支持未认证访问（仅限公共数据），修改操作需要认证
2. **PAT 安全**：PAT 仅在创建时显示一次，需妥善保存；建议设置过期时间
3. **PAT 权限**：PAT 拥有用户的完整权限，无细粒度权限控制
4. **MCP 只读模式**：建议 AI 助手使用只读端点（`/mcp/readonly`）避免意外修改

### 5.3 触发覆盖范围

| 操作 | 是否触发 webhook | ActivityType |
|------|-------------------|--------------|
| 创建 Memo | ✅ | `memos.memo.created` |
| 更新 Memo 内容 | ✅ | `memos.memo.updated` |
| 更新 Memo 可见性 | ✅ | `memos.memo.updated` |
| 更新附件 | ✅ | `memos.memo.updated` |
| 更新关系 | ✅ | `memos.memo.updated` |
| 置顶/取消置顶 | ✅ | `memos.memo.updated` |
| 归档/取消归档 | ✅ | `memos.memo.updated` |
| 删除 Memo | ✅ | `memos.memo.deleted` |
| 创建评论 | ✅ (发送给原帖作者) | `memos.memo.comment.created` |
| 添加/删除反应 | ❌ | - |
| 添加/删除标签 | ❌ (标签在 payload 中解析，不单独触发) | - |

---

## 六、扩展建议

### 6.1 Webhook 增强

1. **添加请求签名**：使用 HMAC 签名请求体，让外部服务验证来源
2. **实现重试机制**：使用指数退避策略重试失败的 webhook
3. **添加 Webhook 日志**：记录 webhook 发送历史和状态
4. **支持事件过滤**：允许用户配置仅接收特定类型的事件
5. **支持自定义 Header**：允许用户配置自定义 HTTP 头（如 API Key）

### 6.2 输入能力增强

1. **Webhook 接收端点**：提供专门的 inbound webhook 端点，允许外部服务通过 webhook 推送数据
2. **PAT 细粒度权限**：为 PAT 添加权限范围（如只读、仅创建等）
3. **API 速率限制**：为 API 添加速率限制防止滥用
4. **MCP 工具增强**：添加更多 MCP 工具，如批量操作、导入导出等

### 6.3 同步模式建议

对于需要双向同步的场景（如与其他笔记工具同步），建议：

1. **向外**：使用 webhook 监听变化，结合 `activityType` 和 `memo.update_time` 去重
2. **向内**：使用 MCP 协议或 REST API + PAT 进行写入
3. **同步状态**：在 memo 内容或 payload 中添加同步标记（如 `[synced:tool-name]`）避免循环同步
