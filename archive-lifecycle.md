# Memo 内容归档与彻底删除生命周期分析报告

## 1. 概述

本报告详细分析 51-memos 项目中 memo 内容的三条生命周期操作路径：

1. **正常态直删**：从 NORMAL 状态直接硬删除
2. **归档后删除**：先归档到 ARCHIVED，再硬删除
3. **归档后恢复**：从 ARCHIVED 状态恢复到 NORMAL

报告涵盖各路径的前端触发、API 校验、Store 清理、事件传播（SSE/Webhook/缓存刷新）的先后顺序和失败处理机制。

---

## 2. 核心数据模型

### 2.1 状态定义

位置：`store/common.go:12-20`

```go
type RowStatus string

const (
    Normal   RowStatus = "NORMAL"
    Archived RowStatus = "ARCHIVED"
)
```

位置：`proto/api/v1/common.proto:7-11`

```protobuf
enum State {
  STATE_UNSPECIFIED = 0;
  NORMAL = 1;
  ARCHIVED = 2;
}
```

### 2.2 数据库层

位置：`store/migration/sqlite/LATEST.sql:14,39`

```sql
row_status TEXT NOT NULL CHECK (row_status IN ('NORMAL', 'ARCHIVED')) DEFAULT 'NORMAL'
```

所有三种数据库驱动（SQLite、MySQL、PostgreSQL）都支持 `row_status` 字段。

---

## 3. 三条操作路径总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                     │
│              正常状态 (NORMAL)                                       │
│         ┌─────────────────────────────────────────┐                │
│         │ - 所有用户可见（根据 visibility）          │                │
│         │ - 可编辑、可评论、可反应                   │                │
│         └───────────────┬─────────────────────────┘                │
│                         │                                            │
│         ┌───────────────┴───────────────┐                            │
│         │                               │                            │
│         ▼                               ▼                            │
│  ┌───────────────┐              ┌───────────────┐                    │
│  │ 路径 1:        │              │ 路径 2:        │                    │
│  │ 正常态直删     │              │ 归档          │                    │
│  │ (硬删除)       │              │ (软删除)       │                    │
│  └───────┬───────┘              └───────┬───────┘                    │
│          │                              │                            │
│          │                              ▼                            │
│          │                    ┌─────────────────┐                    │
│          │                    │ 归档状态         │                    │
│          │                    │ (ARCHIVED)      │                    │
│          │                    │                 │                    │
│          │                    │ - 仅创建者可见    │                    │
│          │                    │ - 从正常列表隐藏   │                    │
│          │                    │ - 在归档列表显示   │                    │
│          │                    └────────┬────────┘                    │
│          │                             │                             │
│          │              ┌──────────────┴──────────────┐              │
│          │              │                             │              │
│          │              ▼                             ▼              │
│          │       ┌───────────────┐           ┌───────────────┐       │
│          │       │ 路径 3:        │           │ 路径 1 (变体):  │       │
│          │       │ 归档后恢复     │           │ 归档后删除      │       │
│          │       │ (恢复到正常)   │           │ (硬删除)        │       │
│          │       └───────┬───────┘           └───────┬───────┘       │
│          │               │                           │                │
│          │               │                           │                │
│          └───────────────┘                           │                │
│                                                      ▼                │
│                                            ┌─────────────────┐        │
│                                            │ 永久删除         │        │
│                                            │ (无法恢复)        │        │
│                                            └─────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 路径 1：正常态直删（硬删除）

**路径**：NORMAL → 永久删除

### 4.1 前端触发

位置：`web/src/components/MemoActionMenu/MemoActionMenu.tsx:111-117`

```typescript
// 无论 memo.state 是 NORMAL 还是 ARCHIVED，都可以触发删除
<DropdownMenuItem onClick={handleDeleteMemoClick}>
  <TrashIcon className="w-4 h-auto" />
  {t("common.delete")}
</DropdownMenuItem>
```

位置：`web/src/components/MemoActionMenu/hooks.ts:97-116`

```typescript
const handleDeleteMemoClick = useCallback(() => {
  setDeleteDialogOpen(true);  // 打开确认对话框
}, [setDeleteDialogOpen]);

const confirmDeleteMemo = useCallback(async () => {
  try {
    await deleteMemo(memo.name);
  } catch (error: unknown) {
    handleError(error, toast.error, { 
      context: "Delete memo", 
      fallbackMessage: "An error occurred" 
    });
    return;
  }
  toast.success(t("message.deleted-successfully"));
  
  // 导航处理
  if (memo.parent) {
    queryClient.invalidateQueries({ queryKey: memoKeys.comments(memo.parent) });
  }
  if (isInMemoDetailPage) {
    navigateTo(ROUTES.HOME);
  }
  memoUpdatedCallback();
}, [memo.name, memo.parent, t, isInMemoDetailPage, navigateTo, memoUpdatedCallback, deleteMemo, queryClient]);
```

