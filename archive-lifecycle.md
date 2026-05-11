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

### 2.2 MemoRelation 关系模型

位置：`store/memo_relation.go:16-20`

```go
type MemoRelation struct {
    MemoID        int32            // 评论 memo 的 ID（源）
    RelatedMemoID int32            // 被评论 memo 的 ID（目标）
    Type          MemoRelationType // "COMMENT" 或 "REFERENCE"
}
```

**评论关系说明**：
- `MemoID`：评论 memo 本身（子）
- `RelatedMemoID`：被评论的 memo（父）
- 关系是单向的，没有嵌套关系记录

---

## 3. 评论判定机制详解

### 3.1 数据库层：ListMemos 的 ExcludeComments 逻辑

位置：`store/db/sqlite/memo.go:139-144`

```sql
SELECT ..., 
  CASE WHEN `parent_memo`.`uid` IS NOT NULL THEN `parent_memo`.`uid` ELSE NULL END AS `parent_uid`
FROM `memo`
  LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id`
  LEFT JOIN `memo_relation` 
      ON `memo`.`id` = `memo_relation`.`memo_id` 
      AND `memo_relation`.`type` = "COMMENT" 
  LEFT JOIN `memo` AS `parent_memo` 
      ON `memo_relation`.`related_memo_id` = `parent_memo`.`id`
WHERE ...
```

位置：`store/db/sqlite/memo.go:104-106`

```go
if find.ExcludeComments {
    where = append(where, "`parent_uid` IS NULL")
}
```

**关键理解**：

1. **评论的判定方式**：通过 `memo_relation` 表中是否存在 `type = "COMMENT"` 的关系来判定
   - 如果 memo 在 `memo_relation` 表中作为 `memo_id` 存在，且 `type = "COMMENT"` → 这是一条评论
   - `parent_uid` 字段是通过 LEFT JOIN 计算出来的，不是 memo 表的固有字段

2. **ExcludeComments 的过滤条件**：`parent_uid IS NULL`
   - 只有当 `memo_relation.memo_id = memo.id` 且 `type = "COMMENT"` 时，`parent_uid` 才有值
   - 如果 `parent_uid IS NULL` → 不是评论（或关系已被删除）

---

### 3.2 API 层：ListMemos 默认排除评论

位置：`server/router/api/v1/memo_service.go:189-193`

```go
func (s *APIV1Service) ListMemos(ctx context.Context, request *v1pb.ListMemosRequest) (*v1pb.ListMemosResponse, error) {
    memoFind := &store.FindMemo{
        ExcludeComments: true,  // 默认排除评论
    }
    // ...
}
```

**默认行为**：
- 正常 memo 列表查询默认 `ExcludeComments = true`
- 只有通过 `ListMemoRelations` 或专门的评论查询才会获取评论

---

### 3.3 前端层：isComment 判定

位置：`web/src/components/MemoActionMenu/MemoActionMenu.tsx:38`

```typescript
const isComment = Boolean(memo.parent);
```

位置：`web/src/components/MemoDetailSidebar/MemoDetailSidebar.tsx:51`

```typescript
const canManageShares = !memo.parent && (...);
```

**前端判定依据**：
- `memo.parent` 字段来自 API 返回的 `Memo.parent`
- 如果 `parent` 有值 → 是评论
- 如果 `parent` 为空 → 不是评论

---

### 3.4 评论判定完整链路

```
数据库层                    API 层                      前端层
    │                          │                          │
    │ 1. LEFT JOIN memo_relation│                          │
    │    WHERE type = "COMMENT"│                          │
    │                          │                          │
    │ 2. 计算 parent_uid:       │                          │
    │    - 有关系 → parent_uid  │                          │
    │      = 被评论 memo 的 UID │                          │
    │    - 无关系 → parent_uid  │                          │
    │      = NULL               │                          │
    │                          │                          │
    │ 3. ExcludeComments=true   │                          │
    │    → WHERE parent_uid    │                          │
    │      IS NULL              │                          │
    │                          │                          │
    │ ────────────────────────► │                          │
    │   返回 memo 列表          │                          │
    │   parent_uid 非空的被过滤  │                          │
    │                          │                          │
    │                          │ 4. loadMemoRelations      │
    │                          │    构建 Memo.parent       │
    │                          │    (从 relations 中       │
    │                          │    提取 COMMENT 类型)     │
    │                          │                          │
    │                          │ ───────────────────────► │
    │                          │   返回 Memo 列表         │
    │                          │   包含 parent 字段       │
    │                          │                          │
    │                          │                          │ 5. isComment =         │
    │                          │                          │    Boolean(memo.parent)│
    │                          │                          │    - parent 有值 → 评论 │
    │                          │                          │    - parent 为空 → 非评 │
    │                          │                          │                         │
