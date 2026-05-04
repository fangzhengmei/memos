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
// Archived memos are only visible to their creator.
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
- **ARCHIVED 状态优先级最高**：会覆盖可见性检查
- **信息隐藏**：非创建者访问归档 memo 时返回 `NotFound`（假装不存在），而不是 `PermissionDenied`
- 只有创建者可以查看和操作归档的 memo
- ListMemos 默认只返回 `NORMAL` 状态的 memo
- 要查看归档 memo，需显式指定 `state=ARCHIVED`，且只能看到自己的
- **归档的 memo 无法通过分享链接访问**

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

## 四、访问控制机制

Memo 有两种独立的访问路径：**常规 API 访问** 和 **分享链接访问**。

### 4.1 常规 API 访问控制

常规 API 包括 `GetMemo`、`ListMemos`、`ListMemoComments` 等，这些 API 在 ACL 中被标记为公开，但在服务层有严格的权限检查。

#### 4.1.1 权限检查顺序

**GetMemo 的检查逻辑** - `memo_service.go:338-360`:

```go
// 第一步：检查归档状态（优先级最高）
if memo.RowStatus == store.Archived {
    user, err := s.fetchCurrentUser(ctx)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get user")
    }
    if user == nil || memo.CreatorID != user.ID {
        return nil, status.Errorf(codes.NotFound, "memo not found")  // 信息隐藏
    }
}

// 第二步：检查可见性（仅对非 PUBLIC 可见性）
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

**关键规则**：

1. **ARCHIVED 状态优先**：
   - 如果是 ARCHIVED，非创建者统一返回 `NotFound`（假装不存在）
   - 这会**覆盖**后续的可见性检查

2. **可见性检查（仅当状态为 NORMAL 时）**：
   - `PUBLIC`：所有人可见（包括未登录）
   - `PROTECTED`：未登录返回 `Unauthenticated`，已登录则允许
   - `PRIVATE`：未登录返回 `Unauthenticated`，已登录非创建者返回 `PermissionDenied`

#### 4.1.2 完整权限矩阵

| 状态 | 可见性 | 未登录用户 | 已登录非创建者 | 创建者/管理员 |
|---|---|---|---|---|
| NORMAL | PUBLIC | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| NORMAL | PROTECTED | ❌ 401 Unauthenticated | ✅ 可见 | ✅ 可见 |
| NORMAL | PRIVATE | ❌ 401 Unauthenticated | ❌ 403 PermissionDenied | ✅ 可见 |
| ARCHIVED | PUBLIC | ❌ 404 NotFound | ❌ 404 NotFound | ✅ 可见 |
| ARCHIVED | PROTECTED | ❌ 404 NotFound | ❌ 404 NotFound | ✅ 可见 |
| ARCHIVED | PRIVATE | ❌ 404 NotFound | ❌ 404 NotFound | ✅ 可见 |

#### 4.1.3 错误码含义

| 错误码 | 触发场景 |
|---|---|
| 401 Unauthenticated | 未登录用户访问非 PUBLIC 的 NORMAL memo |
| 403 PermissionDenied | 已登录非创建者访问 PRIVATE 的 NORMAL memo |
| 404 NotFound | 任何非创建者访问 ARCHIVED memo（信息隐藏） |

#### 4.1.4 ListMemos 过滤逻辑

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

**ListMemos 行为总结**：

| 查询条件 | 未登录用户 | 已登录用户 |
|---|---|---|
| 无特殊参数 | 仅看到 NORMAL + PUBLIC | 看到 NORMAL +（自己的所有 + 他人的 PUBLIC/PROTECTED） |
| `?state=ARCHIVED` | 返回空列表 | 仅看到自己的 ARCHIVED memo（忽略可见性） |

#### 4.1.5 评论权限检查

**位置**: `server/router/api/v1/memo_service.go:641-643`

```go
if relatedMemo.Visibility == store.Private && relatedMemo.CreatorID != user.ID && !isSuperUser(user) {
    return nil, status.Errorf(codes.PermissionDenied, "permission denied")
}
```

**规则**：
- `PRIVATE` memo: 只有创建者和管理员可以评论
- `PROTECTED`/`PUBLIC`: 任何已登录用户可以评论

### 4.2 分享链接访问控制

分享链接是独立于可见性的访问机制，可以绕过常规的可见性限制。

#### 4.2.1 公开端点配置

**位置**: `server/router/api/v1/acl_config.go:38-39`

```go
// Memo sharing - share-token endpoints require no authentication
"/memos.api.v1.MemoService/GetMemoByShare": {},
```

`GetMemoByShare` 是完全公开的，**无需任何认证**。

#### 4.2.2 GetMemoByShare 实现

**位置**: `server/router/api/v1/memo_share_service.go:151-192`

```go
// GetMemoByShare resolves a share token to its memo. No authentication required.
// Returns NOT_FOUND for invalid or expired tokens (no information leakage).
func (s *APIV1Service) GetMemoByShare(ctx context.Context, request *v1pb.GetMemoByShareRequest) (*v1pb.Memo, error) {
    ms, err := s.getActiveMemoShare(ctx, request.ShareId)
    if err != nil {
        return nil, err
    }

    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: &ms.MemoID})
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get memo")
    }
    // Treat archived or missing memos the same as an invalid token — no information leakage.
    if memo == nil || memo.RowStatus == store.Archived {
        return nil, status.Errorf(codes.NotFound, "not found")
    }
    // ... 后续没有 visibility 检查！
}
```

**关键特性**：
- **无需认证**: 任何人持有有效 token 即可访问
- **绕过可见性**: 不检查 memo 的 `visibility` 字段（PRIVATE 也能访问）
- **唯一限制**: 
  - Token 必须有效且未过期
  - Memo 不能是 `ARCHIVED` 状态

#### 4.2.3 分享链接访问矩阵

| Memo 状态 | 任何用户（持有有效 token） |
|---|---|
| NORMAL + PUBLIC | ✅ 可见 |
| NORMAL + PROTECTED | ✅ 可见 |
| NORMAL + PRIVATE | ✅ 可见 |
| ARCHIVED + 任意 | ❌ 404 NotFound |

**重要**：分享链接可以访问 `PRIVATE` 的 memo！这是分享链接的设计目的——让你可以将私有内容分享给特定的人。

#### 4.2.4 附件的分享链接访问

**位置**: `server/router/fileserver/fileserver.go:665-673`

```go
// Check share token fallback: allow access if request carries a valid, non-expired share token
// that was issued for this specific memo. This covers attachment requests made from the shared
// memo page for private or protected memos.
if shareToken := (*c).QueryParam("share_token"); shareToken != "" {
    ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
    if err == nil && ms != nil && !isMemoShareExpired(ms) && ms.MemoID == memo.ID {
        return nil
    }
}
```

**规则**：
- 附件可以通过 `?share_token=` URL 参数绕过可见性检查
- 这允许分享链接页面正确显示私有/受保护 memo 的附件

#### 4.2.5 分享链接管理

**创建分享链接** (`CreateMemoShare`):
- 只能由 memo 创建者或管理员创建
- 可以设置过期时间（可选）
- Token 使用 shortuuid 生成（22 字符，122 位熵）

**撤销分享链接** (`DeleteMemoShare`):
- 只能由 memo 创建者或管理员撤销
- 撤销后 token 立即失效

**查看分享链接** (`ListMemoShares`):
- 只能由 memo 创建者或管理员查看

### 4.3 两种访问路径对比

| 特性 | 常规 API 访问 | 分享链接访问 |
|---|---|---|
| 认证要求 | 部分需要（依可见性而定） | 无需认证 |
| 可见性检查 | 严格执行 | **不检查** |
| 访问范围 | 依状态、可见性和身份而定 | 持有有效 token 即可 |
| 归档限制 | 仅创建者可见 | **完全无法访问** |
| 能否访问 PRIVATE | 仅创建者/管理员 | ✅ 持有 token 即可 |
| 典型用途 | 日常使用、浏览 | 分享给外部用户 |

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

## 六、状态与可见性组合矩阵

### 6.1 常规访问路径

| 状态 | 可见性 | 谁可以看到 |
|---|---|---|
| NORMAL | PUBLIC | 所有人（未登录 + 已登录） |
| NORMAL | PROTECTED | 已登录用户（任意） |
| NORMAL | PRIVATE | 仅创建者/管理员 |
| ARCHIVED | 任意 | 仅创建者/管理员（非创建者返回 404） |

### 6.2 分享链接访问路径

| 状态 | 可见性 | 谁可以看到（持有有效 token） |
|---|---|---|
| NORMAL | PUBLIC | ✅ 任何人 |
| NORMAL | PROTECTED | ✅ 任何人 |
| NORMAL | PRIVATE | ✅ 任何人 |
| ARCHIVED | 任意 | ❌ 无人（返回 404） |

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
| GetMemo 权限检查 | `server/router/api/v1/memo_service.go` | 338-360 |
| ListMemos 可见性过滤 | `server/router/api/v1/memo_service.go` | 197-206 |
| ListMemos 归档过滤 | `server/router/api/v1/memo_service.go` | 167-178 |
| GetMemoByShare | `server/router/api/v1/memo_share_service.go` | 151-192 |
| 分享链接 ACL 配置 | `server/router/api/v1/acl_config.go` | 38-39 |
| 附件分享 token 检查 | `server/router/fileserver/fileserver.go` | 665-673 |
| 状态转换函数 | `server/router/api/v1/common.go` | 20-36 |
| 可见性转换函数 | `server/router/api/v1/memo_service_converter.go` | 350-372 |
