# Memo 生命周期与可见性控制

本文档详细描述 Memo 从创建、编辑到归档的完整生命周期，以及系统的可见性控制机制。

## 一、状态定义

### 1.1 核心状态枚举

Memo 有两种核心状态，在 API 层和 Store 层有对应映射：

**API 层 (State 枚举)** - `proto/api/v1/common.proto`:

| 值 | 枚举名 | 说明 |
|---|---|---|
| 0 | STATE_UNSPECIFIED | 未指定 |
| 1 | NORMAL | 正常状态 |
| 2 | ARCHIVED | 已归档 |

**Store 层 (RowStatus)** - `store/common.go:12-24`:

| 值 | 常量名 | 说明 |
|---|---|---|
| "NORMAL" | Normal | 正常状态 |
| "ARCHIVED" | Archived | 已归档 |

### 1.2 状态转换函数

状态在 API 层和 Store 层之间通过以下函数转换：

- `convertStateFromStore` - `server/router/api/v1/common.go:20-29`: Store → API
- `convertStateToStore` - `server/router/api/v1/common.go:31-36`: API → Store

## 二、可见性定义

### 2.1 可见性枚举

Memo 支持三种可见性级别：

**API 层 (Visibility 枚举)** - `proto/api/v1/memo_service.proto:142-147`:

| 值 | 枚举名 | 说明 |
|---|---|---|
| 0 | VISIBILITY_UNSPECIFIED | 未指定 |
| 1 | PRIVATE | 私有 - 仅创建者可见 |
| 2 | PROTECTED | 受保护 - 已登录用户可见 |
| 3 | PUBLIC | 公开 - 所有人可见 |

**Store 层 (Visibility)** - `store/memo.go:11-31`:

| 值 | 常量名 | 说明 |
|---|---|---|
| "PRIVATE" | Private | 私有 |
| "PROTECTED" | Protected | 受保护 |
| "PUBLIC" | Public | 公开 |

### 2.2 可见性转换函数

- `convertVisibilityFromStore` - `server/router/api/v1/memo_service_converter.go:350-361`: Store → API
- `convertVisibilityToStore` - `server/router/api/v1/memo_service_converter.go:363-372`: API → Store
  - 注意：默认返回 `Private`，即未指定时默认为私有

## 三、生命周期详解

### 3.1 创建 (CreateMemo)

**位置**: `server/router/api/v1/memo_service.go:41-155`

#### 创建流程

1. **权限检查**: 用户必须已登录
2. **UID 生成**: 验证或生成唯一的 memo UID
3. **构建创建对象**:
   ```go
   create := &store.Memo{
       UID:        memoUID,
       CreatorID:  user.ID,
       Content:    request.Memo.Content,
       Visibility: convertVisibilityToStore(request.Memo.Visibility),
   }
   ```
   - `RowStatus` 未显式设置，默认为 `Normal`
   - `Visibility` 从请求获取，默认 `Private`
4. **内容长度检查**: 超过限制返回 `InvalidArgument`
5. **Payload 重建**: 解析 Markdown 内容，提取 tags、properties 等
6. **创建 Memo**: 调用 `s.Store.CreateMemo(ctx, create)`
7. **处理附件和关系**: 如有需要，设置 attachments 和 relations
8. **Webhook 通知**: 触发 `memos.memo.created` webhook
9. **SSE 广播**: 向相关用户广播 `SSEEventMemoCreated` 事件
10. **@提及通知**: 解析内容中的 @mention 并发送通知

#### 默认状态和可见性

- **状态**: 默认为 `NORMAL`
- **可见性**: 
  - 请求中未指定时 → `PRIVATE`
  - 用户可在设置中配置默认可见性 (`memo_visibility`)
  - 系统 seed 默认: `PUBLIC` (`store/seed/sqlite/01__dump.sql:53`)
  - 用户设置默认: `PRIVATE` (`server/router/api/v1/user_service.go:434`)

### 3.2 编辑 (UpdateMemo)

**位置**: `server/router/api/v1/memo_service.go:436-541`

#### 可更新字段

通过 `update_mask` 指定要更新的字段：

| 字段路径 | 说明 |
|---|---|
| content | 内容 |
| visibility | 可见性 |
| pinned | 置顶状态 |
| state | 状态（用于归档/取消归档） |
| create_time | 创建时间 |
| update_time | 更新时间 |
| location | 位置信息 |
| attachments | 附件列表 |
| relations | 关联关系 |

