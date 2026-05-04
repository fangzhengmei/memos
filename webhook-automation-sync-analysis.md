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

---

## 二、关键时序问题（重要！）

### 2.1 CreateMemo 携带附件/关系时的触发顺序

#### 2.1.1 问题发现

**在创建 Memo 时，如果请求中包含 `attachments` 或 `relations`，会触发多个 webhook，且顺序是 `updated` 先于 `created`！**

#### 2.1.2 代码证据

```go
// server/router/api/v1/memo_service.go:86-138

func (s *APIV1Service) CreateMemo(ctx context.Context, request *v1pb.CreateMemoRequest) (*v1pb.Memo, error) {
    // 1. 首先创建 Memo
    memo, err := s.Store.CreateMemo(ctx, create)  // 第86行
    
    // 2. 如果有附件，设置附件
    if len(request.Memo.Attachments) > 0 {  // 第100行
        // ⚠️ 注意：SetMemoAttachments 内部会触发 updated webhook！
        _, err := s.SetMemoAttachments(ctx, &v1pb.SetMemoAttachmentsRequest{
            Name:        fmt.Sprintf("%s%s", MemoNamePrefix, memo.UID),
            Attachments: request.Memo.Attachments,
        })
        // ...
    }
    
    // 3. 如果有关系，设置关系
    if len(request.Memo.Relations) > 0 {  // 第117行
        // ⚠️ 注意：SetMemoRelations 内部会触发 updated webhook！
        _, err := s.SetMemoRelations(ctx, &v1pb.SetMemoRelationsRequest{
            Name:      fmt.Sprintf("%s%s", MemoNamePrefix, memo.UID),
            Relations: request.Memo.Relations,
        })
        // ...
    }
    
    // 4. 最后才触发 created webhook！
    memoMessage, err := s.convertMemoFromStore(ctx, memo, nil, attachments, relations)
    // ...
    // 第136行：⚠️ 这是最后一步！
    if err := s.DispatchMemoCreatedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo created webhook", slog.Any("err", err))
    }
    // ...
}
```

#### 2.1.3 为什么 `SetMemoAttachments` 会触发 `updated`？

```go
// server/router/api/v1/memo_attachment_service.go:48

func (s *APIV1Service) SetMemoAttachments(ctx context.Context, request *v1pb.SetMemoAttachmentsRequest) (*emptypb.Empty, error) {
    // ... 更新附件逻辑 ...
    
    // ⚠️ 第48行：这个函数会触发 updated webhook！
    s.dispatchMemoUpdatedSideEffects(ctx, updatedMemo, parentMemo, memoMessage)
    
    return &emptypb.Empty{}, nil
}

// 同样的，SetMemoRelations 也会触发：
// server/router/api/v1/memo_relation_service.go:48
// s.dispatchMemoUpdatedSideEffects(ctx, updatedMemo, parentMemo, memoMessage)
```

#### 2.1.4 完整时序图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CreateMemo 完整执行顺序（带附件和关系）                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CreateMemo() 入口                                                        │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Store.CreateMemo()  → 数据库创建 Memo                       │   │
│  │    memo.create_time = T1                                          │   │
│  │    memo.update_time = T1 (初始值)                              │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                            │                                          │
│                            ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. 检查是否有 attachments?                                              │   │
│  │    ┌──────────────────────────────────────────────────────┐      │
│  │    │ 有 attachments                                        │      │
│  │    │  ▼                                                   │      │
│  │    │ SetMemoAttachments()                                   │      │
│  │    │   ├── touchMemoUpdatedTimestamp()                    │      │
│  │    │   │     → memo.update_time = T2 (更新时间戳)           │      │
│  │    │   │                                                    │      │
│  │    │   └── dispatchMemoUpdatedSideEffects()              │      │
│  │    │       └── DispatchMemoUpdatedWebhook()                   │      │
│  │    │           └── ⚠️ 发送第一个 webhook！                    │      │
│  │    │              activityType: "memos.memo.updated"         │      │
│  │    │              memo.update_time: T2                       │      │
│  │    └──────────────────────────────────────────────────────┘      │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                            │                                          │
│                            ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. 检查是否有 relations?                                              │   │
│  │    ┌──────────────────────────────────────────────────────┐      │
│  │    │ 有 relations                                          │      │
│  │    │  ▼                                                   │      │
│  │    │ SetMemoRelations()                                     │      │
│  │    │   ├── touchMemoUpdatedTimestamp()                    │      │
│  │    │   │     → memo.update_time = T3 (再次更新时间戳)      │      │
│  │    │   │                                                    │      │
│  │    │   └── dispatchMemoUpdatedSideEffects()              │      │
│  │    │       └── DispatchMemoUpdatedWebhook()                   │      │
│  │    │           └── ⚠️ 发送第二个 webhook！                │      │
│  │    │              activityType: "memos.memo.updated"         │      │
│  │    │              memo.update_time: T3                       │      │
│  │    └──────────────────────────────────────────────────────┘      │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                            │                                          │
│                            ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 4. 最后才触发 created webhook！                               │   │
│  │    ┌──────────────────────────────────────────────────────┐      │
│  │    │ convertMemoFromStore()                                  │      │
│  │    │    → memo.update_time 是 T3 (最新的更新时间)           │      │
│  │    │                                                       │      │
│  │    │ DispatchMemoCreatedWebhook()                         │      │
│  │    │   └── ⚠️ 发送第三个 webhook！                         │      │
│  │    │         activityType: "memos.memo.created"          │      │
│  │    │         memo.update_time: T3                         │      │
│  │    │         memo.create_time: T1 (创建时间)               │      │
│  │    └──────────────────────────────────────────────────────┘      │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.5 不同场景的触发数量

