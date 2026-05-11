# Memo 内容归档与彻底删除生命周期分析报告

## 1. 概述

本报告详细分析 51-memos 项目中 memo 内容的归档（Archive）和彻底删除（Delete/Purge）两个阶段的生命周期管理机制，包括多模块间的协调方式和生命周期事件的传播路径。

---

## 2. 核心数据模型

### 2.1 RowStatus 状态定义

位置：`store/common.go:12-20`

```go
type RowStatus string

const (
    Normal   RowStatus = "NORMAL"
    Archived RowStatus = "ARCHIVED"
)
```

### 2.2 Proto 定义

位置：`proto/api/v1/common.proto:7-11`

```protobuf
enum State {
  STATE_UNSPECIFIED = 0;
  NORMAL = 1;
  ARCHIVED = 2;
}
```

### 2.3 数据库层

位置：`store/migration/sqlite/LATEST.sql:14,39`

```sql
row_status TEXT NOT NULL CHECK (row_status IN ('NORMAL', 'ARCHIVED')) DEFAULT 'NORMAL'
```

所有三种数据库驱动（SQLite、MySQL、PostgreSQL）都支持 `row_status` 字段。

---

## 3. 第一阶段：归档（Archive）

### 3.1 归档流程概述

归档是**软删除**机制，memo 不会从数据库中移除，只是将状态从 `NORMAL` 变更为 `ARCHIVED`。

### 3.2 前端入口

位置：`web/src/components/MemoActionMenu/MemoActionMenu.tsx:104-109`

- **触发方式**：用户点击菜单中的 "Archive" 或 "Restore" 按钮

### 3.3 前端处理逻辑

位置：`web/src/components/MemoActionMenu/hooks.ts:55-81`

```typescript
const handleToggleMemoStatusClick = useCallback(async () => {
  const isArchiving = memo.state !== State.ARCHIVED;
  const state = memo.state === State.ARCHIVED ? State.NORMAL : State.ARCHIVED;
  
  await updateMemo({
    update: { name: memo.name, state },
    updateMask: ["state"],
  });
  
  // 导航处理：如果在详情页归档则跳转到首页/归档页
  if (isInMemoDetailPage) {
    navigateTo(memo.state === State.ARCHIVED ? ROUTES.HOME : ROUTES.ARCHIVED);
  }
}, [...]);
```

### 3.4 API 层处理

位置：`server/router/api/v1/memo_service.go:460-575`

**关键步骤**：

1. **权限校验**：只有创建者或管理员才能修改 memo 状态
   ```go
   if memo.CreatorID != user.ID && !isSuperUser(user) {
       return status.Errorf(codes.PermissionDenied, "permission denied")
   }
   ```

2. **更新操作**：通过 UpdateMemo 更新 row_status
   ```go
   update := &store.UpdateMemo{
       ID: memo.ID,
       RowStatus: &rowStatus,  // Normal 或 Archived
   }
   ```