**前端触发流程**：
1. 用户点击 "Delete" 菜单
2. 打开确认对话框
3. 用户确认后调用 `deleteMemo`
4. 成功后：
   - 显示成功 toast
   - 如果是评论，失效父评论缓存
   - 如果在详情页，导航到首页
   - 失效用户统计缓存
5. 失败后：
   - 显示错误 toast
   - 不执行导航和缓存操作

---

### 4.2 API 校验

位置：`server/router/api/v1/memo_service.go:577-602`

```go
func (s *APIV1Service) DeleteMemo(ctx context.Context, request *v1pb.DeleteMemoRequest) (*emptypb.Empty, error) {
    // 步骤 1: 解析 memo UID
    memoUID, err := ExtractMemoUIDFromName(request.Name)
    if err != nil {
        return nil, status.Errorf(codes.InvalidArgument, "invalid memo name: %v", err)
    }
    
    // 步骤 2: 从数据库获取 memo
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{UID: &memoUID})
    if err != nil {
        return nil, err
    }
    if memo == nil {
        return nil, status.Errorf(codes.NotFound, "memo not found")
    }
    
    // 步骤 3: 获取当前用户
    user, err := s.fetchCurrentUser(ctx)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get current user")
    }
    if user == nil {
        return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
    }
    
    // 步骤 4: 权限校验 - 只有创建者或管理员可以删除
    if memo.CreatorID != user.ID && !isSuperUser(user) {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
    // ...
}
```

**API 校验步骤及失败处理**：

| 步骤 | 校验内容 | 失败时错误码 | 失败处理 |
|------|----------|-------------|----------|
| 1 | 解析 memo UID 格式 | InvalidArgument | 返回错误，不执行后续操作 |
| 2 | 从数据库获取 memo | NotFound | memo 不存在时返回 |
| 3 | 用户认证 | Unauthenticated | 未登录用户无法操作 |
| 4 | 权限校验 | PermissionDenied | 非创建者且非管理员拒绝 |

**关键设计**：删除操作不校验 `memo.RowStatus`，NORMAL 和 ARCHIVED 状态都可以直接删除。

---

### 4.3 Webhook 发送（删除前）

位置：`server/router/api/v1/memo_service.go:604-624`

```go
// 步骤 5: 在删除前收集关联数据，用于 webhook
reactions, err := s.Store.ListReactions(ctx, &store.FindReaction{
    ContentID: &request.Name,
})
if err != nil {
    return nil, status.Errorf(codes.Internal, "failed to list reactions")
}

attachments, err := s.Store.ListAttachments(ctx, &store.FindAttachment{
    MemoID: &memo.ID,
})
if err != nil {
    return nil, status.Errorf(codes.Internal, "failed to list attachments")
}

deleteRelations, _ := s.loadMemoRelations(ctx, memo)
if memoMessage, err := s.convertMemoFromStore(ctx, memo, reactions, attachments, deleteRelations); err == nil {
    // 步骤 6: 发送 webhook（异步，失败只记录警告）
    if err := s.DispatchMemoDeletedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo deleted webhook", slog.Any("err", err))
    }
}
```

**Webhook 发送步骤**：
1. 收集 reactions（失败则中断整个删除）
2. 收集 attachments（失败则中断整个删除）
3. 收集 relations（失败不中断，忽略）
4. 构建完整的 memoMessage
5. 异步发送 webhook（`memos.memo.deleted`）
   - webhook 发送失败不会影响删除操作
   - 失败仅记录 slog.Warn

**Webhook 实现**：

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

- 使用异步队列（容量 128）
- 4 个 worker goroutine 处理
- 队列满时丢弃并记录警告
- 单个 webhook 超时 30 秒

---

### 4.4 Store 清理

位置：`server/router/api/v1/memo_service.go:626-641`

```go
// 步骤 7: 先删除该 memo 的所有评论（递归调用 DeleteMemo）
commentType := store.MemoRelationComment
relations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{
    RelatedMemoID: &memo.ID, 
    Type: &commentType,
})
if err != nil {
    return nil, status.Errorf(codes.Internal, "failed to list memo comments")
}
for _, relation := range relations {
    if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
        return nil, status.Errorf(codes.Internal, "failed to delete memo comment")
    }
}

// 步骤 8: 删除 memo 本体（包含关联数据清理）
if err = s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: memo.ID}); err != nil {
    return nil, status.Errorf(codes.Internal, "failed to delete memo")
}
```

位置：`store/memo.go:140-159`