```

---

## 4. 三条操作路径总览

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

## 5. 路径 1：正常态直删（硬删除）

**路径**：NORMAL → 永久删除

### 5.1 前端触发

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

### 5.2 API 校验

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

### 5.3 Webhook 发送（删除前）

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

---

### 5.4 Store 清理（含评论处理逻辑）

位置：`server/router/api/v1/memo_service.go:626-641`

```go
// 步骤 7: 列出并删除直接评论（仅直接子评论，非递归）
commentType := store.MemoRelationComment
relations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{
    RelatedMemoID: &memo.ID,  // 查找所有 related_memo_id = 父 memo ID 的关系
    Type: &commentType,        // 类型为 COMMENT
})
if err != nil {
    return nil, status.Errorf(codes.Internal, "failed to list memo comments")
}
for _, relation := range relations {
    // 对每个直接评论调用 Store.DeleteMemo
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
    // 删除所有 memo_id = delete.ID 的关系（包括该 memo 作为评论的关系）
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    // 步骤 8.2: 清理 memo_relation 记录（作为目标）
    // 删除所有 related_memo_id = delete.ID 的关系（包括该 memo 被评论的关系）
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

---

### 5.5 评论删除的关键分析

#### 5.5.1 仅删除直接评论，非递归

**代码分析**：

位置：`server/router/api/v1/memo_service.go:626-636`

```go
commentType := store.MemoRelationComment
relations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{
    RelatedMemoID: &memo.ID,  // 只查找 related_memo_id = 父 memo ID
    Type: &commentType,
})
// ...
for _, relation := range relations {
    // 对每个直接评论调用 Store.DeleteMemo
    if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
        return nil, status.Errorf(codes.Internal, "failed to delete memo comment")
    }
}
```

**为什么不是递归？**

1. **查询条件限制**：`ListMemoRelations` 只查询 `RelatedMemoID = memo.ID` 的关系
   - 这只会返回**直接评论**（子）
   - 不会返回评论的评论（孙）

2. **关系模型设计**：`MemoRelation` 只有一层关系
   - 没有 `parent_id` 链式结构
   - 没有嵌套关系的层级记录

3. **实现简化**：
   - 评论是独立的 memo，通过 `MemoRelation` 关联到父 memo
   - 删除父 memo 时，只删除一层直接关联的评论
   - 评论的评论（多级）需要单独处理

---

#### 5.5.2 子评论走 Store.DeleteMemo 的链路分析

**链路对比**：

| 链路 | 调用入口 | Webhook | SSE |
|------|---------|---------|-----|
| **父 memo 删除** | API 层 `DeleteMemo` 方法 | ✅ 发送 `memos.memo.deleted` | ✅ 广播 `memo.deleted` |
| **直接子评论删除** | Store 层 `Store.DeleteMemo` 方法 | ❌ **不发送** | ❌ **不广播** |

**详细分析**：

```
API.DeleteMemo (父 memo)
    │
    ├── 收集 reactions/attachments/relations
    │
    ├── ✅ DispatchMemoDeletedWebhook (仅父 memo)
    │
    ├── 循环删除直接评论:
    │   │
    │   └── Store.DeleteMemo (子评论)  ← 直接调用 Store 层，绕过 API 层
    │           │
    │           ├── DeleteMemoRelation (清理关系)
    │           ├── DeleteAttachment (清理附件)
    │           └── driver.DeleteMemo (删除 memo 本体)
    │           │
    │           └── ❌ 无 Webhook 调用
    │           └── ❌ 无 SSE 广播
    │
    └── Store.DeleteMemo (父 memo 本体)
    │
    └── ✅ SSEHub.Broadcast (memo.deleted)
```

**根本原因**：

位置：`server/router/api/v1/memo_service.go:632-636`

```go
// API 层删除父 memo 时，对子评论直接调用 Store.DeleteMemo
for _, relation := range relations {
    // 直接调用 Store 层，跳过 API 层的事件分发逻辑
    if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
        return nil, status.Errorf(codes.Internal, "failed to delete memo comment")
    }
}
```

Webhook 和 SSE 的触发逻辑在 API 层，不在 Store 层：

```go
// Webhook 和 SSE 只在 API 层调用
// API.DeleteMemo 中：
if err := s.DispatchMemoDeletedWebhook(ctx, memoMessage); err != nil { ... }
// ...
s.SSEHub.Broadcast(&SSEEvent{Type: SSEEventMemoDeleted, ...})

// 但子评论走的是 Store.DeleteMemo，不会经过这些逻辑
```

**影响范围**：

| 事件类型 | 父 memo | 直接子评论 | 评论的评论（多级） |
|---------|--------|-----------|------------------|
| Webhook | ✅ | ❌ | ❌ |
| SSE | ✅ | ❌ | ❌ |
| 前端缓存失效 | ✅（移除 detail + 失效 lists） | ❌（依赖父 memo 的 lists 失效间接清理） | ❌ |

**前端的补偿机制**：

位置：`web/src/hooks/useLiveMemoRefresh.ts:390-394`

```typescript
case SSE_EVENT_TYPES.memoDeleted:
    queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
    queryClient.invalidateQueries({ queryKey: memoKeys.lists() });  // ← 这个会重新查询所有列表
    queryClient.invalidateQueries({ queryKey: userKeys.stats() });
    break;
```

父 memo 删除后，前端会**重新查询所有列表**，间接地让子评论从 UI 中消失（因为数据库中已经不存在了）。

---

#### 5.5.3 多级评论关系的删除残留风险与可见性分析

**Store.DeleteMemo 的关系清理规则（关键理解）**：

位置：`store/memo.go:140-159`

```go
func (s *Store) DeleteMemo(ctx context.Context, delete *DeleteMemo) error {
    // 规则 1：删除该 memo 作为源的关系（memo_id = delete.ID）
    //    → 删除"我评论别人"的关系
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    // 规则 2：删除该 memo 作为目标的关系（related_memo_id = delete.ID）
    //    → 删除"别人评论我"的关系
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{RelatedMemoID: &delete.ID}); err != nil {
        return err
    }
    // 清理 attachments，删除 memo 本体
    return s.driver.DeleteMemo(ctx, delete)
}
```

**场景假设（四级评论）**：

```
Memo A (主 memo, 正常状态)
    ├── Memo B (评论 A)  ← 直接评论，relation = (B, A, COMMENT)
    │       └── Memo C (评论 B)  ← 评论的评论，relation = (C, B, COMMENT)
    │               └── Memo D (评论 C)  ← 评论的评论的评论，relation = (D, C, COMMENT)
    │
    └── Memo E (评论 A)  ← 直接评论，relation = (E, A, COMMENT)
```

**初始关系表**：

| memo_id | related_memo_id | type | 含义 |
|---------|----------------|------|------|
| B | A | COMMENT | B 评论 A |
| C | B | COMMENT | C 评论 B |
| D | C | COMMENT | D 评论 C |
| E | A | COMMENT | E 评论 A |

---

**删除 Memo A 时的实际执行推演**：

**步骤 1**：API.DeleteMemo(A) 列出直接评论
- 查询 `ListMemoRelations(RelatedMemoID = A, Type = COMMENT)`
- 找到关系 (B, A, COMMENT) 和 (E, A, COMMENT)
- 对应直接评论：B 和 E

**步骤 2**：调用 `Store.DeleteMemo(B)`（删除直接评论 B）

根据清理规则：
- 规则 1：删除 `memo_id = B` 的关系 → 删除 **(B, A, COMMENT)**（B 评论 A）
- 规则 2：删除 `related_memo_id = B` 的关系 → 删除 **(C, B, COMMENT)**（C 评论 B）⚠️
- 删除 B 的 attachments
- 删除 B 本体

**此时状态**：
- B 已删除
- 关系 (B, A, COMMENT) 已删
- 关系 (C, B, COMMENT) **已被删除**（因为 B 作为目标）
- **但 C 本体未被删除！** API 层只处理直接评论 B，不会递归处理 C

**步骤 3**：调用 `Store.DeleteMemo(E)`（删除直接评论 E）

根据清理规则：
- 规则 1：删除 `memo_id = E` 的关系 → 删除 **(E, A, COMMENT)**（E 评论 A）
- 规则 2：删除 `related_memo_id = E` 的关系 → 无（没有 memo 评论 E）
- 删除 E 的 attachments
- 删除 E 本体

**此时状态**：
- E 已删除
- 关系 (E, A, COMMENT) 已删

**步骤 4**：调用 `Store.DeleteMemo(A)`（删除主 memo A）

根据清理规则：
- 规则 1：删除 `memo_id = A` 的关系 → 无（A 没有评论其他 memo）
- 规则 2：删除 `related_memo_id = A` 的关系 → (B,A) 和 (E,A) 已被 B、E 的删除清理
- 删除 A 的 attachments
- 删除 A 本体

---

**最终状态汇总**：

| Memo | 本体状态 | 原因 |
|------|---------|------|
| A | ✅ 已删除 | 主 memo，被 API.DeleteMemo(A) 处理 |
| B | ✅ 已删除 | 直接评论，被 Store.DeleteMemo(B) 处理 |
| C | ❌ **残留** | 评论的评论，API 层不递归处理，未被调用 Store.DeleteMemo(C) |
| D | ❌ **残留** | 更深层级评论，同上 |
| E | ✅ 已删除 | 直接评论，被 Store.DeleteMemo(E) 处理 |

**最终关系表**：

| memo_id | related_memo_id | type | 是否保留 | 原因 |
|---------|----------------|------|---------|------|
| (B, A, COMMENT) | - | - | ❌ 已删 | Store.DeleteMemo(B) 规则 1 |
| (C, B, COMMENT) | - | - | ❌ 已删 | Store.DeleteMemo(B) 规则 2（B 作为目标） |
| (D, C, COMMENT) | - | - | ✅ **保留** | C 未被删除，没有人清理这个关系 |
| (E, A, COMMENT) | - | - | ❌ 已删 | Store.DeleteMemo(E) 规则 1 |

**关键差异**：
- **C 的关系 (C, B, COMMENT)**：已删除，因为 B 被删除时触发了规则 2（清理"别人评论我"的关系）
- **D 的关系 (D, C, COMMENT)**：**保留**，因为 C 没有被删除，没人触发规则 2 来清理这个关系

---

#### 5.5.4 残留 memo C 的可见性分析

**删除后的状态**：

| Memo | memo 本体是否删除 | memo_relation 是否删除 | parent_uid 计算结果 |
|------|-----------------|---------------------|-------------------|
| A | ✅ 已删除 | ✅ 已删除 | (已删除) |
| B | ✅ 已删除 | ✅ (B,A) 已删除 | (已删除) |
| C | ❌ **残留** | ✅ (C,B) 已删除 | **NULL**（因为关系已删） |
| D | ✅ 已删除 | ✅ (D,A) 已删除 | (已删除) |

**关键结论**：Memo C 的 `parent_uid = NULL`

这意味着什么？回顾第 3 节的评论判定机制：

```sql
-- ListMemos 的查询逻辑
SELECT ..., 
  CASE WHEN `parent_memo`.`uid` IS NOT NULL 
       THEN `parent_memo`.`uid` 
       ELSE NULL END AS `parent_uid`
FROM `memo`
  LEFT JOIN `memo_relation` 
      ON `memo`.`id` = `memo_relation`.`memo_id` 
      AND `memo_relation`.`type` = "COMMENT" 
  LEFT JOIN `memo` AS `parent_memo` 
      ON `memo_relation`.`related_memo_id` = `parent_memo`.`id`
WHERE ...
  AND `parent_uid` IS NULL  -- ExcludeComments = true 时的条件
```

**由于 Memo C 的 `memo_relation` 记录已被删除**：
- LEFT JOIN `memo_relation` 时找不到匹配
- `parent_uid = NULL`
- **满足 `parent_uid IS NULL` 的条件**
- **会被 `ListMemos(ExcludeComments=true)` 返回！**

---

#### 5.5.5 残留 memo C 的影响范围（修正后的精确结论）

**前提条件**：
- Memo C 原本是 Memo B 的评论
- 删除 Memo A 时，Memo B 被删除，C→B 的关系被清理
- Memo C 本体残留，但关系已丢失

**数据库状态**：

| 字段 | 值 |
|------|-----|
| memo.id | C 的 ID |
| memo.row_status | NORMAL（或原状态） |
| memo_relation 中 memo_id = C 的记录 | ❌ 已删除 |
| memo_relation 中 related_memo_id = C 的记录 | 可能有（如果 C 有自己的评论） |

**可见性分析**：

| 查询方式 | 是否可见 | 原因 |
|---------|---------|------|
| `ListMemos(ExcludeComments=true)`（默认列表） | **✅ 可见** | `parent_uid = NULL`，满足过滤条件 |
| `ListMemoRelations`（专门查评论关系） | ❌ 不可见 | 关系已被删除 |
| 直接访问 `/memos/:uid` | **✅ 可见** | memo 本体还在，GetMemo 不查关系 |
| 用户统计 `GetUserStats` | **✅ 被计入** | 使用 `ExcludeComments=true`，C 满足条件 |
| RSS 订阅 | **✅ 可见** | 使用 `ExcludeComments=true` |
| MCP 工具查询 | **✅ 可见** | 使用 `ExcludeComments=true` |

**用户体验影响**：

1. **残留 memo 突然出现在正常列表中**：
   - 用户删除 Memo A 后，原本的评论 Memo C 可能出现在首页、时间线等正常 memo 列表中
   - 因为 C 不再被识别为评论（关系已丢失）

2. **统计数据异常**：
   - `TotalMemoCount` 会包含残留的评论 memo
   - `TagCount`、`MemoTypeStats`（LinkCount、CodeCount、TodoCount、UndoCount）都会计入
   - `MemoCreatedTimestamps`、`MemoUpdatedTimestamps` 也会包含

3. **内容暴露风险**：
   - 如果 Memo C 原本是评论（可能是对敏感内容的讨论），现在变成"正常 memo"
   - 可能会被公开访问（如果 visibility = PUBLIC/PROTECTED）
   - 可能被其他用户在正常列表中看到

**精确的前提条件总结**：

| 条件 | 说明 |
|------|------|
| 必须存在多级评论 | 至少三级：A → B → C |
| 被删除的是顶层 memo | 删除 A 时，B 被删除，C 的关系被清理但本体残留 |
| 残留 memo 的关系被清理 | Store.DeleteMemo(B) 时删除了 (C, B, COMMENT) 关系 |
| 残留 memo 本身未被调用 Store.DeleteMemo | API 层只处理直接评论 B 和 D，不递归处理 C |
| ListMemos 使用 ExcludeComments=true | 默认行为，parent_uid=NULL 时满足条件 |

---

#### 5.5.6 Store 清理顺序及失败处理

| 步骤 | 操作内容 | 失败时 | 失败影响 |
|------|----------|--------|----------|
| 7 | 列出直接评论 relations | Internal | 中断删除 |
| 7.1 | 对每个直接评论调用 `Store.DeleteMemo` | Internal | 中断删除，已删除的评论不会回滚 |
| 8.1 | 删除作为源的 memo_relation | error | 中断删除 |
| 8.2 | 删除作为目标的 memo_relation | error | 中断删除 |
| 8.3 | 列出 attachments | error | 中断删除 |
| 8.4 | 逐个删除 attachment | error | 中断删除，已删除的不会回滚 |
| 8.5 | 删除 memo 本体 | error | 中断删除 |

**重要说明**：
- 删除操作**没有事务保护**，是分步执行的
- 如果中途失败，已删除的数据**不会回滚**
- 多级评论存在**残留风险**（评论的评论不会被删除）
- **残留 memo 可能被当成正常 memo 返回**（因为关系被清理，parent_uid=NULL）

---

### 5.6 SSE 广播（删除后）

位置：`server/router/api/v1/memo_service.go:643-649`

```go
// 步骤 9: 广播删除事件（仅父 memo，子评论不广播）
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

**SSE 失败处理**：
- JSON 序列化失败：静默忽略
- 客户端缓冲满：丢弃事件，不阻塞
- 权限不匹配：不发送

---

### 5.7 前端缓存刷新

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

**注意**：这个失效只会让前端重新查询，但残留的 memo C 会在重新查询时被当作正常 memo 返回。

---

### 5.8 正常态直删完整时序图

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
 │                                       │ 6. 级联删除直接评论（非递归）          │                                  │                              │
 │                                       │    ├─ ListMemoRelations              │ ─────────────────────────────►   │                              │
 │                                       │    │  (RelatedMemoID=父ID, Type=COMMENT)│                                  │                              │
 │                                       │    │  ◄──────────────────────────────  │ 返回直接评论 B, D              │                              │
 │                                       │    │                                  │                                  │                              │
 │                                       │    ├─ Store.DeleteMemo(B)            │ ─────────────────────────────►   │                              │
 │                                       │    │  ├─ DeleteMemoRelation (源)       │                                  │                              │
 │                                       │    │  ├─ DeleteMemoRelation (目标)     │                                  │                              │
 │                                       │    │  │  ⚠️ 删除 C→B 的关系            │                                  │                              │
 │                                       │    │  ├─ DeleteAttachments             │                                  │                              │
 │                                       │    │  └─ DeleteMemo 本体               │                                  │                              │
 │                                       │    │  ❌ 无 Webhook                    │                                  │                              │
 │                                       │    │  ❌ 无 SSE                        │                                  │                              │
 │                                       │    │  ⚠️ Memo C 本体残留               │                                  │                              │
 │                                       │    │  ⚠️ Memo C 的关系被清理           │                                  │                              │
 │                                       │    │  ⚠️ Memo C 变成"正常 memo"        │                                  │                              │
 │                                       │    │                                  │                                  │                              │
 │                                       │    └─ Store.DeleteMemo(D)            │ ─────────────────────────────►   │                              │
 │                                       │       同上                           │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 7. DeleteMemo 父本体                 │                                  │                              │
 │                                       │    ──────────────────────────────►   │                                  │                              │
 │                                       │                                      │ 7.1 DeleteMemoRelation (源)    │                              │
 │                                       │                                      │ 7.2 DeleteMemoRelation (目标)  │                              │
 │                                       │                                      │ 7.3 DeleteAttachments          │                              │
 │                                       │                                      │ 7.4 DeleteMemo 本体             │                              │
 │                                       │    ◄───────────────────────────────   │                                  │                              │
 │                                       │                                      │                                  │                              │
 │                                       │ 8. Broadcast SSE                     │                                  │                              │
 │                                       │    Type: memo.deleted                │                                  │                              │
 │                                       │    (仅父 memo)                       │                                  │                              │
 │                                       │ ──────────────────────────────────────────────────────────────────────► │
 │                                       │                                      │                                  │                              │
 │ ◄────────────────────────────────────────────────────────────────────────────────────────────────────────── │
 │ SSE memo.deleted (父 memo)             │                                      │                                  │                              │
 │ ────────────────────────────►         │                                      │                                  │                              │
 │ 1. removeQueries(detail)              │                                      │                                  │                              │
 │ 2. invalidateQueries(lists)           │                                      │                                  │                              │
 │ 3. invalidateQueries(stats)           │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │ ⚠️ 重新查询列表时，Memo C 会被返回       │                                      │                                  │                              │
 │    因为 C 的关系已清理，parent_uid=NULL │                                      │                                  │                              │
 │                                       │                                      │                                  │                              │
 │ 导航到首页                             │                                      │                                  │                              │
 │ 显示成功 Toast                         │                                      │                                  │                              │
```

---

## 6. 路径 2：归档（软删除）

**路径**：NORMAL → ARCHIVED

### 6.1 前端触发

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

### 6.2 API 校验

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

### 6.3 Store 更新

位置：`store/memo.go:133-138`

```go
func (s *Store) UpdateMemo(ctx context.Context, update *UpdateMemo) error {
    if update.UID != nil && !base.UIDMatcher.MatchString(*update.UID) {
        return errors.New("invalid uid")
    }
    return s.driver.UpdateMemo(ctx, update)
}
```

**Store 更新失败处理**：
- 数据库层面的更新失败会返回 error
- 三种驱动实现类似：MySQL、PostgreSQL 都有对应实现

---

### 6.4 Webhook 发送

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

### 6.5 SSE 广播

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

### 6.6 前端缓存刷新

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

### 6.7 归档完整时序图

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

## 7. 路径 3：归档后恢复

**路径**：ARCHIVED → NORMAL

### 7.1 前端触发

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

### 7.2 API 校验

与归档路径完全相同，使用同一个 `UpdateMemo` 接口。

**关键差异**：
- `rowStatus` 目标值为 `NORMAL`
- 权限校验和其他逻辑完全一致

---

### 7.3 完整事件流程

与归档路径完全一致：
1. Store.UpdateMemo（ARCHIVED → NORMAL）
2. Webhook：`memos.memo.updated`（异步）
3. SSE：`memo.updated`（广播）
4. 前端缓存：失效 detail 和 lists

---

### 7.4 归档后恢复完整时序图

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

## 8. 路径 1（变体）：归档后删除

**路径**：ARCHIVED → 永久删除

### 8.1 与正常态直删的异同

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

### 8.2 删除流程

与"路径 1：正常态直删"完全相同，详见第 5 节。

**注意**：
- 归档 memo 的评论删除逻辑与正常 memo 相同
- 同样存在多级评论残留风险
- 同样不会触发子评论的 Webhook 和 SSE
- 同样可能将残留 memo 当成正常 memo 返回

---

## 9. 评论删除链路深度分析

### 9.1 评论关系模型

位置：`store/memo_relation.go:16-20`

```go
type MemoRelation struct {
    MemoID        int32            // 评论 memo ID（子）
    RelatedMemoID int32            // 被评论 memo ID（父）
    Type          MemoRelationType // "COMMENT"
}
```

**关系示例**：

```
Memo A (父)
  ├── Memo B (评论 A): relation = (B, A, COMMENT)
  │     └── Memo C (评论 B): relation = (C, B, COMMENT)
  │
  └── Memo D (评论 A): relation = (D, A, COMMENT)
```

---

### 9.2 删除父 memo 时的执行路径

```
API.DeleteMemo(A)
    │
    ├── ✅ 收集 A 的 reactions/attachments/relations
    │
    ├── ✅ DispatchMemoDeletedWebhook(A)
    │
    ├── ListMemoRelations(RelatedMemoID=A, Type=COMMENT)
    │   → 返回 [B, D]
    │
    ├── Store.DeleteMemo(B)  ← 直接调用 Store 层，绕过 API 层
    │   │
    │   ├── DeleteMemoRelation(MemoID=B)    → 删除 (B, A, COMMENT)
    │   ├── DeleteMemoRelation(RelatedMemoID=B) → 删除 (C, B, COMMENT)
    │   ├── DeleteAttachments(B)
    │   ├── driver.DeleteMemo(B)
    │   │
    │   └── ❌ 没有 DispatchMemoDeletedWebhook(B)
    │   └── ❌ 没有 SSEHub.Broadcast(memo.deleted, B)
    │   └── ⚠️ Memo C 本体残留（只删除了关系，没删 memo）
    │   └── ⚠️ Memo C 的 parent_uid = NULL（被当成正常 memo）
    │
    ├── Store.DeleteMemo(D)
    │   └── 同上，D 被删除，没有事件
    │
    ├── Store.DeleteMemo(A)
    │
    └── ✅ SSEHub.Broadcast(memo.deleted, A)
```

---

### 9.3 事件传播对比表

| 项目 | 父 memo（API 层） | 直接子评论（Store 层） | 评论的评论（多级） |
|------|-----------------|---------------------|------------------|
| **调用入口** | `API.DeleteMemo` | `Store.DeleteMemo` | 不处理 |
| **Webhook** | ✅ `memos.memo.deleted` | ❌ 无 | ❌ 无 |
| **SSE** | ✅ `memo.deleted` | ❌ 无 | ❌ 无 |
| **memo 本体** | ✅ 被删除 | ✅ 被删除 | ❌ 残留 |
| **relations** | ✅ 被清理 | ✅ 被清理 | ✅ 关系被清理（但 memo 残留） |
| **attachments** | ✅ 被清理 | ✅ 被清理 | ❌ 残留 |
| **parent_uid** | (已删除) | (已删除) | **NULL（被当成正常 memo）** |

---

### 9.4 多级评论残留场景（修正后的精确分析）

**前提**：存在三级评论结构

```
A (主 memo, row_status = NORMAL)
  ├── B (评论 A, 有 relation (B, A, COMMENT))
  │     └── C (评论 B, 有 relation (C, B, COMMENT))
  │           └── D (评论 C, 有 relation (D, C, COMMENT))
  └── E (评论 A, 有 relation (E, A, COMMENT))
```

**关系表**：

| memo_id | related_memo_id | type |
|---------|----------------|------|
| B | A | COMMENT |
| C | B | COMMENT |
| D | C | COMMENT |
| E | A | COMMENT |

**删除 A 后的状态**：

| memo | 本体是否删除 | 关系是否删除 | parent_uid | 被 ListMemos(ExcludeComments=true) 返回 |
|------|-------------|-------------|-----------|--------------------------------------|
| A | ✅ 已删除 | ✅ 已删除 | (已删除) | 否 |
| B | ✅ 已删除 | ✅ 已删除 | (已删除) | 否 |
| C | ❌ 残留 | ✅ (C,B) 已删除 | **NULL** | **是** |
| D | ❌ 残留 | ✅ (D,C) 已删除 | **NULL** | **是** |
| E | ✅ 已删除 | ✅ 已删除 | (已删除) | 否 |

**残留 memo C 和 D 的特征**：
- `row_status` 保持原值（NORMAL 或 ARCHIVED）
- `memo_relation` 表中没有任何关联（因为 B、C 被删除时清理了关系）
- **`parent_uid = NULL`**（因为 LEFT JOIN memo_relation 找不到匹配）
- **会被 `ListMemos(ExcludeComments=true)` 返回**（满足 `parent_uid IS NULL`）
- 会被统计数据计入（TotalMemoCount、TagCount、MemoTypeStats 等）
- 会出现在 RSS 订阅、MCP 工具查询中
- 会被其他用户看到（如果 visibility = PUBLIC/PROTECTED）

---

## 10. 三条路径对比

### 10.1 操作对比表

| 维度 | 正常态直删 | 归档 | 归档后恢复 | 归档后删除 |
|------|-----------|------|-----------|-----------|
| 前端触发 | Delete 菜单 | Archive 菜单 | Restore 菜单 | Delete 菜单（归档页） |
| API 接口 | DeleteMemo | UpdateMemo (state) | UpdateMemo (state) | DeleteMemo |
| 状态变更 | (删除) | NORMAL→ARCHIVED | ARCHIVED→NORMAL | (删除) |
| 权限校验 | creator/admin | creator/admin | creator/admin | creator/admin |
| 状态校验 | 无（任意状态可删） | 无（可反复切换） | 无（可反复切换） | 无（任意状态可删） |
| 直接评论处理 | 调用 Store.DeleteMemo | 无 | 无 | 调用 Store.DeleteMemo |
| 多级评论处理 | ❌ 不处理，残留 | 无 | 无 | ❌ 不处理，残留 |
| 子评论 Webhook | ❌ 不触发 | 无 | 无 | ❌ 不触发 |
| 子评论 SSE | ❌ 不广播 | 无 | 无 | ❌ 不广播 |
| 事务保护 | 无 | 无 | 无 | 无 |
| 残留 memo 可见性 | **会被当成正常 memo 返回** | 无 | 无 | **会被当成正常 memo 返回** |

---

### 10.2 事件对比表

| 维度 | 正常态直删 | 归档 | 归档后恢复 | 归档后删除 |
|------|-----------|------|-----------|-----------|
| Webhook 时机 | 删除前发送 | 更新后发送 | 更新后发送 | 删除前发送 |
| Webhook 类型 | memos.memo.deleted | memos.memo.updated | memos.memo.updated | memos.memo.deleted |
| Webhook 覆盖 | 仅父 memo | 父 memo | 父 memo | 仅父 memo |
| Webhook 失败影响 | 无（仅 Warn） | 无（仅 Warn） | 无（仅 Warn） | 无（仅 Warn） |
| SSE 时机 | 删除后广播 | 更新后广播 | 更新后广播 | 删除后广播 |
| SSE 类型 | memo.deleted | memo.updated | memo.updated | memo.deleted |
| SSE 覆盖 | 仅父 memo | 父 memo | 父 memo | 仅父 memo |
| SSE 失败影响 | 无（静默丢弃） | 无（静默丢弃） | 无（静默丢弃） | 无（静默丢弃） |
| 缓存操作 | remove detail + invalidate lists/stats | invalidate detail + lists | invalidate detail + lists | remove detail + invalidate lists/stats |

---

## 11. 失败处理机制汇总

### 11.1 前端失败处理

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

### 11.2 API 层失败处理

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

### 11.3 Store 层失败处理

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
- **多级评论存在残留风险**
- **残留 memo 可能被当成正常 memo 返回**

---

### 11.4 Webhook 失败处理

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
- **子评论删除不触发 Webhook**

---

### 11.5 SSE 失败处理

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
- **子评论删除不触发 SSE**

---

## 12. 关键设计决策分析

### 12.1 状态管理设计

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

### 12.2 评论删除非递归设计

**当前实现**：
- 只删除直接评论（RelatedMemoID = 父 ID）
- 通过调用 `Store.DeleteMemo` 删除直接评论
- 评论的评论（多级）不会被递归处理

**为什么不是递归？**

1. **关系模型限制**：
   - 关系是扁平的，没有层级结构
   - 递归需要多次数据库查询，性能较差

2. **实现简化**：
   - 评论本质上是独立的 memo
   - 简化实现，避免复杂的递归逻辑

3. **潜在风险**：
   - 多级评论会变成孤立 memo 残留
   - **残留 memo 的关系被清理，parent_uid=NULL，会被当成正常 memo 返回**

**设计意图**：
- 假设多级评论场景少见
- 接受残留风险，换取实现简洁

---

### 12.3 子评论走 Store.DeleteMemo 绕过事件分发

**当前实现**：

```go
// API.DeleteMemo 中删除子评论
for _, relation := range relations {
    // 直接调用 Store 层，绕过 API 层的事件分发
    if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
        return nil, status.Errorf(codes.Internal, "failed to delete memo comment")
    }
}
```

**为什么这样设计？**

1. **性能考虑**：
   - 避免为每个子评论都发送 Webhook 和 SSE
   - 父 memo 的删除事件已经足够通知前端

2. **事件风暴避免**：
   - 如果有 100 条评论，会产生 101 个事件
   - 前端通过失效 lists 缓存间接清理评论

3. **前端补偿**：
   ```typescript
   case SSE_EVENT_TYPES.memoDeleted:
       queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
       queryClient.invalidateQueries({ queryKey: memoKeys.lists() });  // 重新查询所有列表
       break;
   ```

**设计意图**：
- 父 memo 的删除事件已经足够
- 子评论的事件会被父事件的"重新查询列表"间接处理
- 减少不必要的事件广播

**但 Webhook 外部系统不会收到子评论删除通知**——这是一个潜在问题。

---

### 12.4 删除无事务保护

**当前实现**：分步执行，失败不回滚

**可能的问题**：
- 如果删除到一半失败，可能出现：
  - 评论已删除，主 memo 未删除
  - 部分 attachments 已删除
  - relations 已清理，memo 本体残留
  - **多级评论残留，且被当成正常 memo 返回**

**设计意图**：
- 简化实现
- 依赖应用层重试或人工清理

---

### 12.5 Webhook 异步设计

**当前实现**：
- 4 个 worker，队列容量 128
- 无重试，无持久化
- 失败只记录日志

**设计意图**：
- Webhook 通知是"尽力而为"（best-effort）
- 不影响核心 memo 操作的性能和可靠性
- 外部系统需要自行处理丢失的通知
- **子评论删除不发送 Webhook**（设计选择，但可能是遗漏）

---

### 12.6 SSE 静默丢弃设计

**当前实现**：
- 慢客户端丢弃事件
- 前端重连后重新查询 active queries

**设计意图**：
- 保护服务器不被慢客户端阻塞
- 依赖 React Query 的重新查询机制保证最终一致性
- **子评论删除不发送 SSE**（依赖父事件的列表刷新）

---

### 12.7 评论判定基于 memo_relation 关系

**当前实现**：

```sql
-- 评论判定依赖 memo_relation 表的存在
LEFT JOIN `memo_relation` 
    ON `memo`.`id` = `memo_relation`.`memo_id` 
    AND `memo_relation`.`type` = "COMMENT" 

-- 排除评论的条件
WHERE `parent_uid` IS NULL
```

**设计意图**：
- 评论是通过关系表关联的，不是 memo 的固有属性
- 一个 memo 可以同时是多个 memo 的评论（理论上）
- 关系删除后，memo 就不再被认为是评论

**潜在问题**：
- **如果关系被清理但 memo 本体残留，该 memo 会被当成正常 memo 返回**
- 可能导致原本的评论内容出现在正常列表中
- 可能导致统计数据异常

---

## 13. 残留 memo 的影响范围汇总（修正后的精确结论）

### 13.1 触发条件

| 条件 | 说明 |
|------|------|
| 存在多级评论 | 至少三级：A → B → C（C 是 B 的评论，B 是 A 的评论） |
| 删除顶层 memo | 删除 A 时触发 B 的删除，B 的删除触发 C→B 关系的清理 |
| API 层只处理直接评论 | 只递归一层，不处理更深层级 |
| Store.DeleteMemo 清理关系但不删除关联 memo | 清理 related_memo_id = B 的关系（即 C→B），但不会递归删除 C |
| ListMemos 使用 ExcludeComments=true | 默认行为，parent_uid=NULL 时满足条件 |

---

### 13.2 影响范围（精确）

| 影响方面 | 精确描述 | 前提条件 |
|---------|---------|---------|
| **数据库残留** | 多级评论（三级及以下）的 memo 本体残留，relations 被清理 | 满足上述所有触发条件 |
| **正常列表可见** | 残留 memo 会出现在首页、时间线等正常 memo 列表中 | `ListMemos(ExcludeComments=true)` 查询 |
| **用户统计异常** | TotalMemoCount、TagCount、MemoTypeStats 会计入残留 memo | `GetUserStats` 使用 `ExcludeComments=true` |
| **RSS 订阅** | 残留 memo 会出现在 RSS feed 中 | RSS 使用 `ExcludeComments=true` |
| **MCP 工具** | 残留 memo 会被 AI 助手查询到 | MCP 工具使用 `ExcludeComments=true` |
| **直接访问** | 通过 URL 直接访问残留 memo 仍可正常打开 | memo 本体存在，GetMemo 不查关系 |
| **内容暴露** | 如果残留 memo 是 PUBLIC/PROTECTED，可能被其他用户看到 | 依赖 visibility 设置 |
| **存储空间** | 残留 memo 的 content、payload、attachments 占用空间 | attachments 未被清理 |
| **Webhook 外部系统** | 外部系统不会收到残留 memo 的删除通知 | 子评论删除不触发 Webhook |

---

### 13.3 精确的可见性判定逻辑

```
残留 memo C 的数据库状态：
├── memo.id = C 的 ID
├── memo.row_status = NORMAL (或原值)
├── memo_relation.memo_id = C 的记录 → ❌ 已删除
└── memo_relation.related_memo_id = C 的记录 → 可能有（如果 C 有自己的评论）

ListMemos 查询时：
1. LEFT JOIN memo_relation 
   ON memo.id = memo_relation.memo_id 
   AND memo_relation.type = "COMMENT"
   → C 找不到匹配，因为关系已删除

2. 计算 parent_uid:
   CASE WHEN parent_memo.uid IS NOT NULL 
        THEN parent_memo.uid 
        ELSE NULL END
   → C 的 parent_uid = NULL

3. 过滤条件 (ExcludeComments=true):
   WHERE parent_uid IS NULL
   → C 满足条件！

4. 结论：
   C 会被返回，被当成正常 memo！
```

---

## 14. 相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `store/common.go` | RowStatus 定义 |
| `store/memo.go` | Store 层 DeleteMemo/UpdateMemo 实现 |
| `store/memo_relation.go` | MemoRelation 关系模型定义 |
| `store/db/sqlite/memo.go` | SQLite ListMemos 查询（含 ExcludeComments 逻辑） |
| `server/router/api/v1/memo_service.go` | API 层 DeleteMemo/UpdateMemo 实现 |
| `server/router/api/v1/memo_update_helpers.go` | 更新副作用（Webhook+SSE） |
| `server/router/api/v1/sse_hub.go` | SSE Hub 实现 |
| `server/router/api/v1/user_service_stats.go` | 用户统计（使用 ExcludeComments=true） |
| `internal/webhook/webhook.go` | Webhook 异步实现 |
| `web/src/components/MemoActionMenu/MemoActionMenu.tsx` | 前端操作菜单 UI（isComment 判定） |
| `web/src/components/MemoActionMenu/hooks.ts` | 前端操作处理逻辑 |
| `web/src/hooks/useLiveMemoRefresh.ts` | 前端 SSE 监听和缓存处理 |
| `web/src/pages/Archived.tsx` | 归档页面 |
| `proto/api/v1/common.proto` | Proto State 定义 |