| 创建时携带的内容 | 触发的 webhook 数量 | 事件顺序 |
|-----------------|---------------------|----------|
| 无附件、无关系 | 1 个 | 1. `created` |
| 仅附件 | 2 个 | 1. `updated` → 2. `created` |
| 仅关系 | 2 个 | 1. `updated` → 2. `created` |
| 附件 + 关系 | 3 个 | 1. `updated`(附件) → 2. `updated`(关系) → 3. `created` |

#### 2.1.6 时间戳对比

假设当前时间为 `T1`，`touchMemoUpdatedTimestamp` 间隔为 `T2`、`T3`：

| 事件 | memo.create_time | memo.update_time | 说明 |
|------|-----------------|------------------|------|
| 数据库创建 | T1 | T1 | 初始值相同 |
| 第1次 updated (附件) | T1 | T2 | update_time 被更新 |
| 第2次 updated (关系) | T1 | T3 | update_time 再次更新 |
| 最后 created | T1 | T3 | 使用最新的 update_time |

**关键观察**：
- 所有事件的 `create_time` 都是相同的（数据库创建时间）
- `update_time` 会逐渐递增
- 最后的 `created` 事件的 `update_time` 是最新的

---

### 2.2 外部消费方的判重策略

#### 2.2.1 问题分析

外部消费方如果不了解这个时序，可能会遇到以下问题：

**问题场景 1**：
```
消费方逻辑：
- 收到 created → 在本地创建记录
- 收到 updated → 在本地更新记录

实际收到的顺序：
1. updated (附件) → 本地找不到记录 → 报错/丢弃
2. updated (关系) → 本地找不到记录 → 报错/丢弃
3. created → 本地创建记录 → 但此时 memo 已经有附件和关系？
   → 问题：created 的 memo 已经包含了附件和关系（因为是最新状态）
```

**问题场景 2**：
```
消费方逻辑：
- 所有事件都写入消息队列，按时间戳处理

实际问题：
- 先收到的 updated 的 update_time 较旧
- 后收到的 created 的 update_time 较新
- 如果按 update_time 排序，created 会排在后面（正确）
- 但 activityType 是 created，不是 updated，这会造成混淆
```

#### 2.2.2 推荐的判重策略

**策略 A：基于 `create_time` + `update_time` 组合