```go
func (s *Store) DeleteMemo(ctx context.Context, delete *DeleteMemo) error {
    // 步骤 8.1: 清理 memo_relation 记录（作为源）
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    // 步骤 8.2: 清理 memo_relation 记录（作为目标）
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{RelatedMemoID: &delete.ID}); err != nil {
        return err
    }
    // 步骤 8.3: 清理 attachments
    attachments, err := s.ListAttachments(ctx, &FindAttachment{MemoID: &delete.ID})
    if err != nil {
        return err
    }
    for _, attachment := range attachments {
        if err := s.DeleteAttachment(ctx, &DeleteAttachment{ID: attachment.ID}); err != nil {
            return err
        }
    }
    // 步骤 8.4: 从数据库删除 memo 本体
    return s.driver.DeleteMemo(ctx, delete)
}
```

**Store 清理顺序及失败处理**：

| 步骤 | 操作内容 | 失败时 | 失败影响 |
|------|----------|--------|----------|
| 7 | 列出评论 relations | Internal | 中断删除 |
| 7.1 | 递归删除每条评论（评论自身也会执行完整删除流程） | Internal | 中断删除，已删除的评论不会回滚 |
| 8.1 | 删除作为源的 memo_relation | error | 中断删除 |
| 8.2 | 删除作为目标的 memo_relation | error | 中断删除 |
| 8.3 | 列出 attachments | error | 中断删除 |
| 8.4 | 逐个删除 attachment | error | 中断删除，已删除的不会回滚 |
| 8.5 | 删除 memo 本体 | error | 中断删除 |

**重要说明**：删除操作没有事务保护，是分步执行的。如果中途失败，已删除的数据不会回滚。

---

### 4.5 SSE 广播（删除后）

位置：`server/router/api/v1/memo_service.go:643-649`

```go
// 步骤 9: 广播删除事件
s.SSEHub.Broadcast(&SSEEvent{
    Type:       SSEEventMemoDeleted,
    Name:       request.Name,
    Visibility: memo.Visibility,
    CreatorID:  resolveSSECreatorID(memo, nil),
})
```

**SSE 广播机制**：

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
        if !c.canReceive(event) {  // 权限过滤
            continue
        }
        select {
        case c.events <- data:
        default:
            // 慢客户端丢弃事件，不阻塞广播
        }
    }
}
```

**SSE 权限过滤**：

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

**SSE 失败处理**：
- JSON 序列化失败：静默忽略
- 客户端缓冲满：丢弃事件，不阻塞
- 权限不匹配：不发送

---

### 4.6 前端缓存刷新

位置：`web/src/hooks/useLiveMemoRefresh.ts:390-394`

```typescript
case SSE_EVENT_TYPES.memoDeleted:
    queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
    queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
    queryClient.invalidateQueries({ queryKey: userKeys.stats() });
    break;
```

**缓存操作顺序**：
1. 移除详情缓存（`memoKeys.detail`）
2. 失效列表缓存（`memoKeys.lists`）
3. 失效用户统计缓存（`userKeys.stats`）

---

### 4.7 正常态直删完整时序图

```
前端                                    API                                   Store                             Webhook                         SSE
 │                                       │                                      │                                  │                              │
 │ 用户点击 Delete                        │                                      │                                  │                              │
 │ ───────────────────────────────────►  │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 1. 解析 UID                           │                                  │                              │
 │                                       │ 2. GetMemo                           │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 返回 memo (状态=NORMAL)         │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 3. 权限校验 (creator/admin)          │                                  │                              │
 │                                       │    失败 → 返回 PermissionDenied      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 4. 收集关联数据                       │                                  │                              │
 │                                       │    ListReactions                    │ ─────────────────────────────►   │                              │
 │                                       │    ListAttachments                  │ ─────────────────────────────►   │                              │
 │                                       │    loadMemoRelations                │ ─────────────────────────────►   │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 5. 发送 Webhook (异步)               │                                  │ PostAsync                    │
 │                                       │    失败只 Warn，不中断删除            │                                  │ ◄────────────────────────────  │
 │                                       │                                      │                                  │ 队列 (容量128)                 │
 │                                       │                                      │                                  │ 4个worker处理                 │
 │                                       │                                      │                                  │                              │
 │                                       │ 6. 级联删除评论                       │                                  │                              │
 │                                       │    对每个评论递归调用 DeleteMemo      │ ─────────────────────────────►   │                              │
 │                                       │    (评论自身也执行完整删除流程)       │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 7. DeleteMemo 本体                   │                                  │                              │
 │                                       │    ──────────────────────────────►   │                                  │                              │
 │                                       │                                      │ 7.1 删除 memo_relation (源)    │                              │
 │                                       │                                      │ 7.2 删除 memo_relation (目标)  │                              │
 │                                       │                                      │ 7.3 逐个删除 attachments        │                              │
 │                                       │                                      │ 7.4 删除 memo 本体              │                              │
 │                                       │    ◄───────────────────────────────   │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 8. Broadcast SSE                     │                                  │                              │
 │                                       │    Type: memo.deleted                │                                  │                              │
 │                                       │ ──────────────────────────────────────────────────────────────────────► │
 │                                       │                                      │                                  │                              │
 │ ◄────────────────────────────────────────────────────────────────────────────────────────────────────────── │
 │ SSE memo.deleted                       │                                      │                                  │                              │
 │ ────────────────────────────►         │                                      │                                  │                              │
 │ 1. removeQueries(detail)              │                                      │                                  │                              │
 │ 2. invalidateQueries(lists)           │                                      │                                  │                              │
 │ 3. invalidateQueries(stats)           │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │ 导航到首页                             │                                      │                                  │                              │
 │ 显示成功 Toast                         │                                      │                                  │                              │