3. **事件分发**：调用 `dispatchMemoUpdatedSideEffects

### 3.5 存储层处理

位置：`store/memo.go:133-138`

```go
func (s *Store) UpdateMemo(ctx context.Context, update *UpdateMemo) error {
    return s.driver.UpdateMemo(ctx, update)
}
```

### 3.6 归档状态的可见性控制

位置：`server/router/api/v1/memo_service.go:47-56`

**归档 memo 的访问规则**：

- 归档的 memo **仅对其创建者可见
- 其他用户访问会返回 NotFound

```go
if memo.RowStatus == store.Archived {
    user, err := s.fetchCurrentUser(ctx)
    if err != nil {
        return status.Errorf(codes.Internal, "failed to get user")
    }
    if user == nil || memo.CreatorID != user.ID {
        return status.Errorf(codes.NotFound, "memo not found")
    }
}
```

### 3.7 归档列表查询

位置：`server/router/api/v1/memo_service.go:199-210`

```go
if request.State == v1pb.State_ARCHIVED {
    state := store.Archived
    memoFind.RowStatus = &state
    memoFind.CreatorID = &currentUser.ID  // 仅查询当前用户的归档
} else {
    state := store.Normal
    memoFind.RowStatus = &state
}
```

前端归档页面：`web/src/pages/Archived.tsx:19-22`

---

## 4. 第二阶段：彻底删除（Delete/Purge）

### 4.1 彻底删除流程概述

彻底删除是**硬删除**，memo 及其关联数据会从数据库中永久移除。

### 4.2 前端入口

位置：`web/src/components/MemoActionMenu/MemoActionMenu.tsx:111-117`

- 有确认对话框防止误操作

### 4.3 前端处理逻辑

位置：`web/src/components/MemoActionMenu/hooks.ts:97-116`

```typescript
const handleDeleteMemoClick = useCallback(() => {
  setDeleteDialogOpen(true);  // 打开确认对话框
}, ...);

const confirmDeleteMemo = useCallback(async () => {
  await deleteMemo(memo.name);
  // 导航处理
  if (isInMemoDetailPage) {
    navigateTo(ROUTES.HOME);
  }
}, ...);
```

### 4.4 API 层处理

位置：`server/router/api/v1/memo_service.go:577-652`

**关键步骤**：

1. **权限校验**：同归档，只有创建者或管理员才能删除

2. **关联数据清理前的事件通知**：
   ```go
   // 在删除前获取 reactions、attachments、relations
   // 尝试发送 webhook 通知
   if err := s.DispatchMemoDeletedWebhook(ctx, memoMessage); err != nil {
       slog.Warn("Failed to dispatch memo deleted webhook", ...)
   }
   ```

3. **级联删除评论**：
   ```go
   // 先删除该 memo 的所有评论（递归调用 DeleteMemo）
   for _, relation := range relations {
       if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
           return status.Errorf(codes.Internal, "failed to delete memo comment")
       }
   }
   ```

4. **删除 memo 本体**：
   ```go
   if err = s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: memo.ID}); err != nil {
       return status.Errorf(codes.Internal, "failed to delete memo")
   }
   ```

5. **广播删除事件**：
   ```go
   s.SSEHub.Broadcast(&SSEEvent{
       Type:       SSEEventMemoDeleted,
       Name:       request.Name,
       Visibility: memo.Visibility,
       CreatorID:  resolveSSECreatorID(memo, nil),
   });
   ```

### 4.5 存储层处理

位置：`store/memo.go:140-159`

**DeleteMemo 方法的完整清理流程**：

```go
func (s *Store) DeleteMemo(ctx context.Context, delete *DeleteMemo) error {
    // 1. 清理 memo_relation 记录（作为源或目标）
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{RelatedMemoID: &delete.ID}); err != nil {
        return err
    }
    // 2. 清理 attachments
    attachments, err := s.ListAttachments(ctx, &FindAttachment{MemoID: &delete.ID})
    for _, attachment := range attachments {
        if err := s.DeleteAttachment(ctx, &DeleteAttachment{ID: attachment.ID}); err != nil {
            return err
        }
    }
    // 3. 从数据库删除 memo 本体
    return s.driver.DeleteMemo(ctx, delete)
}
```

### 4.6 数据库驱动实现

三种数据库驱动（SQLite、MySQL、PostgreSQL）都实现了 `DeleteMemo 方法：

- `store/db/sqlite/memo.go
- `store/db/mysql/memo.go
- `store/db/postgres/memo.go

---

## 5. 生命周期事件传播机制

### 5.1 事件传播架构概览

```
用户操作
    │
    ├── SSE（实时推送 → 前端实时刷新
    │
    ├── Webhook → 外部系统通知
    │
    └── React Query 缓存 → 前端本地缓存更新