#### 编辑流程

1. **权限检查**: 
   - 必须已登录
   - 只能是创建者或管理员
2. **获取现有 Memo**: 验证 memo 存在
3. **按 update_mask 处理更新**:
   - `content`: 重建 payload，检查内容长度
   - `visibility`: 转换为 store 层类型
   - `state`: 转换为 RowStatus（用于归档）
   - `attachments`/`relations`: 调用对应内部方法
4. **执行更新**: `s.Store.UpdateMemo(ctx, update)`
5. **触发副作用**:
   - 内容更新: 检查新的 @mention 并发送通知
   - 所有更新: 触发 `memos.memo.updated` webhook，广播 SSE 事件

### 3.3 归档与取消归档

归档通过 `UpdateMemo` API 实现，更新 `state` 字段。

#### 归档操作

**请求示例**:
```
PATCH /api/v1/memos/{memo_id}
{
  "memo": { "state": "ARCHIVED" },
  "update_mask": { "paths": ["state"] }
}
```

**处理逻辑** - `memo_service.go:492-494`:
```go
} else if path == "state" {
    rowStatus := convertStateToStore(request.Memo.State)
    update.RowStatus = &rowStatus
}
```

#### 取消归档操作

**请求示例**:
```
PATCH /api/v1/memos/{memo_id}
{
  "memo": { "state": "NORMAL" },
  "update_mask": { "paths": ["state"] }
}
```

#### 归档 Memo 的特殊性

**ListMemos 过滤** - `memo_service.go:167-178`:
```go
if request.State == v1pb.State_ARCHIVED {
    state := store.Archived
    memoFind.RowStatus = &state
    // 归档的 memo 只对创建者可见
    if currentUser == nil {
        return &v1pb.ListMemosResponse{}, nil
    }
    memoFind.CreatorID = &currentUser.ID
} else {
    state := store.Normal
    memoFind.RowStatus = &state
}
```

**GetMemo 权限检查** - `memo_service.go:338-347`:
```go
// 归档的 memo 只对创建者可见
if memo.RowStatus == store.Archived {
    user, err := s.fetchCurrentUser(ctx)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get user")
    }
    if user == nil || memo.CreatorID != user.ID {
        return nil, status.Errorf(codes.NotFound, "memo not found")
    }
}
```

**关键点**:
- 归档的 memo 对其他用户表现为"不存在"（返回 NotFound）
- 只有创建者可以查看和操作归档的 memo
- ListMemos 默认只返回 `NORMAL` 状态的 memo
- 要查看归档 memo，需显式指定 `state=ARCHIVED`

### 3.4 删除 (DeleteMemo)

**位置**: `server/router/api/v1/memo_service.go:543-618`

#### 删除流程

1. **权限检查**: 必须是创建者或管理员
2. **获取现有 Memo**: 验证存在
3. **Webhook 通知**: 触发 `memos.memo.deleted` webhook
4. **删除评论**: 先删除所有关联的评论 memo
5. **删除 Memo**: 调用 `s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: memo.ID})`
   - Store 层会处理关联的 relations 和 attachments 清理
6. **SSE 广播**: 广播 `SSEEventMemoDeleted` 事件

#### 删除特性

- **物理删除**: 不是软删除，数据从数据库移除
- **级联清理**: 
  - 评论会被先删除
  - 关联的 relations、attachments 会被清理
- **无法恢复**: 删除后无法找回

## 四、可见性控制机制

### 4.1 ListMemos 过滤逻辑

**位置**: `server/router/api/v1/memo_service.go:197-206`

```go
if currentUser == nil {
    // 未登录用户: 只能看到 PUBLIC
    memoFind.VisibilityList = []store.Visibility{store.Public}
} else {
    if memoFind.CreatorID == nil {
        // 未指定创建者: 自己的 + 他人的 PUBLIC/PROTECTED
        filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
        memoFind.Filters = append(memoFind.Filters, filter)
    } else if *memoFind.CreatorID != currentUser.ID {
        // 查看他人: 只能看到 PUBLIC/PROTECTED
        memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
    }
}
```

**规则总结**:

| 用户身份 | 可见范围 |
|---|---|
| 未登录 | 仅 `PUBLIC` |
| 已登录（查看自己） | 所有可见性 |
| 已登录（查看他人） | `PUBLIC` + `PROTECTED` |

### 4.2 GetMemo 权限检查

