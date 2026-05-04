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

#### 1.1.2 触发流程

```
dispatchMemoRelatedWebhook(ctx, memo, activityType)
    │
    ├── 1. ResolveUserByName() → 获取 Memo 创建者
    ├── 2. Store.GetUserWebhooks() → 获取该用户配置的所有 webhooks
    │
    └── 3. 遍历 webhooks:
           │
           ├── convertMemoToWebhookPayload(memo)
           │       └── 返回 WebhookRequestPayload{Creator, Memo}
           │
           ├── payload.ActivityType = activityType
           ├── payload.URL = hook.Url
           │
           └── webhook.PostAsync(payload)
```

### 1.2 请求体实际字段（重要！）

#### 1.2.1 数据结构

```go
// internal/webhook/webhook.go
type WebhookRequestPayload struct {
    URL          string       `json:"url"`           // ⚠️ 注意：有 json tag，会被序列化！
    ActivityType string       `json:"activityType"`
    Creator      string       `json:"creator"`
    Memo         *v1pb.Memo   `json:"memo"`
}
```

#### 1.2.2 实际发送逻辑

```go
// Post 函数内部
body, err := json.Marshal(requestPayload)  // ⚠️ 序列化整个结构体！

req, err := http.NewRequest("POST", requestPayload.URL, bytes.NewBuffer(body))
```

**关键点**：`json.Marshal(requestPayload)` 会序列化**所有字段**，包括 `url`！

#### 1.2.3 实际请求体示例

```json
POST <webhook-configured-url>
Content-Type: application/json

{
    "url": "https://example.com/webhook-endpoint",
    "activityType": "memos.memo.created",
    "creator": "users/steven",
    "memo": {
        "name": "memos/abc123",
        "state": "NORMAL",
        "creator": "users/steven",
        "create_time": "2025-05-05T10:00:00Z",
        "update_time": "2025-05-05T10:00:00Z",
        "content": "#work Meeting notes\n- Item 1",
        "visibility": "PRIVATE",
        "tags": ["work"],
        "pinned": false,
        "attachments": [],
        "relations": []
    }
}
```

**注意**：
- `url` 字段**会被发送**，值是用户配置的 webhook URL
- `memo` 字段是完整的 `v1pb.Memo` 对象，包含所有 API 可见字段

### 1.3 成功判定条件（重要！）

#### 1.3.1 判定逻辑

```go
// internal/webhook/webhook.go:96-121
resp, err := safeClient.Do(req)
// ...

// 条件 1: HTTP 状态码必须在 200-299 之间
if resp.StatusCode < 200 || resp.StatusCode > 299 {
    return errors.Errorf("failed to post webhook %s, status code: %d", requestPayload.URL, resp.StatusCode)
}

// 条件 2: 响应体必须能反序列化为 {code, message} 结构
response := &struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}{}
if err := json.Unmarshal(b, response); err != nil {
    // ⚠️ 注意：如果响应体不是有效的 JSON，或者没有 code 字段，这里会报错！
    return errors.Wrapf(err, "failed to unmarshal webhook response from %s", requestPayload.URL)
}

// 条件 3: code 必须等于 0
if response.Code != 0 {
    return errors.Errorf("receive error code sent by webhook server, code %d, msg: %s", response.Code, response.Message)
}
```

#### 1.3.2 外部服务必须返回的格式

**✅ 正确的响应（成功）：**
```json
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code": 0,
    "message": "success"
}
```

**❌ 错误的响应（会失败）：**

```json
// 失败 1: 空响应体
// → json.Unmarshal 失败

// 失败 2: 纯文本
HTTP/1.1 200 OK
Content-Type: text/plain

OK
// → json.Unmarshal 失败

// 失败 3: 没有 code 字段
HTTP/1.1 200 OK
Content-Type: application/json

{
    "status": "ok"
}
// → 虽然能 Unmarshal，但 response.Code 是默认值 0
// ⚠️ 注意：这种情况实际上会"成功"，因为 Go 结构体默认值是 0

// 失败 4: code 非 0
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code": 1,
    "message": "something went wrong"
}
// → 失败，因为 code != 0
```

#### 1.3.3 特殊情况说明