```go
// 伪代码
func processWebhook(payload WebhookRequestPayload) {
    memoID := payload.Memo.Name  // "memos/abc123"
    
    if payload.ActivityType == "memos.memo.created" {
        // ⚠️ created 事件表示这是一个新创建的 memo
        // 但它的 update_time 可能比之前的 updated 事件新
        // 应该用这个事件来创建或覆盖本地记录
        
        // 因为：
        // 1. created 总是在所有 updated 之后触发
        // 2. created 的 memo 包含了所有附件和关系（完整状态）
        // 3. created 的 update_time 是最新的
        
        upsertMemo(payload.Memo, "created")
    } else if payload.ActivityType == "memos.memo.updated" {
        // ⚠️ updated 事件可能发生在 created 之前！
        // 这是一个"预更新"，表示 memo 正在被设置附件或关系
        
        // 策略 1：如果本地没有这个 memo，先暂存，等 created 事件
        if !localMemoExists(memoID) {
            // 这可能是一个刚创建的 memo，正在设置附件
            // 建议：暂存这个更新，等待 created 事件
            storePendingUpdate(memoID, payload)
            return
        }
        
        // 策略 2：如果本地已经有了，正常更新
        // 但需要比较 update_time，确保不会用旧数据覆盖新数据
        currentUpdateTime := parseTime(payload.Memo.UpdateTime)
        localUpdateTime := getLocalMemoUpdateTime(memoID)
        
        if currentUpdateTime.After(localUpdateTime) {
            updateMemo(payload.Memo, "updated")
        }
    }
}
```

**策略 B：优先处理流程图**

```
┌─────────────────────────────────────────────────────────────────┐
│                    收到 webhook 事件                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              判断 activityType                                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
┌─────────────────────┐      ┌─────────────────────┐
│  activityType ==     │      │  activityType ==│
│  "memos.memo.created"  │      │  "memos.memo.updated" │
└──────────┬──────────────┘      └──────────┬──────────┘
           │                                  │
           ▼                                  ▼
┌─────────────────────────────┐      ┌─────────────────────────────┐
│  这是完整的最终状态        │      │  检查本地是否已存在此 memo  │
│  - 包含所有附件            │      └──────────┬──────────────────┘
│  - 包含所有关系            │                 │
│  - update_time 是最新的    │    ┌────────────┴────────────┐
│                           │    │                         │
│  操作：                    │    ▼                         ▼
│  - 本地不存在则创建          │┌───────────────┐       ┌───────────────┐
│  - 本地存在则覆盖（因为是最终态）││  本地已存在  │       │  本地不存在   │
│  - 清除 pending updates        ││             │       │             │
│  - 这是权威事件！            ││ 比较        │       │ 这可能是：    │
└─────────────────────────────┘│ update_time │       │ - 刚创建的   │
                              │             │       │   正在设附件  │
                              │ 新数据更晚？ │       │ - 真的更新   │
                              └──────┬──────┘       └──────┬────────┘
                                     │                     │
                              ┌──────┴──────┐              │
                              │             │              │
                              ▼             ▼              ▼
                        ┌──────────┐ ┌──────────┐ ┌─────────────────┐
                        │   是     │ │   否     │ │ 暂存到 pending    │
                        │          │ │          │ │ 等待 created 事件│
                        ▼          ▼          │ 或稍后再检查        │
                   ┌──────────┐ ┌──────────┐    └─────────────────┘
                   │ 更新本地  │ │ 忽略    │
                   │        │ │        │
                   └──────────┘ └──────────┘
```

#### 2.2.3 数据结构建议

**本地存储建议包含的字段：

```go
type LocalMemo struct {
    // 基础字段
    MemoID      string    // "memos/abc123"
    Content     string
    UpdateTime  time.Time // 用于判断新旧
    
    // ⚠️ 关键同步字段
    CreateTime  time.Time // 数据库创建时间
    IsSynced    bool      // 是否已通过 created 事件确认同步
    LastActivity string    // 最后处理的 activityType
    
    // 用于处理时序问题
    PendingUpdates []PendingUpdate // 暂存的 updated 事件
}

type PendingUpdate struct {
    ActivityType string
    UpdateTime time.Time
    MemoData   *v1pb.Memo
    ReceivedAt time.Time
}
```

#### 2.2.4 处理流程示例

**场景：创建带附件和关系的 Memo**

```
时间线：

T1: 数据库创建 memo
    create_time = T1, update_time = T1

T2: 设置附件
    → 发送 webhook 1:
      activityType: "memos.memo.updated"
      memo.create_time: T1
      memo.update_time: T2

    外部消费方处理：
    - 本地没有这个 memo
    - 暂存到 PendingUpdates
    - 标记：等待 created 事件

T3: 设置关系
    → 发送 webhook 2:
      activityType: "memos.memo.updated"
      memo.create_time: T1
      memo.update_time: T3

    外部消费方处理：
    - 本地没有这个 memo
    - 暂存到 PendingUpdates（替换旧的，因为 T3 > T2）

T4: 触发 created webhook
    → 发送 webhook 3:
      activityType: "memos.memo.created"
      memo.create_time: T1
      memo.update_time: T4 (注意：可能不是 T3，取决于时间精度）

    外部消费方处理：
    - 本地没有这个 memo
    - 使用这个 memo 创建本地记录
    - IsSynced = true
    - 清空 PendingUpdates（因为 created 是权威）
    - 这是最终状态！
```