```

### 5.2 SSE（Server-Sent Events）机制

#### 5.2.1 SSE 事件类型定义

位置：`server/router/api/v1/sse_hub.go:14-21`

```go
const (
    SSEEventMemoCreated        SSEEventType = "memo.created"
    SSEEventMemoUpdated        SSEEventType = "memo.updated"
    SSEEventMemoDeleted        SSEEventType = "memo.deleted"
    SSEEventMemoCommentCreated SSEEventType = "memo.comment.created"
    SSEEventReactionUpserted   SSEEventType = "reaction.upserted"
    SSEEventReactionDeleted    SSEEventType = "reaction.deleted"
)
```

#### 5.2.2 归档操作触发的事件

归档操作通过 `UpdateMemo 触发 `memo.updated` 事件：

位置：`server/router/api/v1/memo_update_helpers.go:66-78`

```go
func (s *APIV1Service) dispatchMemoUpdatedSideEffects(ctx context.Context, memo *store.Memo, parentMemo *store.Memo, memoMessage *v1pb.Memo) {
    if err := s.DispatchMemoUpdatedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo updated webhook", slog.Any("err", err))
    }

    s.SSEHub.Broadcast(&SSEEvent{
        Type:       SSEEventMemoUpdated,
        Name:       memoMessage.Name,
        Parent:     memoMessage.GetParent(),
        Visibility: memo.Visibility,
        CreatorID:  resolveSSECreatorID(memo, parentMemo),
    })
}
```

#### 5.2.3 彻底删除操作触发的事件

删除操作触发 `memo.deleted` 事件：

位置：`server/router/api/v1/memo_service.go:644-649`

```go
s.SSEHub.Broadcast(&SSEEvent{
    Type:       SSEEventMemoDeleted,
    Name:       request.Name,
    Visibility: memo.Visibility,
    CreatorID:  resolveSSECreatorID(memo, nil),
})
```

#### 5.2.4 SSE Hub 广播机制

位置：`server/router/api/v1/sse_hub.go:115-132`

```go
func (h *SSEHub) Broadcast(event *SSEEvent) {
    data := event.JSON()
    if len(data) == 0 {
        return
    }
    h.mu.RLock()
    defer h.mu.RUnlock()
    for c := range h.clients {
        if !c.canReceive(event) {  // 基于权限过滤
            continue
        }
        select {
        case c.events <- data:
        default:
            // 慢客户端丢弃事件避免阻塞
        }
    }
}
```

#### 5.2.5 SSE 事件权限过滤

位置：`server/router/api/v1/sse_hub.go:134-144`

```go
func (c *SSEClient) canReceive(event *SSEEvent) bool {
    switch event.Visibility {
    case store.Private:
        return c.userID == event.CreatorID || c.role == store.RoleAdmin
    case store.Public, store.Protected, "":
        return true
    default:
        return false
    }
}
```

#### 5.2.6 前端 SSE 事件处理

位置：`web/src/hooks/useLiveMemoRefresh.ts:219-254`

```typescript
function handleSSEEvent(event: SSEChangeEvent, queryClient: ReturnType<typeof useQueryClient>) {
    switch (event.type) {
        case SSE_EVENT_TYPES.memoUpdated:
            queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
            queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
            if (event.parent) {
                queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
            }
            break;

        case SSE_EVENT_TYPES.memoDeleted:
            queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
            queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
            queryClient.invalidateQueries({ queryKey: userKeys.stats() });
            break;
    }
}
```

### 5.3 Webhook 机制

#### 5.3.1 Webhook 事件类型

位置：`server/router/api/v1/memo_service.go:921-934`