如果外部服务返回的 JSON 不包含 `code` 字段：
```json
{
    "message": "hello"
}
```

`json.Unmarshal` 会成功，但 `response.Code` 会是 Go `int` 类型的默认值 `0`，所以会被判定为**成功**。

这是一个潜在的"漏洞"，但也是 Go JSON 反序列化的标准行为。

### 1.4 异步发送实现

#### 1.4.1 异步队列机制

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
                    slog.Warn("Failed to dispatch webhook asynchronously",
                        slog.String("url", payload.URL),
                        slog.String("activityType", payload.ActivityType),
                        slog.Any("err", err))
                }
            }
        }()
    }
}
```

#### 1.4.2 PostAsync 投递逻辑

```go
func PostAsync(requestPayload *WebhookRequestPayload) {
    if requestPayload == nil {
        slog.Warn("Dropped webhook dispatch because payload is nil")
        return
    }
    select {
    case asyncPostQueue <- requestPayload:
        // 成功入队
    default:
        // 队列满，丢弃并记录警告
        slog.Warn("Dropped webhook dispatch because the async queue is full",
            slog.String("url", requestPayload.URL),
            slog.String("activityType", requestPayload.ActivityType))
    }
}
```

### 1.5 安全机制

#### 1.5.1 SSRF 防护

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

// 连接时验证 IP（在 Dial 阶段）
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

#### 1.5.2 可配置的安全开关

```go
// 允许私有 IP（用于本地部署场景）
var AllowPrivateIPs bool
```

#### 1.5.3 URL 验证（创建 webhook 时）

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

---

## 二、向内输入与事件触发的共用路径（重要！）

### 2.1 核心发现：REST 和 MCP 共用同一路径

#### 2.1.1 架构说明

```
┌─────────────────────────────────────────────────────────────────┐
│                        外部输入入口                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐           ┌──────────────────┐           │
│  │   REST API       │           │    MCP 协议      │           │
│  │  (Connect RPC)   │           │  (AI 助手专用)   │           │
│  └────────┬─────────┘           └────────┬─────────┘           │
│           │                               │                       │
│           │ 1. HTTP 请求                  │ 1. HTTP + SSE        │
│           │ 2. Connect RPC 反序列化       │ 2. MCP 消息解析      │
│           │                               │                       │
│           └───────────────┬───────────────┘                       │
│                           │                                          │
│                           ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              APIV1Service 核心业务逻辑                    │     │
│  │                    (共用路径)                             │     │
│  │                                                           │     │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │     │
│  │  │CreateMemo() │ │UpdateMemo() │ │DeleteMemo() │        │     │
│  │  │SetAttachment│ │SetRelation  │ │CreateComment│        │     │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘        │     │
│  │         │               │               │                 │     │
│  │         ▼               ▼               ▼                 │     │
│  │  ┌──────────────────────────────────────────────────┐    │     │
│  │  │              副作用触发层                         │    │     │
│  │  │                                                   │    │     │
│  │  │  1. DispatchMemo*Webhook()  → 向外发送 webhook  │    │     │
│  │  │  2. SSEHub.Broadcast()       → 实时推送通知      │    │     │
│  │  │  3. dispatchMentionNotifications → 提及通知      │    │     │
│  │  └──────────────────────────────────────────────────┘    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 MCP 工具如何调用共用路径

```go
// server/router/mcp/tools_memo.go

func (s *MCPService) handleCreateMemo(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // ... 参数解析 ...

    // ⚠️ 关键：直接调用 APIV1Service 的方法！
    created, err := s.apiV1Service.CreateMemo(ctx, &v1pb.CreateMemoRequest{
        Memo: &v1pb.Memo{
            Content:    content,
            Visibility: visibilityToProto(visibility),
        },
    })
    // ...
}

func (s *MCPService) handleUpdateMemo(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // ...
    // ⚠️ 同样调用 APIV1Service.UpdateMemo()
    updated, err := s.apiV1Service.UpdateMemo(ctx, &v1pb.UpdateMemoRequest{
        Memo:       update,
        UpdateMask: updateMask,
    })
    // ...
}

func (s *MCPService) handleDeleteMemo(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // ...
    // ⚠️ 同样调用 APIV1Service.DeleteMemo()
    if _, err := s.apiV1Service.DeleteMemo(ctx, &v1pb.DeleteMemoRequest{Name: "memos/" + uid}); err != nil {
        // ...
    }
    // ...
}

func (s *MCPService) handleCreateMemoComment(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // ...
    // ⚠️ 同样调用 APIV1Service.CreateMemoComment()
    comment, err := s.apiV1Service.CreateMemoComment(ctx, &v1pb.CreateMemoCommentRequest{
        Name: "memos/" + uid,
        Comment: &v1pb.Memo{
            Content:    content,
            Visibility: visibilityToProto(parent.Visibility),
        },
    })
    // ...
}
```