#### 2.2.5 简化策略（推荐）

对于大多数场景，可以使用更简单的策略：

**策略 C：以 `created` 为权威，忽略创建前后的 `updated`

```go
// 简化策略：
// 1. 对于任何 memo，只要没有收到 created 之前，
//    所有 updated 都暂存或忽略
// 2. 收到 created 后，使用该 memo 的数据
// 3. 之后的 updated 正常处理

// 为什么这样做的理由：
// - created 总是在所有 created 时的 updated 之后触发
// - created 的 memo 已经包含了所有附件和关系
// - created 之后的 updated 才是真正的"更新"操作

func processWebhookSimplified(payload WebhookRequestPayload) {
    memoID := payload.Memo.Name
    
    if payload.ActivityType == "memos.memo.created" {
        // ⚠️ 这是权威事件
        // 用这个 memo 包含创建时的所有附件和关系
        // 这是这个 memo 的"初始完整状态"
        createOrUpdateMemo(payload.Memo)
        markAsCreated(memoID)
    } else if payload.ActivityType == "memos.memo.updated" {
        // 检查是否已通过 created 确认
        if isMarkedAsCreated(memoID) {
            // ⚠️ 这是真正的更新操作
            // 不是创建时的附件/关系设置
            // 可以安全更新
            updateMemoIfNewer(payload.Memo)
        } else {
            // ⚠️ 这是创建时的附件/关系设置
            // 忽略它，等待 created 事件
            // 因为 created 会包含完整状态
            log("Ignoring pre-created update for", memoID)
        }
    }
}
```

**策略 C 的优点：
- 实现简单
- 不会出现时序问题
- created 是权威，不会有数据不一致

**策略 C 的潜在问题**：
- 如果创建时的 created 事件丢失了，后续的 updated 也会被忽略
- 建议：有轮询机制作为补充（可选）

---

### 2.3 实际测试验证

#### 2.3.1 测试用例

**测试场景 1：创建带附件的 Memo**

```go
// 请求：
POST /api/v1/memos
{
    "memo": {
        "content": "Test with attachment",
        "attachments": [{"name": "attachments/abc123"}]
    }
}

// 预期收到的 webhook 顺序：
1. activityType: "memos.memo.updated"  (附件设置)
2. activityType: "memos.memo.created"  (最后触发)

// 注意：两个事件的 memo.create_time 相同
//       memo.update_time 不同（updated 的更早）
```

**测试场景 2：创建带关系的 Memo**

```go
// 请求：
POST /api/v1/memos
{
    "memo": {
        "content": "Test with relation",
        "relations": [{
            "relatedMemo": {"name": "memos/def456"},
            "type": "REFERENCE"
        }]
    }
}

// 预期收到的 webhook 顺序：
1. activityType: "memos.memo.updated"  (关系设置)
2. activityType: "memos.memo.created"  (最后触发)
```

**测试场景 3：创建同时带附件和关系的 Memo**

```go
// 请求：
POST /api/v1/memos
{
    "memo": {
        "content": "Test with both",
        "attachments": [{"name": "attachments/abc123"}],
        "relations": [{
            "relatedMemo": {"name": "memos/def456"},
            "type": "REFERENCE"
        }]
    }
}

// 预期收到的 webhook 顺序：
1. activityType: "memos.memo.updated"  (附件设置)
2. activityType: "memos.memo.updated"  (关系设置)
3. activityType: "memos.memo.created"  (最后触发)
```

#### 2.3.2 相关测试代码

```go
// server/router/api/v1/sse_service_test.go:191
func TestSetMemoAttachments_EmitsMemoUpdatedSSEEvent(t *testing.T) {
    // 测试 SetMemoAttachments 会触发 SSE 事件
    // 同样的逻辑也适用于 webhook
}

// server/router/api/v1/sse_service_test.go:234
func TestSetMemoRelations_EmitsMemoUpdatedSSEEvent(t *testing.T) {
    // 测试 SetMemoRelations 会触发 SSE 事件
}
```

---

## 三、请求体与响应规则

### 3.1 请求体实际字段