```go
func (s *APIV1Service) DispatchMemoCreatedWebhook(ctx context.Context, memo *v1pb.Memo) error {
    return s.dispatchMemoRelatedWebhook(ctx, memo, "memos.memo.created")
}

func (s *APIV1Service) DispatchMemoUpdatedWebhook(ctx context.Context, memo *v1pb.Memo) error {
    return s.dispatchMemoRelatedWebhook(ctx, memo, "memos.memo.updated")
}

func (s *APIV1Service) DispatchMemoDeletedWebhook(ctx context.Context, memo *v1pb.Memo) error {
    return s.dispatchMemoRelatedWebhook(ctx, memo, "memos.memo.deleted")
}
```

#### 5.3.2 Webhook 异步分发

位置：`internal/webhook/webhook.go:126-140`

```go
func PostAsync(requestPayload *WebhookRequestPayload) {
    if requestPayload == nil {
        slog.Warn("Dropped webhook dispatch because payload is nil")
        return
    }
    select {
    case asyncPostQueue <- requestPayload:
    default:
        slog.Warn("Dropped webhook dispatch because the async queue is full", ...)
    }
}
```

#### 5.3.3 Webhook 工作池

位置：`internal/webhook/webhook.go:32-48`

```go
asyncPostQueue = make(chan *WebhookRequestPayload, 128)

func init() {
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

#### 5.3.4 Webhook Payload 结构

位置：`internal/webhook/webhook.go:72-81`

```go
type WebhookRequestPayload struct {
    URL          string      `json:"url"`
    ActivityType string      `json:"activityType"`
    Creator      string      `json:"creator"`
    Memo         *v1pb.Memo  `json:"memo"`
}
```

#### 5.3.5 Webhook SSRF 防护

位置：`internal/webhook/webhook.go:52-70`

- 阻止连接到保留/私有 IP 地址
- 防止 DNS 重绑定攻击

### 5.4 React Query 缓存机制

#### 5.4.1 归档操作的本地更新

位置：`web/src/components/MemoActionMenu/hooks.ts:32-35`

```typescript
const memoUpdatedCallback = useCallback(() => {
    queryClient.invalidateQueries({ queryKey: userKeys.stats() });
}, [queryClient]);
```

#### 5.4.2 删除操作的本地更新

位置：`web/src/components/MemoActionMenu/hooks.ts:101-116`

```typescript
const confirmDeleteMemo = useCallback(async () => {
    await deleteMemo(memo.name);
    
    if (memo.parent) {
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(memo.parent) });
    }
    
    memoUpdatedCallback();
}, ...);
```

---

## 6. 完整生命周期流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        正常状态 (NORMAL)                              │
│  - 所有用户可见（根据 visibility 控制）                             │
│  - 可编辑、可评论、可反应                                       │
└────────────────────────┬────────────────────────────────────────────┘
                     │
                     │ 点击 "Archive" 按钮
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      归档阶段 1: 状态变更                              │
│  UpdateMemo (row_status: NORMAL → ARCHIVED)                        │
│                                                                     │
│  事件传播:                                                         │
│  ├─ SSE: memo.updated                                        │
│  ├─ Webhook: memos.memo.updated                                 │
│  └─ React Query: 失效 memoKeys.detail + lists                 │
└────────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      归档状态 (ARCHIVED)                              │
│  - 仅创建者可见                                                 │
│  - 从正常列表中隐藏                                             │
│  - 在归档列表中显示                                               │
│  - 可恢复（Restore）                                              │
└────────────────────────┬────────────────────────────────────────────┘
                     │
         ┌─────────────┴─────────────┐
         │                       │
         │ 点击 "Restore"      │ 点击 "Delete"
         ▼                       ▼
┌─────────────────┐        ┌─────────────────────────────────────────┐
│ 恢复状态   │        │            彻底删除阶段 2: 级联删除          │
│            │        │  DeleteMemo (硬删除)                    │
│ 事件传播:  │        │                                           │
│ ├─ SSE:  │        │ 级联清理:                             │
│ │ memo.updated  │        │ ├─ 删除所有评论 (递归)            │
│ └─ Webhook:    │        │ ├─ 清理 memo_relation           │
│ │ memos.memo.updated │    │ ├─ 清理 attachments              │
│ └─ React Query:   │        │ └─ 从数据库删除 memo 本体          │
│ │ 失效缓存         │        │                                     │
│ └─ 重新查询       │        │ 事件传播:                         │
└────────────┘         │        │ ├─ SSE: memo.deleted          │
                       │        │ ├─ Webhook: memos.memo.deleted │
                       │        │ └─ React Query:               │
                       │        │   ├─ 移除 memoKeys.detail │
                       │        │   └─ 失效 memoKeys.lists │
                       │        │   + userKeys.stats       │
                       │        └─────────────────────────────────────┘
                       │
                       ▼
              ┌─────────────────────┐
              │  永久删除      │
              │  无法恢复      │
              └─────────────────┘
```