```

---

## 5. 路径 2：归档（软删除）

**路径**：NORMAL → ARCHIVED

### 5.1 前端触发

位置：`web/src/components/MemoActionMenu/MemoActionMenu.tsx:104-109`

```typescript
// 评论不能归档/恢复
// 只有非评论 memo 显示此选项
{!isComment && (
    <DropdownMenuItem onClick={handleToggleMemoStatusClick}>
        {isArchived ? <ArchiveRestoreIcon ... /> : <ArchiveIcon ... />}
        {isArchived ? t("common.restore") : t("common.archive")}
    </DropdownMenuItem>
)}
```

位置：`web/src/components/MemoActionMenu/hooks.ts:55-81`

```typescript
const handleToggleMemoStatusClick = useCallback(async () => {
    const isArchiving = memo.state !== State.ARCHIVED;
    const state = memo.state === State.ARCHIVED ? State.NORMAL : State.ARCHIVED;
    const message = memo.state === State.ARCHIVED 
        ? t("message.restored-successfully") 
        : t("message.archived-successfully");

    try {
        await updateMemo({
            update: {
                name: memo.name,
                state,
            },
            updateMask: ["state"],  // 只更新 state 字段
        });
        toast.success(message);
    } catch (error: unknown) {
        handleError(error, toast.error, {
            context: `${isArchiving ? "Archive" : "Restore"} memo`,
            fallbackMessage: "An error occurred",
        });
        return;
    }

    // 导航处理
    if (isInMemoDetailPage) {
        navigateTo(memo.state === State.ARCHIVED ? ROUTES.HOME : ROUTES.ARCHIVED);
    }
    memoUpdatedCallback();
}, [memo.name, memo.state, t, isInMemoDetailPage, navigateTo, memoUpdatedCallback, updateMemo]);
```

**前端触发流程**：
1. 判断当前状态（isArchived）
2. 计算目标状态
3. 调用 `updateMemo`，`updateMask: ["state"]`
4. 成功后：
   - 显示对应 toast
   - 如果在详情页，根据操作导航：
     - 归档 → 跳转到首页
     - 恢复 → 跳转到归档页
   - 失效用户统计缓存
5. 失败后：
   - 显示错误 toast
   - 不执行导航和缓存操作

---

### 5.2 API 校验

位置：`server/router/api/v1/memo_service.go:460-487`

```go
func (s *APIV1Service) UpdateMemo(ctx context.Context, request *v1pb.UpdateMemoRequest) (*v1pb.Memo, error) {
    // 步骤 1: 解析 memo UID
    memoUID, err := ExtractMemoUIDFromName(request.Memo.Name)
    if err != nil {
        return nil, status.Errorf(codes.InvalidArgument, "invalid memo name: %v", err)
    }
    
    // 步骤 2: 校验 updateMask
    if request.UpdateMask == nil || len(request.UpdateMask.Paths) == 0 {
        return nil, status.Errorf(codes.InvalidArgument, "update mask is required")
    }
    
    // 步骤 3: 从数据库获取 memo
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{UID: &memoUID})
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get memo: %v", err)
    }
    if memo == nil {
        return nil, status.Errorf(codes.NotFound, "memo not found")
    }
    
    // 步骤 4: 获取当前用户
    user, err := s.fetchCurrentUser(ctx)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get current user")
    }
    if user == nil {
        return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
    }
    
    // 步骤 5: 权限校验 - 只有创建者或管理员可以更新
    if memo.CreatorID != user.ID && !isSuperUser(user) {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
    // ...
}
```

位置：`server/router/api/v1/memo_service.go:526-528`

```go
// 步骤 6: 处理 state 更新
} else if path == "state" {
    rowStatus := convertStateToStore(request.Memo.State)
    update.RowStatus = &rowStatus
}
```

**API 校验步骤及失败处理**：

| 步骤 | 校验内容 | 失败时错误码 | 失败处理 |
|------|----------|-------------|----------|
| 1 | 解析 memo UID 格式 | InvalidArgument | 返回错误 |
| 2 | updateMask 必填 | InvalidArgument | 返回错误 |
| 3 | 从数据库获取 memo | NotFound | memo 不存在时返回 |
| 4 | 用户认证 | Unauthenticated | 未登录用户无法操作 |
| 5 | 权限校验 | PermissionDenied | 非创建者且非管理员拒绝 |
| 6 | 状态转换 | Internal | Store 更新失败时返回 |

**关键设计**：UpdateMemo 不校验当前状态和目标状态的关系，可以：
- NORMAL → ARCHIVED（归档）
- ARCHIVED → NORMAL（恢复）
- NORMAL → NORMAL（无变化，但仍会触发事件）
- ARCHIVED → ARCHIVED（无变化，但仍会触发事件）

---

### 5.3 Store 更新

位置：`store/memo.go:133-138`

```go
func (s *Store) UpdateMemo(ctx context.Context, update *UpdateMemo) error {
    if update.UID != nil && !base.UIDMatcher.MatchString(*update.UID) {
        return errors.New("invalid uid")
    }
    return s.driver.UpdateMemo(ctx, update)
}
```

位置：`store/db/sqlite/memo.go:205-207`（以 SQLite 为例）

```go
if v := update.RowStatus; v != nil {
    set, args = append(set, "`row_status` = ?"), append(args, *v)
}
```

**Store 更新失败处理**：
- 数据库层面的更新失败会返回 error
- 三种驱动实现类似：MySQL、PostgreSQL 都有对应实现

---

### 5.4 Webhook 发送

位置：`server/router/api/v1/memo_service.go:569-572`

```go
// 重新获取最新状态
memo, err = s.Store.GetMemo(ctx, &store.FindMemo{ID: &memo.ID})
if err != nil {
    return nil, errors.Wrap(err, "failed to get memo")
}
memo, parentMemo, memoMessage, err := s.buildUpdatedMemoState(ctx, memo.ID)
if err != nil {
    return nil, errors.Wrap(err, "failed to build updated memo state")
}
// ...
s.dispatchMemoUpdatedSideEffects(ctx, memo, parentMemo, memoMessage)
```

位置：`server/router/api/v1/memo_update_helpers.go:66-69`

```go
func (s *APIV1Service) dispatchMemoUpdatedSideEffects(ctx context.Context, memo *store.Memo, parentMemo *store.Memo, memoMessage *v1pb.Memo) {
    if err := s.DispatchMemoUpdatedWebhook(ctx, memoMessage); err != nil {
        slog.Warn("Failed to dispatch memo updated webhook", slog.Any("err", err))
    }
    // ...
}
```

**Webhook 发送**：
- 活动类型：`memos.memo.updated`
- 异步发送，失败只记录警告

---

### 5.5 SSE 广播

位置：`server/router/api/v1/memo_update_helpers.go:71-77`

```go
s.SSEHub.Broadcast(&SSEEvent{
    Type:       SSEEventMemoUpdated,
    Name:       memoMessage.Name,
    Parent:     memoMessage.GetParent(),
    Visibility: memo.Visibility,
    CreatorID:  resolveSSECreatorID(memo, parentMemo),
})
```

**SSE 事件类型**：`memo.updated`

---

### 5.6 前端缓存刷新

位置：`web/src/hooks/useLiveMemoRefresh.ts:382-388`

```typescript
case SSE_EVENT_TYPES.memoUpdated:
    queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
    queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
    if (event.parent) {
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
    }
    break;