#### 2.1.3 共用路径的代码证据

**CreateMemo 的 webhook 触发**（来自 `memo_service.go`）：

```go
func (s *APIV1Service) CreateMemo(ctx context.Context, request *v1pb.CreateMemoRequest) (*v1pb.Memo, error) {
    // ... 业务逻辑 ...
    
    // 无论从 REST 还是 MCP 调用，都会执行到这里
    if err := s.DispatchMemoCreatedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo created webhook", slog.Any("err", err))
    }
    
    // 以及 SSE 广播
    if !isSSESuppressed(ctx) {
        s.SSEHub.Broadcast(&SSEEvent{
            Type:       SSEEventMemoCreated,
            Name:       memoMessage.Name,
            Visibility: memo.Visibility,
            CreatorID:  resolveSSECreatorID(memo, nil),
        })
    }
    
    return memoMessage, nil
}
```

**UpdateMemo 的 webhook 触发**：

```go
func (s *APIV1Service) UpdateMemo(ctx context.Context, request *v1pb.UpdateMemoRequest) (*v1pb.Memo, error) {
    // ... 业务逻辑 ...
    
    // 无论从 REST 还是 MCP 调用，都会执行到这里
    s.dispatchMemoUpdatedSideEffects(ctx, memo, parentMemo, memoMessage)
    
    return memoMessage, nil
}

// dispatchMemoUpdatedSideEffects 实现
func (s *APIV1Service) dispatchMemoUpdatedSideEffects(ctx context.Context, memo *store.Memo, parentMemo *store.Memo, memoMessage *v1pb.Memo) {
    // 触发 webhook
    if err := s.DispatchMemoUpdatedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo updated webhook", slog.Any("err", err))
    }
    
    // 以及 SSE 广播
    s.SSEHub.Broadcast(&SSEEvent{
        Type:       SSEEventMemoUpdated,
        Name:       memoMessage.Name,
        Parent:     memoMessage.GetParent(),
        Visibility: memo.Visibility,
        CreatorID:  resolveSSECreatorID(memo, parentMemo),
    })
}
```

### 2.2 输入方式汇总

#### 2.2.1 三种输入方式

| 方式 | 协议 | 端点 | 认证 | 适用场景 |
|------|------|------|------|----------|
| **REST API** | HTTP + Connect RPC | `/api/v1/memos` 等 | JWT 或 PAT | 通用自动化、脚本 |
| **MCP 协议** | HTTP + SSE | `/mcp` | JWT 或 PAT | AI 助手（Claude Desktop 等） |
| **PAT 直接调用** | 任何 HTTP 客户端 | 任意 API 端点 | PAT 头 | 长期运行的服务 |

#### 2.2.2 PAT（Personal Access Token）

**用途**：
- 长期有效的访问令牌
- 无需用户交互的自动化场景
- MCP 服务器认证
- REST API 认证

**格式**：
- Token 值：`memos_pat_<random-string>`
- HTTP 头：`Authorization: Bearer <token>`

**特性**：
- SHA-256 哈希存储（明文仅在创建时显示一次）
- 可选过期时间
- 用户可添加描述用于标识

### 2.3 触发覆盖范围

无论通过哪种方式输入，以下操作都会触发 webhook：