#### 3.1.1 数据结构

```go
// internal/webhook/webhook.go
type WebhookRequestPayload struct {
    URL          string       `json:"url"`           // ⚠️ 有 json tag，会被序列化！
    ActivityType string       `json:"activityType"`
    Creator      string       `json:"creator"`
    Memo         *v1pb.Memo   `json:"memo"`
}
```

#### 3.1.2 实际发送逻辑

```go
// Post 函数内部
body, err := json.Marshal(requestPayload)  // ⚠️ 序列化整个结构体！

req, err := http.NewRequest("POST", requestPayload.URL, bytes.NewBuffer(body))
```

**关键点**：`json.Marshal(requestPayload)` 会序列化**所有字段**，包括 `url`！

#### 3.1.3 实际请求体示例

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
        "update_time": "2025-05-05T10:00:02Z",
        "content": "#work Meeting notes\n- Item 1",
        "visibility": "PRIVATE",
        "tags": ["work"],
        "pinned": false,
        "attachments": [
            {"name": "attachments/def456", "filename": "image.png", ...}
        ],
        "relations": [...]
    }
}
```

**注意**：
- `url` 字段**会被发送**，值是用户配置的 webhook URL
- `memo` 字段是完整的 `v1pb.Memo` 对象，包含所有 API 可见字段

### 3.2 成功判定条件

#### 3.2.1 三层判定逻辑

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

#### 3.2.2 外部服务必须返回的格式

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

#### 3.2.3 特殊情况说明

如果外部服务返回的 JSON 不包含 `code` 字段：
```json
{
    "message": "hello"
}
```

`json.Unmarshal` 会成功，但 `response.Code` 会是 Go `int` 类型的默认值 `0`，所以会被判定为**成功**。

这是一个潜在的"漏洞"，但也是 Go JSON 反序列化的标准行为。

---

## 四、异步发送与安全机制

### 4.1 异步发送实现

#### 4.1.1 异步队列机制

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

#### 4.1.2 PostAsync 投递逻辑

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

### 4.2 安全机制

#### 4.2.1 SSRF 防护

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
```

#### 4.2.2 可配置的安全开关

```go
// 允许私有 IP（用于本地部署场景）
var AllowPrivateIPs bool
```

---

## 五、向内输入与共用触发路径

### 5.1 共用路径确认

#### 5.1.1 MCP 工具直接调用 APIV1Service

```go
// server/router/mcp/tools_memo.go

func (s *MCPService) handleCreateMemo(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // ... 参数解析 ...

    // ⚠️ 关键：直接调用 APIV1Service 的方法！
    created, err := s.apiV1Service.CreateMemo(ctx, &v1pb.CreateMemoRequest{
        Memo: &v1pb.Memo{
            Content:    content,
            Visibility: visibilityToProto(visibility),
            // ⚠️ 注意：如果 MCP 调用时传入 attachments 或 relations
            // 同样会触发 created 之前的 updated webhook！
            Attachments: attachments,
            Relations:   relations,
        },
    })
    // ...
}
```

#### 5.1.2 共用路径架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      外部输入入口                              │
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
│  │  ┌────────────────────────────────────────────────────┐  │     │
│  │  │ CreateMemo()                                        │  │     │
│  │  │   ├── Store.CreateMemo()                           │  │     │
│  │  │   ├── 检查 attachments? → SetMemoAttachments()│  │     │
│  │  │   │       └── dispatchMemoUpdatedSideEffects()     │  │     │
│  │  │   │           └── ⚠️ 触发 updated webhook!            │  │     │
│  │  │   ├── 检查 relations? → SetMemoRelations()        │  │     │
│  │  │   │       └── dispatchMemoUpdatedSideEffects()     │  │     │
│  │  │   │           └── ⚠️ 触发 updated webhook!            │  │     │
│  │  │   └── DispatchMemoCreatedWebhook() ← ⚠️ 最后触发!    │  │     │
│  │  └────────────────────────────────────────────────────┘  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**关键结论**：
- 无论通过 REST API 还是 MCP 工具，最终都调用相同的 `APIV1Service.CreateMemo()`
- 如果请求中包含 `attachments` 或 `relations`，都会触发 **created 之前的 updated** webhook
- 这是统一的行为，与输入通道无关

### 5.2 输入方式汇总

#### 5.2.1 三种输入方式