---

## 7. 多模块协调机制

### 7.1 模块职责划分

| 模块 | 职责 | 关键文件 |
|------|------|----------|
| 前端 UI | 用户交互、状态管理、导航处理 | MemoActionMenu.tsx, hooks.ts |
| API Service | 权限校验、业务编排、事件触发 | memo_service.go |
| Store | 数据操作、关联数据清理 | memo.go |
| SSE Hub | 实时事件广播、权限过滤 | sse_hub.go |
| Webhook | 异步通知外部系统 | webhook.go |
| React Query | 前端缓存管理 | useLiveMemoRefresh.ts |

### 7.2 归档流程协调要点

1. **前端 → API**：通过 gRPC/Connect 调用 UpdateMemo
2. **API → Store**：调用 driver.UpdateMemo
3. **API → SSE/Webhook**：dispatchMemoUpdatedSideEffects
4. **SSE → 前端**：useLiveMemoRefresh 监听事件
5. **Webhook → 外部系统**：异步队列分发

### 7.3 删除流程协调要点

1. **前端 → API**：通过 gRPC/Connect 调用 DeleteMemo
2. **API → Webhook**：先发送删除通知（在数据删除前）
3. **API → Store**：递归删除评论 → 清理 relations → 清理 attachments → 删除 memo
4. **API → SSE**：广播 memo.deleted 事件
5. **SSE → 前端**：移除缓存 + 失效列表缓存

---

## 8. 关键设计决策

### 8.1 两阶段设计

**归档（软删除）**：

- 优点：可恢复、误操作保护、审计追踪
- 适用场景：临时移除、内容整理

**彻底删除（硬删除）**：

- 优点：释放存储空间、彻底清除敏感数据
- 适用场景：确定不需要的数据清理

### 8.2 事件传播顺序

**删除时的特殊处理**：

- Webhook 在删除前发送（确保能获取完整数据）
- SSE 在删除后发送（确保状态一致）

### 8.3 权限控制

- 归档和删除都需要创建者或管理员权限
- 归档内容仅创建者可见

### 8.4 级联删除

- 评论递归删除
- relations 双向清理
- attachments 清理

### 8.5 异步处理

- Webhook 使用异步队列（4 个 worker）
- 队列大小 128
- 慢客户端丢弃避免阻塞

---

## 9. 相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `store/common.go | RowStatus 定义 |
| `store/memo.go | 存储层实现 |
| `server/router/api/v1/memo_service.go | API 服务实现 |
| `server/router/api/v1/memo_update_helpers.go | 更新副作用处理 |
| `server/router/api/v1/sse_hub.go | SSE Hub 实现 |
| `internal/webhook/webhook.go | Webhook 实现 |
| `web/src/components/MemoActionMenu/MemoActionMenu.tsx | 前端操作菜单 |
| `web/src/components/MemoActionMenu/hooks.ts | 前端操作处理 |
| `web/src/hooks/useLiveMemoRefresh.ts | 前端 SSE 监听 |
| `web/src/pages/Archived.tsx | 归档页面 |
| `proto/api/v1/common.proto | Proto 状态定义 |