```

---

### 5.7 归档完整时序图

```
前端                                    API                                   Store                             Webhook                         SSE
 │                                       │                                      │                                  │                              │
 │ 用户点击 Archive                       │                                      │                                  │                              │
 │ ───────────────────────────────────►  │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 1. 解析 UID                           │                                  │                              │
 │                                       │ 2. 校验 updateMask (必须有 "state")   │                                  │                              │
 │                                       │ 3. GetMemo                           │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 返回 memo (状态=NORMAL)         │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 4. 权限校验 (creator/admin)          │                                  │                              │
 │                                       │    失败 → 返回 PermissionDenied      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 5. 构建 UpdateMemo                   │                                  │                              │
 │                                       │    RowStatus = Archived              │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 6. UpdateMemo                        │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 更新 row_status=NORMAL→ARCHIVED  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 7. 重新获取 memo (最新状态)            │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 返回 memo (状态=ARCHIVED)       │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 8. buildUpdatedMemoState             │ ─────────────────────────────►   │                              │
 │                                       │    (reactions, attachments, relations)│                                 │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 9. dispatchMemoUpdatedSideEffects    │                                  │                              │
 │                                       │    ├─ Webhook (异步)                  │                                  │ ◄────────────────────────────  │
 │                                       │    │  ActivityType: memo.updated      │                                  │ PostAsync                    │
 │                                       │    │  失败只 Warn                      │                                  │                              │
 │                                       │    └─ SSE Broadcast                   │                                  │                              │
 │                                       │       Type: memo.updated              │                                  │                              │
 │                                       │ ──────────────────────────────────────────────────────────────────────► │
 │                                       │                                      │                                  │                              │
 │ ◄────────────────────────────────────────────────────────────────────────────────────────────────────────── │
 │ SSE memo.updated                       │                                      │                                  │                              │
 │ ────────────────────────────►         │                                      │                                  │                              │
 │ 1. invalidateQueries(detail)           │                                      │                                  │                              │
 │ 2. invalidateQueries(lists)            │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │ 导航到首页                             │                                      │                                  │                              │
 │ 显示 "Archived successfully" Toast    │                                      │                                  │                              │