| 方式 | 协议 | 端点 | 认证 | 适用场景 |
|------|------|------|------|----------|
| **REST API** | HTTP + Connect RPC | `/api/v1/memos` 等 | JWT 或 PAT | 通用自动化、脚本 |
| **MCP 协议** | HTTP + SSE | `/mcp` | JWT 或 PAT | AI 助手（Claude Desktop 等） |
| **PAT 直接调用** | 任何 HTTP 客户端 | 任意 API 端点 | PAT 头 | 长期运行的服务 |

### 5.3 触发覆盖范围

无论通过哪种方式输入，以下操作都会触发 webhook：

| 操作 | ActivityType | 触发条件 |
|------|--------------|----------|
| 创建 Memo（无附件无关系） | `memos.memo.created` | ✅ 1 个事件 |
| 创建 Memo（带附件） | `memos.memo.updated` + `memos.memo.created` | ✅ 2 个事件，顺序：updated → created |
| 创建 Memo（带关系） | `memos.memo.updated` + `memos.memo.created` | ✅ 2 个事件，顺序：updated → created |
| 创建 Memo（带附件+关系） | `memos.memo.updated` ×2 + `memos.memo.created` | ✅ 3 个事件，顺序：updated(附件) → updated(关系) → created |
| 更新 Memo 内容 | `memos.memo.updated` | ✅ 总是触发 |
| 更新附件 | `memos.memo.updated` | ✅ 总是触发 |
| 更新关系 | `memos.memo.updated` | ✅ 总是触发 |
| 删除 Memo | `memos.memo.deleted` | ✅ 总是触发 |
| 创建评论 | `memos.memo.comment.created` | ✅ 发送给**原帖作者**（不是评论者） |
| 添加/删除反应 | - | ❌ 不触发 |

---

## 六、关键代码位置索引

### 6.1 时序相关代码

| 功能 | 文件路径 | 关键函数/行号 |
|------|----------|---------------|
| CreateMemo 主流程 | `server/router/api/v1/memo_service.go:86-138` | 完整时序 |
| SetMemoAttachments 触发 updated | `server/router/api/v1/memo_attachment_service.go:48` | `dispatchMemoUpdatedSideEffects()` |
| SetMemoRelations 触发 updated | `server/router/api/v1/memo_relation_service.go:48` | `dispatchMemoUpdatedSideEffects()` |
| dispatchMemoUpdatedSideEffects | `server/router/api/v1/memo_update_helpers.go:66` | 包含 webhook + SSE |
| MCP 创建 Memo 调用 | `server/router/mcp/tools_memo.go:366` | `s.apiV1Service.CreateMemo()` |

### 6.2 向外事件发送

| 功能 | 文件路径 | 关键函数/行号 |
|------|----------|---------------|
| Webhook POST 发送 | `internal/webhook/webhook.go:84` | `Post()` |
| 请求体序列化 | `internal/webhook/webhook.go:85` | `json.Marshal(requestPayload)` |
| 响应判定逻辑 | `internal/webhook/webhook.go:107-121` | 状态码 + Unmarshal + code == 0 |
| 异步投递 | `internal/webhook/webhook.go:128` | `PostAsync()` |

### 6.3 协议定义

| 功能 | 文件路径 |
|------|----------|
| Webhook 请求体结构 | `internal/webhook/webhook.go:72` (WebhookRequestPayload) |
| Webhook 配置数据结构 | `proto/api/v1/user_service.proto:672` (UserWebhook) |
| Memo 数据结构 | `proto/api/v1/memo_service.proto:187` (Memo) |

---

## 七、注意事项与限制

### 7.1 时序相关（最重要！）

1. **created 可能不是第一个事件**：
   - 创建带附件或关系的 memo 时，会先收到 `updated`，然后才收到 `created`
   - 这是因为 `SetMemoAttachments` 和 `SetMemoRelations` 内部会触发 `dispatchMemoUpdatedSideEffects`
   - 而 `DispatchMemoCreatedWebhook` 是在所有这些操作**之后**才调用