| 操作 | ActivityType | 触发条件 |
|------|--------------|----------|
| 创建 Memo | `memos.memo.created` | ✅ 总是触发 |
| 更新 Memo 内容 | `memos.memo.updated` | ✅ 总是触发 |
| 更新 Memo 可见性 | `memos.memo.updated` | ✅ 总是触发 |
| 更新附件 | `memos.memo.updated` | ✅ 总是触发 |
| 更新关系 | `memos.memo.updated` | ✅ 总是触发 |
| 置顶/取消置顶 | `memos.memo.updated` | ✅ 总是触发 |
| 归档/取消归档 | `memos.memo.updated` | ✅ 总是触发 |
| 删除 Memo | `memos.memo.deleted` | ✅ 总是触发 |
| 创建评论 | `memos.memo.comment.created` | ✅ 发送给**原帖作者**（不是评论者） |
| 添加/删除反应 | - | ❌ 不触发 |
| 添加/删除标签 | - | ❌ 不触发（标签是 payload 的一部分，更新内容时才会触发） |

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
│  POST <webhook-configured-url>                                  │
│  Content-Type: application/json                                 │
│                                                                  │
│  ⚠️ 请求体（包含 url 字段！）：                                  │
│  {                                                               │
│    "url": "https://example.com/webhook-endpoint",              │
│    "activityType": "memos.memo.created",                        │
│    "creator": "users/steven",                                   │
│    "memo": {                                                     │
│      "name": "memos/abc123",                                    │
│      "content": "...",                                           │
│      ...                                                         │
│    }                                                             │
│  }                                                               │
│                                                                  │
│  ⚠️ 外部服务必须返回（否则视为失败）：                           │
│  HTTP/1.1 200 OK                                                 │
│  Content-Type: application/json                                 │
│                                                                  │
│  {                                                               │
│    "code": 0,                                                    │
│    "message": "success"                                          │
│  }                                                               │
│                                                                  │
│  注意：如果返回的 JSON 没有 code 字段，Go 会使用默认值 0，     │
│       这种情况会被视为"成功"。                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 向内输入与共用触发路径