```

---

## 6. 路径 3：归档后恢复

**路径**：ARCHIVED → NORMAL

### 6.1 前端触发

归档页面入口：

位置：`web/src/pages/Archived.tsx:19-22`

```typescript
const { listSort, orderBy } = useMemoSorting({
    pinnedFirst: true,
    state: State.ARCHIVED,  // 查询 ARCHIVED 状态
});
```

MemoActionMenu 在归档状态时显示 "Restore" 选项：

位置：`web/src/components/MemoActionMenu/MemoActionMenu.tsx:104-109`

```typescript
{!isComment && (
    <DropdownMenuItem onClick={handleToggleMemoStatusClick}>
        {isArchived ? <ArchiveRestoreIcon ... /> : <ArchiveIcon ... />}
        {isArchived ? t("common.restore") : t("common.archive")}
    </DropdownMenuItem>
)}
```

位置：`web/src/components/MemoActionMenu/hooks.ts:55-81`

```typescript
const handleToggleMemoStatusClick = useCallback(async () => {
    const isArchiving = memo.state !== State.ARCHIVED;  // false，因为当前是 ARCHIVED
    const state = memo.state === State.ARCHIVED ? State.NORMAL : State.ARCHIVED;  // NORMAL
    const message = memo.state === State.ARCHIVED 
        ? t("message.restored-successfully")  // 显示恢复成功
        : t("message.archived-successfully");

    await updateMemo({
        update: {
            name: memo.name,
            state,  // NORMAL
        },
        updateMask: ["state"],
    });
    toast.success(message);
    
    // 导航处理
    if (isInMemoDetailPage) {
        navigateTo(memo.state === State.ARCHIVED ? ROUTES.HOME : ROUTES.ARCHIVED);
        // 注意：memo.state 还是旧值（ARCHIVED），所以会跳转到 ROUTES.ARCHIVED
    }
}, [...]);
```

**前端触发流程**（与归档基本一致，只是方向相反）：
1. 判断当前状态（ARCHIVED → isArchived=true）
2. 目标状态 = NORMAL
3. 调用 `updateMemo`
4. 成功后：
   - 显示 "Restored successfully"
   - 如果在详情页，导航到归档页（注意：使用的是旧状态判断）
5. 失败后：
   - 显示错误 toast

---

### 6.2 API 校验

与归档路径完全相同，使用同一个 `UpdateMemo` 接口。

**关键差异**：
- `rowStatus` 目标值为 `NORMAL`
- 权限校验和其他逻辑完全一致

---

### 6.3 完整事件流程

与归档路径完全一致：
1. Store.UpdateMemo（ARCHIVED → NORMAL）
2. Webhook：`memos.memo.updated`（异步）
3. SSE：`memo.updated`（广播）
4. 前端缓存：失效 detail 和 lists

---

### 6.4 归档后恢复完整时序图

```
前端                                    API                                   Store                             Webhook                         SSE
 │                                       │                                      │                                  │                              │
 │ 用户在归档页点击 Restore                │                                      │                                  │                              │
 │ ───────────────────────────────────►  │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 1. 解析 UID                           │                                  │                              │
 │                                       │ 2. 校验 updateMask                    │                                  │                              │
 │                                       │ 3. GetMemo                           │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 返回 memo (状态=ARCHIVED)       │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 4. 权限校验 (creator/admin)          │                                  │                              │
 │                                       │    失败 → 返回 PermissionDenied      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 5. 构建 UpdateMemo                   │                                  │                              │
 │                                       │    RowStatus = Normal                │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 6. UpdateMemo                        │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 更新 row_status=ARCHIVED→NORMAL  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 7. 重新获取 memo (最新状态)            │ ─────────────────────────────►   │                              │
 │                                       │    ◄───────────────────────────────  │ 返回 memo (状态=NORMAL)         │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 8. buildUpdatedMemoState             │ ─────────────────────────────►   │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 9. dispatchMemoUpdatedSideEffects    │                                  │                              │
 │                                       │    ├─ Webhook (异步)                  │                                  │ ◄────────────────────────────  │
 │                                       │    │  ActivityType: memo.updated      │                                  │ PostAsync                    │
 │                                       │    │  失败只 Warn                      │                                  │                              │
 │                                       │    └─ SSE Broadcast                   │                                  │                              │
 │                                       │       Type: memo.updated              │                                  │                              │
 │                                       │ ──────────────────────────────────────────────────────────────────────► │
 │                                       │                                      │                                  │                              │
 │ ◄────────────────────────────────────────────────────────────────────────────────────────────────────────── │
 │ SSE memo.updated                       │                                      │                                  │                              │
 │ ────────────────────────────►         │                                      │                                  │                              │
 │ 1. invalidateQueries(detail)           │                                      │                                  │                              │
 │ 2. invalidateQueries(lists)            │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │ 导航到归档页 (使用旧状态判断)            │                                      │                                  │                              │
 │ 显示 "Restored successfully" Toast    │                                      │                                  │                              │