2. **事件数量**：
   - 无附件无关系：1 个 `created`
   - 仅附件：2 个（`updated` → `created`）
   - 仅关系：2 个（`updated` → `created`）
   - 附件+关系：3 个（`updated`×2` → `created`）

3. **时间戳**：
   - 所有事件的 `create_time` 相同（数据库创建时间）
   - `update_time` 会逐渐递增
   - 最后的 `created` 事件的 `update_time` 是最新的

4. **外部消费方必须处理这个时序**：
   - 推荐策略：以 `created` 为权威，忽略创建前后的 `updated`
   - 或者：暂存 `updated`，等待 `created` 事件
   - 不要假设 `created` 总是第一个事件

### 7.2 请求体与响应

1. **`url` 字段会被发送**：请求体中包含用户配置的 webhook URL
2. **响应必须返回 `{code: 0}`**：否则会被视为失败
3. **无重试机制**：失败后仅记录日志
4. **队列可能溢出**：高并发下可能丢弃

### 7.3 安全

1. **SSRF 保护**：默认禁止向私有/保留 IP 发送
2. **本地部署**：自托管场景需设置 `AllowPrivateIPs = true`

---

## 八、扩展建议

### 8.1 时序问题的解决方案

#### 8.1.1 代码层面优化（建议修改）

**问题根源**：`SetMemoAttachments` 和 `SetMemoRelations` 不应该在创建时触发 `updated` webhook。

**建议修改**：

```go
// 方案 1：在 CreateMemo 中直接调用内部方法，跳过副作用

// 修改 CreateMemo：
func (s *APIV1Service) CreateMemo(ctx context.Context, request *v1pb.CreateMemoRequest) (*v1pb.Memo, error) {
    // ...
    if len(request.Memo.Attachments) > 0 {
        // ⚠️ 直接调用内部方法，不触发 webhook
        if err := s.setMemoAttachmentsInternal(ctx, user, memo, request.Memo.Attachments); err != nil {
            return nil, errors.Wrap(err, "failed to set memo attachments")
        }
        // 只更新时间戳，不触发 webhook
        if err := s.touchMemoUpdatedTimestamp(ctx, memo.ID); err != nil {
            return nil, err
        }
    }
    // 同理处理 relations
    // ...
    
    // 最后只触发一个 created webhook
    if err := s.DispatchMemoCreatedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo created webhook", slog.Any("err", err))
    }
    // ...
}
```

**方案 2：添加上下文标记，让 `dispatchMemoUpdatedSideEffects` 知道是否在创建中

```go
// 在 context 中添加标记
type createMemoKey struct{}

func isCreatingMemo(ctx context.Context) bool {
    return ctx.Value(createMemoKey{}) != nil
}

func withCreatingMemo(ctx context.Context) context.Context {
    return context.WithValue(ctx, createMemoKey{}, true)
}

// 修改 CreateMemo：
func (s *APIV1Service) CreateMemo(ctx context.Context, request *v1pb.CreateMemoRequest) (*v1pb.Memo, error) {
    // 添加标记
    ctx = withCreatingMemo(ctx)
    // ...
}

// 修改 dispatchMemoUpdatedSideEffects：
func (s *APIV1Service) dispatchMemoUpdatedSideEffects(ctx context.Context, ...) {
    // 如果正在创建中，不触发 webhook
    if isCreatingMemo(ctx) {
        return
    }
    // ... 正常触发
}
```

#### 8.1.2 外部消费方的实现建议

**推荐实现（当前代码行为）：

```go
// 伪代码实现

type WebhookProcessor struct {
    // 存储已通过 created 确认的 memo
    confirmedMemos map[string]bool // memoID -> bool
    
    // 存储 pending 的 updates（等待 created）
    pendingUpdates map[string][]*WebhookRequestPayload // memoID -> updates
    
    // 持久化存储
    storage Storage
}

func (p *WebhookProcessor) Process(payload WebhookRequestPayload) error {
    memoID := payload.Memo.Name
    
    if payload.ActivityType == "memos.memo.created" {
        return p.processCreated(payload, memoID)
    } else if payload.ActivityType == "memos.memo.updated" {
        return p.processUpdated(payload, memoID)
    } else if payload.ActivityType == "memos.memo.deleted" {
        return p.processDeleted(payload, memoID)
    }
    return nil
}

func (p *WebhookProcessor) processCreated(payload WebhookRequestPayload, memoID string) error {
    // ⚠️ 这是权威事件
    // 1. 创建或覆盖本地记录
    if err := p.storage.UpsertMemo(payload.Memo); err != nil {
        return err
    }
    
    // 2. 标记为已确认
    p.confirmedMemos[memoID] = true
    
    // 3. 清空 pending updates（因为 created 包含完整状态）
    delete(p.pendingUpdates, memoID)
    
    log("Processed created event for", memoID)
    return nil
}