**位置**: `server/router/api/v1/memo_service.go:349-360`

```go
if memo.Visibility != store.Public {
    user, err := s.fetchCurrentUser(ctx)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get user")
    }
    if user == nil {
        return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
    }
    if memo.Visibility == store.Private && memo.CreatorID != user.ID {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
}
```

**规则总结**:

| Memo 可见性 | 未登录用户 | 已登录非创建者 | 创建者/管理员 |
|---|---|---|---|
| PUBLIC | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| PROTECTED | ❌ 401 Unauthenticated | ✅ 可见 | ✅ 可见 |
| PRIVATE | ❌ 401 Unauthenticated | ❌ 403 PermissionDenied | ✅ 可见 |

### 4.3 评论权限检查

**位置**: `server/router/api/v1/memo_service.go:641-643`

```go
if relatedMemo.Visibility == store.Private && relatedMemo.CreatorID != user.ID && !isSuperUser(user) {
    return nil, status.Errorf(codes.PermissionDenied, "permission denied")
}
```

**规则**:
- `PRIVATE` memo: 只有创建者和管理员可以评论
- `PROTECTED`/`PUBLIC`: 任何已登录用户可以评论

### 4.4 评论列表过滤

**位置**: `server/router/api/v1/memo_service.go:730-735`

```go
if currentUser == nil {
    memoFilter = `visibility == "PUBLIC"`
} else {
    memoFilter = fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
}
```

与 ListMemos 逻辑一致，确保评论也遵循相同的可见性规则。

## 五、默认可见性配置

### 5.1 配置层级

1. **系统级默认** - `store/seed/sqlite/01__dump.sql:53`:
   ```sql
   INSERT INTO system_setting VALUES ('MEMO_RELATED', '{"...","defaultVisibility":"PUBLIC",...}', '');
   ```
   新实例的默认实例设置。

2. **用户级默认** - `server/router/api/v1/user_service.go:434`:
   ```go
   MemoVisibility: "PRIVATE",
   ```
   新用户的默认设置。

3. **用户可配置** - `proto/api/v1/user_service.proto:439-440`:
   ```proto
   // The default visibility of the memo.
   string memo_visibility = 3 [(google.api.field_behavior) = OPTIONAL];
   ```

### 5.2 前端应用

**位置**: `web/src/components/MemoEditor/index.tsx:66-75`

```tsx
// 从用户设置获取默认可见性
const defaultVisibility = userGeneralSetting?.memoVisibility 
  ? convertVisibilityFromString(userGeneralSetting.memoVisibility) 
  : undefined;

// 传递给初始化 hook
useMemoInit({
  // ...
  defaultVisibility,
  // ...
});
```

**位置**: `web/src/components/MemoEditor/hooks/useMemoInit.ts:44-46`

```ts
if (defaultVisibility !== undefined) {
  dispatch(actions.setMetadata({ visibility: defaultVisibility }));
}
```

## 六、状态与可见性组合矩阵

| 状态 | 可见性 | 谁可以看到 | API 查询方式 |
|---|---|---|---|
| NORMAL | PUBLIC | 所有人 | 默认 ListMemos |
| NORMAL | PROTECTED | 已登录用户 | 默认 ListMemos |
| NORMAL | PRIVATE | 仅创建者 | 默认 ListMemos |
| ARCHIVED | 任意 | 仅创建者 | `?state=ARCHIVED` |

## 七、关键代码位置索引

| 功能 | 文件路径 | 行号 |
|---|---|---|
| State 定义 | `proto/api/v1/common.proto` | 7-11 |
| Visibility 定义 | `proto/api/v1/memo_service.proto` | 142-147 |
| RowStatus 定义 | `store/common.go` | 12-24 |
| Store Visibility 定义 | `store/memo.go` | 11-31 |
| CreateMemo | `server/router/api/v1/memo_service.go` | 41-155 |
| UpdateMemo | `server/router/api/v1/memo_service.go` | 436-541 |
| DeleteMemo | `server/router/api/v1/memo_service.go` | 543-618 |
| ListMemos 可见性过滤 | `server/router/api/v1/memo_service.go` | 197-206 |
| GetMemo 权限检查 | `server/router/api/v1/memo_service.go` | 338-360 |
| 状态转换函数 | `server/router/api/v1/common.go` | 20-36 |
| 可见性转换函数 | `server/router/api/v1/memo_service_converter.go` | 350-372 |