```

---

## 7. 路径 1（变体）：归档后删除

**路径**：ARCHIVED → 永久删除

### 7.1 与正常态直删的异同

**相同点**：
- 使用完全相同的 `DeleteMemo` 接口
- 完全相同的 API 校验逻辑
- 完全相同的 Store 清理流程
- 完全相同的 Webhook 发送
- 完全相同的 SSE 广播

**不同点**：
- 初始状态不同（ARCHIVED vs NORMAL）
- 但代码中不校验状态，所以逻辑完全一致
- 唯一差异：用户操作入口（归档页 vs 正常列表/详情页）

### 7.2 删除流程

与"路径 1：正常态直删"完全相同，详见第 4 节。

---

## 8. 三条路径对比

### 8.1 操作对比表

| 维度 | 正常态直删 | 归档 | 归档后恢复 | 归档后删除 |
|------|-----------|------|-----------|-----------|
| 前端触发 | Delete 菜单 | Archive 菜单 | Restore 菜单 | Delete 菜单（归档页） |
| API 接口 | DeleteMemo | UpdateMemo (state) | UpdateMemo (state) | DeleteMemo |
| 状态变更 | (删除) | NORMAL→ARCHIVED | ARCHIVED→NORMAL | (删除) |
| 权限校验 | creator/admin | creator/admin | creator/admin | creator/admin |
| 状态校验 | 无（任意状态可删） | 无（可反复切换） | 无（可反复切换） | 无（任意状态可删） |
| 级联清理 | 评论、relations、attachments | 无 | 无 | 评论、relations、attachments |
| 事务保护 | 无 | 无 | 无 | 无 |

### 8.2 事件对比表

| 维度 | 正常态直删 | 归档 | 归档后恢复 | 归档后删除 |
|------|-----------|------|-----------|-----------|
| Webhook 时机 | 删除前发送 | 更新后发送 | 更新后发送 | 删除前发送 |
| Webhook 类型 | memos.memo.deleted | memos.memo.updated | memos.memo.updated | memos.memo.deleted |
| Webhook 失败影响 | 无（仅 Warn） | 无（仅 Warn） | 无（仅 Warn） | 无（仅 Warn） |
| SSE 时机 | 删除后广播 | 更新后广播 | 更新后广播 | 删除后广播 |
| SSE 类型 | memo.deleted | memo.updated | memo.updated | memo.deleted |
| SSE 失败影响 | 无（静默丢弃） | 无（静默丢弃） | 无（静默丢弃） | 无（静默丢弃） |
| 缓存操作 | remove detail + invalidate lists/stats | invalidate detail + lists | invalidate detail + lists | remove detail + invalidate lists/stats |

---

## 9. 失败处理机制汇总

### 9.1 前端失败处理

位置：`web/src/components/MemoActionMenu/hooks.ts:69-74, 102-107`

```typescript
// 归档/恢复失败
try {
    await updateMemo({...});
    toast.success(message);
} catch (error: unknown) {
    handleError(error, toast.error, {
        context: `${isArchiving ? "Archive" : "Restore"} memo`,
        fallbackMessage: "An error occurred",
    });
    return;  // 失败则不执行导航和缓存操作
}