func (p *WebhookProcessor) processUpdated(payload WebhookRequestPayload, memoID string) error {
    // 检查是否已通过 created 确认
    if p.confirmedMemos[memoID] {
        // ⚠️ 这是真正的更新操作
        // 可以安全更新
        localMemo, err := p.storage.GetMemo(memoID)
        if err != nil {
            return err
        }
        
        // 比较 update_time，确保不会用旧数据覆盖新数据
        payloadTime := parseTime(payload.Memo.UpdateTime)
        localTime := parseTime(localMemo.UpdateTime)
        
        if payloadTime.After(localTime) {
            if err := p.storage.UpdateMemo(payload.Memo); err != nil {
                return err
            }
            log("Processed update event for", memoID)
        } else {
            log("Ignoring stale update for", memoID, "(payload time:", payloadTime, "< local time:", localTime, ")")
        }
        return nil
    }
    
    // ⚠️ 还没有收到 created 事件
    // 这可能是创建时的附件/关系设置
    // 策略：暂存，等待 created
    
    if _, exists := p.pendingUpdates[memoID]
    if !exists {
        p.pendingUpdates[memoID] = []*WebhookRequestPayload{}
    }
    
    // 只保留最新的 update（按 update_time）
    // 或者：保留所有，等 created 来了一起处理
    
    p.pendingUpdates[memoID] = append(p.pendingUpdates[memoID], &payload)
    log("Stored pending update for", memoID, "(waiting for created)")
    return nil
}

func (p *WebhookProcessor) processDeleted(payload WebhookRequestPayload, memoID string) error {
    // 删除操作总是可以处理
    if err := p.storage.DeleteMemo(memoID); err != nil {
        return err
    }
    
    // 清理内存状态
    delete(p.confirmedMemos, memoID)
    delete(p.pendingUpdates, memoID)
    
    log("Processed deleted event for", memoID)
    return nil
}
```

### 8.2 外部服务响应格式

如果要实现一个接收 Memos webhook 的外部服务，必须返回：

```go
// 推荐的处理方式
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 读取请求体
    body, _ := io.ReadAll(r.Body)
    
    // 2. 解析请求体
    var payload WebhookRequestPayload
    json.Unmarshal(body, &payload)
    
    // 3. 处理业务逻辑
    // 注意：要处理时序问题！
    processor.Process(payload)
    
    // 4. ⚠️ 必须返回这样的响应！
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "code":    0,
        "message": "success",
    })
}
```

### 8.3 监控与可靠性

1. **Webhook 日志**：建议外部服务实现 webhook 接收日志，方便调试时序问题
2. **轮询补充**：对于关键数据建议使用轮询 API 作为 webhook 的补充
3. **幂等性**：确保处理 webhook 的逻辑是幂等的（可以重复处理不会有问题）

---

## 九、总结

### 9.1 核心发现

1. **时序问题是最大的坑**：
   - 创建带附件或关系的 memo 时，`updated` 会先于 `created` 触发
   - 这不是 bug，而是当前代码实现的副产品
   - 外部消费方必须处理这个时序

2. **请求体包含 url 字段**：
   - `WebhookRequestPayload.URL` 有 `json:"url"` tag
   - 会被序列化到请求体中发送

3. **响应必须严格格式**：
   - HTTP 2xx + `{"code": 0}`
   - 否则会被视为失败

4. **共用路径**：
   - REST API 和 MCP 工具最终调用相同的 `APIV1Service` 方法
   - 时序问题是统一的行为

### 9.2 快速参考

**创建 Memo 时的事件序列：

| 创建场景 | 事件序列 | 推荐处理方式 |
|---------|----------|-------------|
| 无附件无关系 | `[created]` | 直接创建 |
| 带附件 | `[updated, created]` | 忽略第一个 updated，等待 created |
| 带关系 | `[updated, created]` | 忽略第一个 updated，等待 created |
| 带附件+关系 | `[updated, updated, created]` | 忽略前两个 updated，等待 created |

**判重策略快速选择**：

| 策略 | 复杂度 | 适用场景 |
|------|--------|----------|
| 策略 C（以 created 为权威 | 低 | 大多数场景推荐 |
| 策略 A（暂存+等待） | 中 | 需要精确同步 |
| 策略 B（完整状态机） | 高 | 复杂同步场景 |