```
┌─────────────────────────────────────────────────────────────────┐
│                      外部自动化/AI 助手                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  方式 1: REST API (Connect RPC)                          │   │
│  │                                                           │   │
│  │  POST   /api/v1/memos              → CreateMemo()       │   │
│  │  PATCH  /api/v1/memos/{name}       → UpdateMemo()       │   │
│  │  DELETE /api/v1/memos/{name}       → DeleteMemo()       │   │
│  │                                                           │   │
│  │  Authorization: Bearer <token>  (JWT 或 PAT)           │   │
│  └───────────────────────────┬──────────────────────────────┘   │
│                              │                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  方式 2: MCP 协议 (AI 助手专用)                          │   │
│  │                                                           │   │
│  │  POST /mcp                                                │   │
│  │  SSE  /mcp                                                │   │
│  │                                                           │   │
│  │  工具调用：                                                │   │
│  │  - create_memo    → 调用 s.apiV1Service.CreateMemo()   │   │
│  │  - update_memo    → 调用 s.apiV1Service.UpdateMemo()   │   │
│  │  - delete_memo    → 调用 s.apiV1Service.DeleteMemo()   │   │
│  │  - create_memo_comment → 调用 CreateMemoComment()       │   │
│  │                                                           │   │
│  │  Authorization: Bearer <token>  (JWT 或 PAT)           │   │
│  └───────────────────────────┬──────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    APIV1Service (共用业务层)                     │
│              ⚠️  所有触发逻辑都在这里发生 ⚠️                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CreateMemo()                                             │   │
│  │  ├── 验证权限                                             │   │
│  │  ├── 写入数据库                                           │   │
│  │  ├── 设置附件/关系                                        │   │
│  │  ├── DispatchMemoCreatedWebhook()  ← 触发 webhook       │   │
│  │  └── SSEHub.Broadcast()              ← 实时推送          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  UpdateMemo()                                             │   │
│  │  ├── 验证权限                                             │   │
│  │  ├── 更新数据库                                           │   │
│  │  └── dispatchMemoUpdatedSideEffects()                    │   │
│  │      ├── DispatchMemoUpdatedWebhook()  ← 触发 webhook   │   │
│  │      └── SSEHub.Broadcast()          ← 实时推送          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DeleteMemo()                                             │   │
│  │  ├── 验证权限                                             │   │
│  │  ├── DispatchMemoDeletedWebhook()  ← 触发 webhook       │   │
│  │  ├── 删除数据库                                           │   │
│  │  └── SSEHub.Broadcast()              ← 实时推送          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、关键代码位置索引

### 4.1 向外事件发送

| 功能 | 文件路径 | 关键函数/行号 |
|------|----------|---------------|
| Webhook POST 发送 | `internal/webhook/webhook.go:84` | `Post()` |
| 请求体序列化 | `internal/webhook/webhook.go:85` | `json.Marshal(requestPayload)` |
| 响应判定逻辑 | `internal/webhook/webhook.go:107-121` | 状态码 + Unmarshal + code == 0 |
| 异步投递 | `internal/webhook/webhook.go:128` | `PostAsync()` |
| URL/IP 安全验证 | `internal/webhook/validate.go` | `ValidateURL()`, `isReservedIP()` |
| Memo 创建触发 | `server/router/api/v1/memo_service.go:136` | `DispatchMemoCreatedWebhook()` |
| Memo 更新触发 | `server/router/api/v1/memo_update_helpers.go:66` | `dispatchMemoUpdatedSideEffects()` |
| Memo 删除触发 | `server/router/api/v1/memo_service.go:587` | `DispatchMemoDeletedWebhook()` |
| 评论创建触发 | `server/router/api/v1/memo_service.go:699` | `DispatchMemoCommentCreatedWebhook()` |
| 附件更新触发 | `server/router/api/v1/memo_attachment_service.go:48` | `dispatchMemoUpdatedSideEffects()` |
| 关系更新触发 | `server/router/api/v1/memo_relation_service.go:48` | `dispatchMemoUpdatedSideEffects()` |

### 4.2 向内输入与共用路径

| 功能 | 文件路径 | 关键函数/行号 |
|------|----------|---------------|
| MCP 服务器核心 | `server/router/mcp/mcp.go` | `RegisterRoutes()`, `NewMCPService()` |
| MCP 创建 Memo | `server/router/mcp/tools_memo.go:366` | `s.apiV1Service.CreateMemo()` |
| MCP 更新 Memo | `server/router/mcp/tools_memo.go:426` | `s.apiV1Service.UpdateMemo()` |
| MCP 删除 Memo | `server/router/mcp/tools_memo.go:451` | `s.apiV1Service.DeleteMemo()` |
| MCP 创建评论 | `server/router/mcp/tools_memo.go:591` | `s.apiV1Service.CreateMemoComment()` |
| APIV1Service.CreateMemo | `server/router/api/v1/memo_service.go:41` | 共用业务入口 |
| APIV1Service.UpdateMemo | `server/router/api/v1/memo_service.go:436` | 共用业务入口 |
| APIV1Service.DeleteMemo | `server/router/api/v1/memo_service.go:543` | 共用业务入口 |
| PAT 生成与验证 | `server/auth/` | `GeneratePersonalAccessToken()` |
| Webhook 配置管理 | `server/router/api/v1/user_service.go:990` | `ListUserWebhooks()` 等 |

### 4.3 协议定义

| 功能 | 文件路径 |
|------|----------|
| Webhook 请求体结构 | `internal/webhook/webhook.go:72` (WebhookRequestPayload) |
| Webhook 配置数据结构 | `proto/api/v1/user_service.proto:672` (UserWebhook) |
| Memo 数据结构 | `proto/api/v1/memo_service.proto:187` (Memo) |
| Memo API 定义 | `proto/api/v1/memo_service.proto` |

---

## 五、注意事项与限制

### 5.1 向外事件（Webhook）

#### 5.1.1 请求体相关

1. **`url` 字段会被发送**：`WebhookRequestPayload.URL` 有 `json:"url"` tag，会被包含在请求体中
2. **完整 Memo 对象**：`memo` 字段是完整的 `v1pb.Memo`，包含所有 API 可见字段

#### 5.1.2 响应判定相关（重要！）

1. **严格的响应格式要求**：
   - HTTP 状态码必须 2xx
   - 响应体必须是有效的 JSON
   - 响应体必须能反序列化为 `{code, message}` 结构
   - `code` 必须等于 0

2. **特殊情况**：
   - 如果 JSON 不包含 `code` 字段，`response.Code` 会是 Go `int` 的默认值 `0`，会被判定为**成功**
   - 这是 Go JSON 反序列化的标准行为

3. **常见失败场景**：
   - 外部服务返回空响应体 → `json.Unmarshal` 失败
   - 外部服务返回纯文本（如 `"OK"`）→ `json.Unmarshal` 失败
   - 外部服务返回 `{"code": 1, "message": "error"}` → `code != 0` 失败

#### 5.1.3 可靠性相关

1. **无重试机制**：webhook 发送失败后仅记录日志，不会自动重试
2. **队列可能溢出**：异步队列缓冲 128 个请求，高并发下可能丢弃
3. **无批量发送**：每个 webhook 独立发送，不支持批量
4. **无签名验证**：请求体不包含签名，外部服务无法验证请求来源的真实性

#### 5.1.4 安全相关

1. **SSRF 保护**：默认禁止向私有/保留 IP 发送
2. **本地部署配置**：自托管场景需设置 `AllowPrivateIPs = true`

### 5.2 向内输入

1. **共用路径**：REST API 和 MCP 工具最终都调用 `APIV1Service` 的相同方法，触发相同的 webhook
2. **PAT 安全**：PAT 仅在创建时显示一次，需妥善保存；建议设置过期时间
3. **PAT 权限**：PAT 拥有用户的完整权限，无细粒度权限控制
4. **MCP 只读模式**：建议 AI 助手使用只读端点（`/mcp/readonly`）避免意外修改

### 5.3 触发覆盖范围

| 操作 | 是否触发 webhook | 说明 |
|------|-------------------|------|
| 创建 Memo | ✅ | 通过 REST 或 MCP |
| 更新 Memo（任何字段） | ✅ | 通过 REST 或 MCP |
| 删除 Memo | ✅ | 通过 REST 或 MCP |
| 创建评论 | ✅ | **发送给原帖作者**，不是评论者 |
| 添加/删除反应 | ❌ | 无 webhook 触发 |
| 标签变化 | ❌ | 标签是 content 的一部分，只有更新 content 时才会触发 |

---

## 六、扩展建议

### 6.1 Webhook 增强

1. **移除冗余的 url 字段**：请求体中发送 `url` 字段是冗余的（外部服务知道自己的 URL），建议移除或至少使其可选

2. **请求签名**：使用 HMAC 签名请求体，让外部服务验证来源：
   ```go
   // 建议添加的功能
   signature := hmac_sha256(secret, body)
   req.Header.Set("X-Memos-Signature", "sha256=" + signature)
   ```

3. **响应判定宽松化**：
   - 允许空响应体（2xx 状态码即视为成功）
   - 允许纯文本响应
   - 或者至少提供配置选项让用户选择严格/宽松模式

4. **重试机制**：使用指数退避策略重试失败的 webhook

5. **Webhook 日志**：记录 webhook 发送历史和状态，方便调试

6. **事件过滤**：允许用户配置仅接收特定类型的事件

7. **自定义 Header**：允许用户配置自定义 HTTP 头（如 API Key）

### 6.2 外部服务实现指南

如果要实现一个接收 Memos webhook 的外部服务，必须返回：

```go
// 推荐的处理方式
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 读取请求体
    body, _ := io.ReadAll(r.Body)
    
    // 2. 处理业务逻辑
    // ...
    
    // 3. ⚠️ 必须返回这样的响应！
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "code":    0,
        "message": "success",
    })
}
```

### 6.3 同步场景建议

对于需要双向同步的场景（如与其他笔记工具同步）：

1. **向外同步**：
   - 使用 webhook 监听变化
   - 结合 `activityType` 和 `memo.update_time` 去重
   - 注意 `code` 必须返回 0，否则 webhook 会被视为失败（但不会重试）

2. **向内同步**：
   - 使用 REST API + PAT
   - 或使用 MCP 协议（如果是 AI 驱动的同步）

3. **循环同步防护**：
   - 在 memo 内容或 payload 中添加同步标记（如 `[synced:tool-name]`）
   - 外部服务接收 webhook 时检查标记，避免循环同步

4. **可靠性考虑**：
   - webhook 无重试机制，建议外部服务实现自己的可靠性保证
   - 或考虑轮询 API 作为补充（检查 `update_time`）