// 删除失败
try {
    await deleteMemo(memo.name);
} catch (error: unknown) {
    handleError(error, toast.error, { 
        context: "Delete memo", 
        fallbackMessage: "An error occurred" 
    });
    return;  // 失败则不执行导航和缓存操作
}
```

**前端失败策略**：
- 显示错误 Toast
- 中断后续操作（导航、缓存失效）
- 保持当前页面状态

---

### 9.2 API 层失败处理

位置：`server/router/api/v1/memo_service.go`

**UpdateMemo（归档/恢复）**：
```go
if err = s.Store.UpdateMemo(ctx, update); err != nil {
    return nil, status.Errorf(codes.Internal, "failed to update memo")
}
```

**DeleteMemo**：
```go
if err = s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: memo.ID}); err != nil {
    return nil, status.Errorf(codes.Internal, "failed to delete memo")
}
```

**API 层失败策略**：
- 参数错误 → InvalidArgument
- 资源不存在 → NotFound
- 未认证 → Unauthenticated
- 无权限 → PermissionDenied
- 内部错误 → Internal
- 所有错误都会中断操作，返回 gRPC 状态码

---

### 9.3 Store 层失败处理

**无事务保护**：删除和更新操作都是分步执行的：

```go
// DeleteMemo 中的分步操作
// 1. 删除 memo_relation（作为源）→ 失败则返回
// 2. 删除 memo_relation（作为目标）→ 失败则返回
// 3. 逐个删除 attachments → 任一失败则返回
// 4. 删除 memo 本体 → 失败则返回
```

**Store 层失败策略**：
- 每一步失败立即返回 error
- **已执行的步骤不会回滚**
- 可能导致部分数据已删除，部分数据残留

---

### 9.4 Webhook 失败处理

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

**Webhook 失败策略**：
- 异步发送，不阻塞主流程
- 队列满 → 丢弃，记录 Warn
- 发送超时（30秒）→ 记录 Warn
- **失败不会影响 memo 操作**
- 无重试机制

---

### 9.5 SSE 失败处理

位置：`server/router/api/v1/sse_hub.go:115-132`

```go
func (h *SSEHub) Broadcast(event *SSEEvent) {
    data := event.JSON()
    if len(data) == 0 {
        return  // JSON 序列化失败，静默忽略
    }
    // ...
    for c := range h.clients {
        select {
        case c.events <- data:
        default:
            // 慢客户端丢弃事件，不阻塞广播
        }
    }
}
```

**SSE 失败策略**：
- JSON 序列化失败 → 静默忽略
- 客户端缓冲满 → 丢弃，不阻塞
- **失败不会影响 memo 操作**
- 无重试机制（依赖前端重连后的重新查询）

---

## 10. 关键设计决策分析

### 10.1 状态管理设计

**两状态模型**（NORMAL / ARCHIVED）：

- 优点：简单直观
- 缺点：没有中间状态（如"删除中"）

**无状态转换校验**：

- 可以在任意状态间切换（甚至同状态切换）
- 允许：NORMAL→ARCHIVED→NORMAL→ARCHIVED（反复切换）
- 允许：NORMAL→删除（跳过归档）
- 允许：ARCHIVED→删除
- 设计意图：给用户最大灵活性

---

### 10.2 删除无事务保护

**当前实现**：分步执行，失败不回滚

**可能的问题**：
- 如果删除到一半失败，可能出现：
  - 评论已删除，主 memo 未删除
  - 部分 attachments 已删除
  - relations 已清理，memo 本体残留

**设计意图**：
- 简化实现
- 依赖应用层重试或人工清理

---

### 10.3 Webhook 异步设计

**当前实现**：
- 4 个 worker，队列容量 128
- 无重试，无持久化
- 失败只记录日志

**设计意图**：
- Webhook 通知是"尽力而为"（best-effort）
- 不影响核心 memo 操作的性能和可靠性
- 外部系统需要自行处理丢失的通知

---

### 10.4 SSE 最佳设计

**当前实现**：
- 慢客户端丢弃事件
- 前端重连后重新查询 active queries

**设计意图**：
- 保护服务器不被慢客户端阻塞
- 依赖 React Query 的重新查询机制保证最终一致性

---

## 11. 相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `store/common.go` | RowStatus 定义 |
| `store/memo.go` | Store 层 DeleteMemo/UpdateMemo 实现 |
| `server/router/api/v1/memo_service.go` | API 层 DeleteMemo/UpdateMemo 实现 |
| `server/router/api/v1/memo_update_helpers.go` | 更新副作用（Webhook+SSE） |
| `server/router/api/v1/sse_hub.go` | SSE Hub 实现 |
| `internal/webhook/webhook.go` | Webhook 异步实现 |
| `web/src/components/MemoActionMenu/MemoActionMenu.tsx` | 前端操作菜单 UI |
| `web/src/components/MemoActionMenu/hooks.ts` | 前端操作处理逻辑 |
| `web/src/hooks/useLiveMemoRefresh.ts` | 前端 SSE 监听和缓存处理 |
| `web/src/pages/Archived.tsx` | 归档页面 |
| `proto/api/v1/common.proto` | Proto State 定义 |
