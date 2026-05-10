# Memos 认证与权限边界详细分析 (R2)

## 修订说明

本文档是 `auth-permission-boundary.md` 的补充修订版，重点补充：

1. **ADMIN 在 PRIVATE、PROTECTED、ARCHIVED 资源上的真实访问边界**，分 API、附件、MCP 三个入口详细说明
2. **Bearer JWT、PAT、refresh cookie 在各入口的鉴权优先级与回退路径**
3. **典型请求的 401/403/404 判定矩阵**

---

## 1. ADMIN 权限边界深度分析

### 1.1 关键发现：权限检查的不一致性

**重要结论：** ADMIN 角色在不同入口的权限处理逻辑**不一致**：

| 场景 | 检查函数 | ADMIN 行为 |
|-----|---------|-----------|
| 读取 PRIVATE memo | `checkMemoReadAccess` | ❌ **拒绝**（与普通用户相同） |
| 修改任何 memo | `canModifyMemo` | ✅ **允许** |
| 读取 PRIVATE 附件 | `checkAttachmentPermission` | ✅ **允许** |
| 读取 ARCHIVED memo | `checkMemoReadAccess` | ❌ **拒绝**（与普通用户相同） |
| 读取 ARCHIVED 附件 | `checkAttachmentPermission` | ❌ **拒绝**（间接） |

### 1.2 Connect RPC API 入口

文件位置：`server/router/api/v1/memo_service.go:42-71`

#### 1.2.1 核心函数 `checkMemoReadAccess`

```go
func (s *APIV1Service) checkMemoReadAccess(ctx context.Context, memo *store.Memo) error {
    // ============================================
    // ARCHIVED 备忘录：只对创建者可见（ADMIN 也不例外）
    // ============================================
    if memo.RowStatus == store.Archived {
        user, err := s.fetchCurrentUser(ctx)
        if user == nil || memo.CreatorID != user.ID {
            // ❌ 返回 404，而不是 403
            return status.Errorf(codes.NotFound, "memo not found")
        }
    }

    // ============================================
    // 非 PUBLIC 备忘录需要认证
    // ============================================
    if memo.Visibility != store.Public {
        user, err := s.fetchCurrentUser(ctx)
        if user == nil {
            // ❌ 未认证返回 401
            return status.Errorf(codes.Unauthenticated, "user not authenticated")
        }
        // ============================================
        // PRIVATE 备忘录：只对创建者可见
        // ⚠️ 注意：这里没有检查 ADMIN 角色！
        // ============================================
        if memo.Visibility == store.Private && memo.CreatorID != user.ID {
            // ❌ 返回 403
            return status.Errorf(codes.PermissionDenied, "permission denied")
        }
        // PROTECTED 备忘录：任何已认证用户都可以读取
    }
    return nil
}
```

#### 1.2.2 修改权限 `canModifyMemo`

文件位置：`server/router/api/v1/common.go:81-83`

```go
func canModifyMemo(user *store.User, memo *store.Memo) bool {
    return user != nil && memo != nil && 
           (memo.CreatorID == user.ID || isSuperUser(user))
}

func isSuperUser(user *store.User) bool {
    return user.Role == store.RoleAdmin
}
```

**修改权限矩阵（API 入口）：**

| 操作 | 所有者 | ADMIN（非所有者） | 普通用户（非所有者） |
|-----|-------|------------------|-------------------|
| `GetMemo` (PUBLIC) | ✅ 200 | ✅ 200 | ✅ 200 |
| `GetMemo` (PROTECTED) | ✅ 200 | ✅ 200 | ✅ 200 |
| `GetMemo` (PRIVATE 他人) | N/A | ❌ **403** | ❌ 403 |
| `GetMemo` (ARCHIVED 他人) | N/A | ❌ **404** | ❌ 404 |
| `UpdateMemo` | ✅ 200 | ✅ 200 | ❌ 403 |
| `DeleteMemo` | ✅ 200 | ✅ 200 | ❌ 403 |
| `ListMemos` (默认) | 自己的 + PUBLIC + PROTECTED | 自己的 + PUBLIC + PROTECTED | 自己的 + PUBLIC + PROTECTED |
| `ListMemos` (ARCHIVED) | 只有自己的 | **只有自己的** | 只有自己的 |

#### 1.2.3 ListMemos 过滤逻辑

文件位置：`server/router/api/v1/memo_service.go:189-238`

```go
func (s *APIV1Service) ListMemos(ctx context.Context, request *v1pb.ListMemosRequest) (*v1pb.ListMemosResponse, error) {
    // ...
    
    if request.State == v1pb.State_ARCHIVED {
        // ============================================
        // ARCHIVED 状态：强制限制为当前用户
        // ⚠️ ADMIN 也只能看到自己的归档备忘录
        // ============================================
        if currentUser == nil {
            return &v1pb.ListMemosResponse{}, nil  // 空结果
        }
        memoFind.CreatorID = &currentUser.ID  // 强制过滤
    }
    
    // ...
    
    if currentUser == nil {
        // 匿名用户：只能看到 PUBLIC
        memoFind.VisibilityList = []store.Visibility{store.Public}
    } else {
        if memoFind.CreatorID == nil {
            // 已认证用户：自己的 + PUBLIC + PROTECTED
            // ⚠️ 没有给 ADMIN 特殊处理
            filter := fmt.Sprintf(
                `creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, 
                currentUser.ID,
            )
            memoFind.Filters = append(memoFind.Filters, filter)
        } else if *memoFind.CreatorID != currentUser.ID {
            // 查询他人：只能看到 PUBLIC + PROTECTED
            // ⚠️ ADMIN 也看不到他人的 PRIVATE
            memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
        }
    }
    // ...
}
```

**ListMemos 行为分析：**
- 归档状态查询：**所有人（包括 ADMIN）都只能看到自己的**
- 正常状态查询：ADMIN 也**看不到他人的 PRIVATE 备忘录**
- 这是一个**设计决策**：保护用户隐私

### 1.3 文件服务器入口

文件位置：`server/router/fileserver/fileserver.go:636-688`

#### 1.3.1 核心函数 `checkAttachmentPermission`

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, c *echo.Context, attachment *store.Attachment) error {
    // ============================================
    // Case 1: 未关联的附件 (MemoID == nil)
    // ============================================
    if attachment.MemoID == nil {
        user, err := s.getCurrentUser(ctx, c)
        if user == nil {
            return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
        }
        // ✅ ADMIN 可以访问任何未关联的附件
        if user.ID != attachment.CreatorID && user.Role != store.RoleAdmin {
            return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
        }
        return nil
    }

    // ============================================
    // Case 2: 已关联到备忘录的附件
    // ============================================
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
    if memo == nil {
        return echo.NewHTTPError(http.StatusNotFound, "memo not found")
    }

    if memo.Visibility == store.Public {
        return nil  // PUBLIC 备忘录的附件公开访问
    }

    // ============================================
    // 分享令牌：绕过其他权限检查
    // ============================================
    if shareToken := (*c).QueryParam("share_token"); shareToken != "" {
        ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
        if err == nil && ms != nil && !isMemoShareExpired(ms) && ms.MemoID == memo.ID {
            return nil  // ✅ 有效分享令牌允许访问
        }
    }

    user, err := s.getCurrentUser(ctx, c)
    if user == nil {
        return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
    }

    // ============================================
    // PRIVATE 备忘录的附件
    // ✅ ADMIN 可以访问
    // ============================================
    if memo.Visibility == store.Private && 
       user.ID != memo.CreatorID && 
       user.Role != store.RoleAdmin {
        return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
    }
    // PROTECTED 备忘录的附件：任何已认证用户都可以访问

    return nil
}
```

**附件权限矩阵（File Server 入口）：**

| 附件类型 | 匿名 | 普通用户（非所有者） | ADMIN（非所有者） |
|---------|------|-------------------|-----------------|
| 未关联附件 (MemoID=nil) | ❌ 401 | ❌ 403 | ✅ 200 |
| PUBLIC memo 附件 | ✅ 200 | ✅ 200 | ✅ 200 |
| PROTECTED memo 附件 | ❌ 401 | ✅ 200 | ✅ 200 |
| PRIVATE memo 附件 | ❌ 401 | ❌ **403** | ✅ **200** |
| ARCHIVED memo 附件 (PUBLIC) | ⚠️ 见下方 | ⚠️ 见下方 | ⚠️ 见下方 |
| ARCHIVED memo 附件 (PRIVATE) | ⚠️ 见下方 | ⚠️ 见下方 | ⚠️ 见下方 |
| 有有效 share_token | ✅ 200 | ✅ 200 | ✅ 200 |

**ARCHIVED 附件的特殊情况：**

```go
// 代码中没有显式检查 memo.RowStatus == Archived
// 但是 checkAttachmentPermission 会查询 memo：
// memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
// 问题：GetMemo 是否会返回归档备忘录？
// 
// 从代码分析：
// - checkAttachmentPermission 不检查 memo.RowStatus
// - 但如果 memo 是 ARCHIVED 且是 PRIVATE，ADMIN 仍然可以访问附件
// 
// 这与 API 入口的行为不一致！
// API 中 ADMIN 不能读取 ARCHIVED 备忘录
// File Server 中 ADMIN 可能可以读取 ARCHIVED 备忘录的附件
```

### 1.4 MCP 服务入口

文件位置：`server/router/mcp/access.go:15-64`

#### 1.4.1 核心权限函数

```go
func checkMemoAccess(memo *store.Memo, userID int32) error {
    // userID == 0 表示匿名
    
    // ============================================
    // ARCHIVED：只对创建者可见
    // ⚠️ 没有 ADMIN 特殊处理
    // ============================================
    if memo.RowStatus == store.Archived && memo.CreatorID != userID {
        return errors.New("permission denied")
    }

    switch memo.Visibility {
    case store.Protected:
        // PROTECTED：需要认证
        if userID == 0 {
            return errors.New("permission denied")
        }
    case store.Private:
        // ============================================
        // PRIVATE：只对创建者可见
        // ⚠️ 没有 ADMIN 特殊处理！
        // ============================================
        if memo.CreatorID != userID {
            return errors.New("permission denied")
        }
    default:
        // PUBLIC：允许
    }
    return nil
}

func checkMemoOwnership(memo *store.Memo, userID int32) error {
    // ============================================
    // 只检查 creator_id，没有 ADMIN 特殊处理
    // ============================================
    if memo.CreatorID != userID {
        return errors.New("permission denied")
    }
    return nil
}

func applyVisibilityFilter(find *store.FindMemo, userID int32, rowStatus *store.RowStatus) {
    if rowStatus != nil && *rowStatus == store.Archived {
        // ARCHIVED：强制限制为当前用户
        if userID == 0 {
            impossibleCreatorID := int32(-1)
            find.CreatorID = &impossibleCreatorID  // 返回空结果
            return
        }
        find.CreatorID = &userID  // 只看自己的
        return
    }
    if userID == 0 {
        // 匿名：只能看 PUBLIC
        find.VisibilityList = []store.Visibility{store.Public}
        return
    }
    // ============================================
    // 已认证：自己的 + PUBLIC + PROTECTED
    // ⚠️ 没有 ADMIN 特殊处理
    // ============================================
    find.Filters = append(find.Filters, 
        "creator_id == "+itoa32(userID)+` || visibility in ["PUBLIC", "PROTECTED"]`)
}
```

#### 1.4.2 MCP 认证入口

文件位置：`server/router/mcp/mcp.go:95-103`

```go
authHeader := c.Request().Header.Get("Authorization")
if authHeader != "" {
    result := s.authenticator.Authenticate(c.Request().Context(), authHeader)
    if result == nil {
        // 401 Unauthorized
        return c.JSON(http.StatusUnauthorized, 
            map[string]string{"message": "invalid or expired token"})
    }
    ctx := auth.ApplyToContext(c.Request().Context(), result)
    c.SetRequest(c.Request().WithContext(ctx))
}
// ⚠️ 如果没有 Authorization 头，不返回 401
// 而是继续执行，userID 为 0（匿名）
```

**MCP 权限矩阵：**

| 操作 | 匿名 | 普通用户 | ADMIN（非所有者） |
|-----|------|---------|-----------------|
| `list_memos` (默认) | 只有 PUBLIC | 自己的 + PUBLIC + PROTECTED | 自己的 + PUBLIC + PROTECTED |
| `list_memos` (ARCHIVED) | 空结果 | 只有自己的 | 只有自己的 |
| `get_memo` (PUBLIC) | ✅ | ✅ | ✅ |
| `get_memo` (PROTECTED) | ❌ | ✅ | ✅ |
| `get_memo` (PRIVATE 他人) | ❌ | ❌ | ❌ **（无特殊权限）** |
| `get_memo` (ARCHIVED 他人) | ❌ | ❌ | ❌ **（无特殊权限）** |
| `create_memo` | ❌ 401 | ✅ | ✅ |
| `update_memo` (他人) | ❌ 401 | ❌ | ❌ **（无特殊权限）** |
| `delete_memo` (他人) | ❌ 401 | ❌ | ❌ **（无特殊权限）** |

### 1.5 ADMIN 权限边界总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ADMIN 权限不一致性总结                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Connect RPC API                          File Server                   │
│  ┌──────────────────────────┐            ┌──────────────────────────┐  │
│  │ 读取权限 (checkMemoAccess)│            │ 附件权限 (checkAttachment)│  │
│  │                          │            │                          │  │
│  │ PUBLIC    → ✅ 所有人     │            │ PUBLIC    → ✅ 所有人     │  │
│  │ PROTECTED → ✅ 所有认证   │            │ PROTECTED → ✅ 所有认证   │  │
│  │ PRIVATE   → ❌ 仅所有者   │            │ PRIVATE   → ✅ 所有者+ADMIN│  │
│  │ ARCHIVED  → ❌ 仅所有者   │            │ ARCHIVED  → ⚠️ 未显式检查 │  │
│  │                          │            │                          │  │
│  │ 修改权限 (canModifyMemo)  │            │ 未关联附件                 │  │
│  │ 所有者 → ✅               │            │ → ✅ 所有者+ADMIN          │  │
│  │ ADMIN    → ✅ （所有）    │            │                          │  │
│  └──────────────────────────┘            └──────────────────────────┘  │
│                                                                         │
│                          MCP Service                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  checkMemoAccess / applyVisibilityFilter                         │  │
│  │                                                                  │  │
│  │  ⚠️ 完全没有 ADMIN 特殊处理                                     │  │
│  │                                                                  │  │
│  │  PRIVATE/ARCHIVED 他人 → ❌ ADMIN 也无法访问                    │  │
│  │  修改操作 → 只检查 creator_id，没有 isSuperUser                 │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**设计意图推测：**
1. **API 入口**：保护隐私是优先考虑，ADMIN 也看不到他人的私有内容
2. **修改操作**：ADMIN 可能需要管理违规内容，所以允许修改
3. **文件服务器**：可能是一个"后门"，方便 ADMIN 排查问题
4. **MCP**：可能是为了安全性，限制 AI 助手的权限范围

---

## 2. 鉴权优先级与回退路径

### 2.1 三个入口的鉴权方式对比

| 入口 | Bearer JWT | PAT | Refresh Cookie | 公开端点 | 匿名访问 |
|-----|-----------|-----|---------------|---------|---------|
| **Connect RPC (API)** | ✅ 优先级1 | ✅ 优先级2 | ❌ 不支持 | ✅ 白名单 | ✅ 仅白名单 |
| **File Server** | ✅ 优先级1 | ✅ 优先级2 | ✅ 优先级3 | N/A | ⚠️ 部分资源 |
| **MCP** | ✅ 优先级1 | ✅ 优先级2 | ❌ 不支持 | N/A | ✅ 只读操作 |
| **Auth Service** | N/A | N/A | ✅ 仅刷新端点 | ✅ | ✅ |

### 2.2 Connect RPC API 鉴权流程

文件位置：
- 拦截器：`server/router/api/v1/connect_interceptors.go:222-238`
- Authenticator：`server/auth/authenticator.go:174-210`

```
┌───────────────────────────────────────────────────────────────┐
│                    Connect RPC 鉴权流程                       │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  输入: Authorization Header + Procedure Name                  │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 1: 提取 Bearer Token                            │    │
│  │  ExtractBearerToken("Bearer {token}") → "{token}"   │    │
│  └──────────────────────────────────────────────────────┘    │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 2: Authenticate() 决策                          │    │
│  │  优先级判断逻辑:                                       │    │
│  │                                                        │    │
│  │  token.HasPrefix("memos_pat_")?                       │    │
│  │       │                                               │    │
│  │     ┌─┴──────────────────────────┐                    │    │
│  │     NO                          YES                    │    │
│  │     ↓                           ↓                     │    │
│  │  Step 2a: JWT AT 验证      Step 2b: PAT 验证          │    │
│  │  - HS256 签名验证         - SHA-256 哈希匹配          │    │
│  │  - iss="memos"           - 数据库查询                │    │
│  │  - aud="user.access-token" - 过期检查                 │    │
│  │  - type="access"         - 用户状态检查              │    │
│  │  - exp 检查               - 异步更新 LastUsedAt      │    │
│  │  - 查询用户状态           │                           │    │
│  │     │                                 │               │    │
│  │  成功? ──Yes──→ AuthResult  成功? ──Yes──→ AuthResult │    │
│  │     │     (Claims + AT)       │      (User + PAT)    │    │
│  │    No                        No                      │    │
│  │     │                         │                       │    │
│  │     └───────────┬─────────────┘                       │    │
│  │                 ↓                                     │    │
│  │  Step 3: 两者都失败 → result = nil                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 4: 公开端点检查                                  │    │
│  │  result == nil && !IsPublicMethod(procedure)?        │    │
│  │       │                                               │    │
│  │     ┌─┴─────────────────┐                             │    │
│  │     NO                  YES                          │    │
│  │     ↓                   ↓                             │    │
│  │  继续执行            ❌ 401 Unauthenticated          │    │
│  │  (公开端点)                                             │    │
│  └──────────────────────────────────────────────────────┘    │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 5: 注入上下文                                    │    │
│  │  auth.ApplyToContext(ctx, result)                    │    │
│  │  → UserIDContextKey                                   │    │
│  │  → UserClaimsContextKey                              │    │
│  │  → AccessTokenContextKey                             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ⚠️ 注意：Refresh Cookie 在 Connect RPC 中不使用！           │
│  只有 AuthService.RefreshToken 端点会读取 Cookie            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 2.3 File Server 鉴权流程

文件位置：
- `server/router/fileserver/fileserver.go:690-696`
- `server/auth/authenticator.go:133-172`

```
┌───────────────────────────────────────────────────────────────┐
│                    File Server 鉴权流程                        │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  输入: Authorization Header + Cookie Header                   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 1: AuthenticateToUser() 决策                    │    │
│  │  优先级判断逻辑:                                       │    │
│  │                                                        │    │
│  │  authHeader != ""?                                    │    │
│       │                                               │    │
│     ┌─┴──────────────────────────┐                    │    │
│     NO                          YES                    │    │
│     ↓                           ↓                     │    │
│  跳转到 Step 2          token.HasPrefix("memos_pat_")? │    │
│     (Cookie回退)                  │                     │    │
│                                 ┌─┴─────────┐           │    │
│                                 NO        YES          │    │
│                                 ↓         ↓            │    │
│                           JWT AT 验证  PAT 验证        │    │
│                                 │         │             │    │
│                               成功? ─Yes→ AuthResult   │    │
│                                 │                      │    │
│                                No                      │    │
│                                 │                      │    │
│  ┌──────────────────────────────┴──────────────────┐    │
│  │  Step 2: Refresh Cookie 回退（仅限 File Server） │    │
│  │                                                   │    │
│  │  cookieHeader != ""?                              │    │
│  │       │                                           │    │
│  │     ┌─┴─────────┐                                 │    │
│  │     NO         YES                               │    │
│  │     ↓           ↓                                │    │
│  │  user = nil  AuthenticateByRefreshToken()        │    │
│  │  (匿名)        - JWT 签名验证                    │    │
│  │                - 数据库存在性检查（撤销检查）     │    │
│  │                - 过期检查                         │    │
│  │                - 用户状态检查                     │    │
│  │                     │                            │    │
│  │                   成功? ──Yes──→ AuthResult      │    │
│  │                     │                            │    │
│  │                    No                            │    │
│  │                     ↓                            │    │
│  │                 user = nil                       │    │
│  │                 (匿名)                           │    │
│  └──────────────────────────────────────────────────┘    │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 3: checkAttachmentPermission() 权限检查         │    │
│  │  (根据资源类型决定匿名是否允许)                        │    │
│  │                                                        │    │
│  │  - 未关联附件: 需要认证                                │    │
│  │  - PUBLIC 备忘录附件: 允许匿名                         │    │
│  │  - PROTECTED 备忘录附件: 需要认证                      │    │
│  │  - PRIVATE 备忘录附件: 需要认证 + 所有者或 ADMIN       │    │
│  │  - 有有效 share_token: 允许访问                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 2.4 MCP 服务鉴权流程

文件位置：`server/router/mcp/mcp.go:95-103`

```
┌───────────────────────────────────────────────────────────────┐
│                      MCP 鉴权流程                              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  输入: Authorization Header                                   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Step 1: 检查 Authorization Header                   │    │
│  │  authHeader != ""?                                    │    │
│       │                                               │    │
│     ┌─┴─────────┐                                         │    │
│     NO         YES                                       │    │
│     ↓           ↓                                        │    │
│  Step 2      Step 2: Authenticate()                      │    │
│  (匿名)       优先级: JWT AT → PAT                       │    │
│     │             │                                      │    │
│     │           成功?                                    │    │
│     │        ┌──┴──┐                                     │    │
│     │       Yes   No                                    │    │
│     │       │      │                                    │    │
│     │       │   ❌ 401 Unauthorized                     │    │
│     │       │      "invalid or expired token"           │    │
│     │       ↓                                           │    │
│  Step 3: 注入上下文                                      │    │
│  auth.ApplyToContext()                                  │    │
│                                                               │
│  ⚠️ 关键差异:                                                │
│  1. 如果没有 Authorization 头 → 不返回 401，继续执行         │
│     → userID = 0 (匿名)，可执行只读操作                     │
│  2. 如果有 Authorization 头但验证失败 → 返回 401           │
│  3. 不支持 Refresh Cookie 回退！                            │
│  4. 没有公开端点白名单概念                                    │
│                                                               │
│  Step 4: 工具级别权限检查 (handleXxx 函数)                    │
│  - list_memos: 匿名可执行（只返回 PUBLIC）                   │
│  - create_memo: 必须认证 (extractUserID 检查)                │
│  - update_memo: 必须认证 + 所有权检查                         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 2.5 鉴权优先级总结表

| 入口 | 鉴权函数 | 优先级1 | 优先级2 | 优先级3 | 回退 |
|-----|---------|--------|--------|--------|------|
| **Connect RPC** | `Authenticate()` | JWT Access Token | PAT | ❌ 无 | 公开端点白名单 → 匿名访问 |
| **File Server** | `AuthenticateToUser()` | JWT Access Token | PAT | Refresh Cookie | 部分资源允许匿名 |
| **MCP** | `Authenticate()` | JWT Access Token | PAT | ❌ 无 | 无 Authorization 头 → 匿名 |
| **Refresh Token API** | `AuthenticateByRefreshToken()` | ❌ 不支持 | ❌ 不支持 | Refresh Cookie | ❌ 无 |

### 2.6 回退路径详细说明

**Connect RPC API 回退：**
```
请求 → 拦截器检查
    ↓
有 Authorization 头？
    ├── 是 → JWT → PAT → 都失败 → 公开端点检查
    │                           ├── 是白名单 → 匿名访问
    │                           └── 不是 → 401 Unauthenticated
    │
    └── 没有 Authorization 头 → 公开端点检查
                                ├── 是白名单 → 匿名访问
                                └── 不是 → 401 Unauthenticated
```

**File Server 回退：**
```
请求 → getCurrentUser()
    ↓
有 Authorization 头？
    ├── 是 → JWT → PAT → 都失败 → 检查 Cookie
    │                                  ├── 有 Refresh Token → 验证 → 成功/失败
    │                                  └── 没有/失败 → 匿名
    │
    └── 没有 Authorization 头 → 检查 Cookie
                                    ├── 有 Refresh Token → 验证 → 成功/失败
                                    └── 没有/失败 → 匿名
    ↓
根据资源类型判断匿名是否允许访问
```

**MCP 回退：**
```
请求 → 中间件检查
    ↓
有 Authorization 头？
    ├── 是 → JWT → PAT → 都失败 → 401 Unauthorized
    │
    └── 没有 Authorization 头 → userID = 0（匿名）
                                     ↓
                               工具级别检查
                               - 只读工具（list_memos）→ 允许
                               - 写工具（create_memo）→ 检查 extractUserID → 错误
```

---

## 3. 典型请求的 401/403/404 判定矩阵

### 3.1 HTTP 状态码与 gRPC Code 映射

| gRPC Code | HTTP Status | 含义 |
|-----------|------------|------|
| `codes.Unauthenticated` | **401** | 未认证 / 认证失败 |
| `codes.PermissionDenied` | **403** | 已认证但无权限 |
| `codes.NotFound` | **404** | 资源不存在 / 隐藏（隐私保护） |

**重要区分：**
- **401**："你是谁？" - 需要登录/认证
- **403**："我知道你是谁，但你不能这样做" - 已认证但权限不足
- **404**："不存在" - 资源确实不存在，或为了隐私保护隐藏存在性

### 3.2 Connect RPC API 错误码矩阵

#### 3.2.1 GetMemo 操作

| 资源类型 | 匿名用户 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|---------|---------|-------------------|-----------------|-----------------|
| **PUBLIC, NORMAL** | ✅ 200 | ✅ 200 | ✅ 200 | ✅ 200 |
| **PROTECTED, NORMAL** | ❌ 401 | ✅ 200 | ✅ 200 | ✅ 200 |
| **PRIVATE, NORMAL** | ❌ 401 | ❌ **403** | ✅ 200 | ❌ **403** |
| **PUBLIC, ARCHIVED** | ❌ **404** | ❌ **404** | ✅ 200 | ❌ **404** |
| **PROTECTED, ARCHIVED** | ❌ **404** | ❌ **404** | ✅ 200 | ❌ **404** |
| **PRIVATE, ARCHIVED** | ❌ **404** | ❌ **404** | ✅ 200 | ❌ **404** |
| **不存在的 memo** | ❌ 404 | ❌ 404 | ❌ 404 | ❌ 404 |

**设计说明：**
- ARCHIVED 状态返回 **404** 而不是 403 - 这是为了**隐私保护**
- 攻击者无法区分"真实不存在"和"存在但被归档"

#### 3.2.2 UpdateMemo / DeleteMemo 操作

| 资源类型 | 匿名用户 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|---------|---------|-------------------|-----------------|-----------------|
| **PUBLIC** | ❌ 401 | ❌ 403 | ✅ 200 | ✅ 200 |
| **PROTECTED** | ❌ 401 | ❌ 403 | ✅ 200 | ✅ 200 |
| **PRIVATE** | ❌ 401 | ❌ 403 | ✅ 200 | ✅ 200 |
| **PUBLIC, ARCHIVED** | ❌ 401 | ❌ 403 | ✅ 200 | ✅ 200 |
| **不存在的 memo** | ❌ 401（先检查认证） | ❌ 404（在检查权限前查询） | ❌ 404 | ❌ 404 |

**关键差异：**
- **读取权限**：ADMIN 不能读取他人的 PRIVATE
- **修改权限**：ADMIN **可以**修改任何 memo（包括 ARCHIVED）
- 这是**有意的设计**：ADMIN 可能需要管理违规内容

#### 3.2.3 ListMemos 操作

| 过滤条件 | 匿名用户 | 普通用户 | ADMIN |
|---------|---------|---------|-------|
| **默认 (NORMAL)** | 只有 PUBLIC | 自己的 + PUBLIC + PROTECTED | 自己的 + PUBLIC + PROTECTED |
| **state=ARCHIVED** | **空结果** (200) | **只有自己的** (200) | **只有自己的** (200) |
| **creator_id=他人** | 只有 PUBLIC | 他人的 PUBLIC + PROTECTED | 他人的 PUBLIC + PROTECTED |

**注意：** ListMemos 永远返回 **200**（空列表也是成功），不会返回 403/404，而是通过**查询过滤**实现权限控制。

#### 3.2.4 用户管理操作

| 操作 | 匿名 | 普通用户（自己） | 普通用户（他人） | ADMIN |
|-----|------|---------------|----------------|-------|
| **ListUsers** | ❌ 401 | ❌ 403 | ❌ 403 | ✅ 200 |
| **GetUser** | ✅ 200（公开信息） | ✅ 200 | ✅ 200 | ✅ 200 |
| **UpdateUser 基本信息** | ❌ 401 | ✅ 200 | ❌ 403 | ✅ 200 |
| **UpdateUser role** | ❌ 401 | ❌ 403 | ❌ 403 | ✅ 200 |
| **UpdateUser state** | ❌ 401 | ❌ 403 | ❌ 403 | ✅ 200 |
| **DeleteUser** | ❌ 401 | ✅ 200（自己） | ❌ 403 | ✅ 200 |
| **GetUserSetting** | ❌ 401 | ✅ 200 | ❌ 403 | ✅ 200 |
| **UpdateUserSetting** | ❌ 401 | ✅ 200 | ❌ 403 | ✅ 200 |

### 3.3 File Server 错误码矩阵

| 资源类型 | 匿名用户 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|---------|---------|-------------------|-----------------|-----------------|
| **PUBLIC memo 附件** | ✅ 200 | ✅ 200 | ✅ 200 | ✅ 200 |
| **PROTECTED memo 附件** | ❌ 401 | ✅ 200 | ✅ 200 | ✅ 200 |
| **PRIVATE memo 附件** | ❌ 401 | ❌ **403** | ✅ 200 | ✅ **200** |
| **未关联附件 (MemoID=nil)** | ❌ 401 | ❌ 403 | ✅ 200 | ✅ 200 |
| **memo 已删除** | ❌ 404 | ❌ 404 | ❌ 404 | ❌ 404 |
| **有有效 share_token** | ✅ 200 | ✅ 200 | ✅ 200 | ✅ 200 |

**File Server 与 API 的差异：**
- PRIVATE 附件：ADMIN **可以**访问（API 中 ADMIN 不能读取 memo 内容）
- ARCHIVED 附件：没有显式检查，行为取决于 memo 查询结果

### 3.4 MCP 服务错误码矩阵

MCP 使用工具级别的错误消息，不是标准 HTTP 状态码，但逻辑类似：

| 操作 | 匿名 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|-----|------|-------------------|-----------------|-----------------|
| **list_memos (默认)** | ✅ (只返回 PUBLIC) | ✅ (自己的 + PUBLIC + PROTECTED) | ✅ | ✅ (自己的 + PUBLIC + PROTECTED) |
| **list_memos (ARCHIVED)** | ✅ (空列表) | ✅ (只有自己的) | ✅ | ✅ (只有自己的) |
| **get_memo (PUBLIC)** | ✅ | ✅ | ✅ | ✅ |
| **get_memo (PROTECTED)** | ❌ "permission denied" | ✅ | ✅ | ✅ |
| **get_memo (PRIVATE 他人)** | ❌ "permission denied" | ❌ "permission denied" | ✅ | ❌ "permission denied" |
| **get_memo (ARCHIVED 他人)** | ❌ "permission denied" | ❌ "permission denied" | ✅ | ❌ "permission denied" |
| **create_memo** | ❌ "unauthenticated" | ✅ | ✅ | ✅ |
| **update_memo (自己的)** | ❌ "unauthenticated" | ✅ | ✅ | ✅ |
| **update_memo (他人的)** | ❌ "unauthenticated" | ❌ "permission denied" | ✅ | ❌ "permission denied" |
| **delete_memo (他人的)** | ❌ "unauthenticated" | ❌ "permission denied" | ✅ | ❌ "permission denied" |

**MCP 的关键特点：**
- 没有 ADMIN 特殊处理
- 所有用户遵循相同的规则
- 错误消息使用简单字符串，不是结构化错误码

### 3.5 认证层错误码（所有入口）

| 认证场景 | Connect RPC | File Server | MCP |
|---------|------------|-------------|-----|
| **无凭证，非公开端点** | 401 Unauthenticated | N/A（在权限层检查） | 工具级错误 |
| **无凭证，公开端点** | 200（匿名访问） | N/A | N/A |
| **有 Bearer 头，JWT 过期** | 401 Unauthenticated | 401 Unauthorized | 401 Unauthorized |
| **有 Bearer 头，JWT 签名错误** | 401 Unauthenticated | 401 Unauthorized | 401 Unauthorized |
| **有 Bearer 头，PAT 过期** | 401 Unauthenticated | 401 Unauthorized | 401 Unauthorized |
| **有 Bearer 头，PAT 被删除** | 401 Unauthenticated | 401 Unauthorized | 401 Unauthorized |
| **Refresh Token 过期** | N/A（401 触发前端刷新） | 401 Unauthorized | N/A |
| **Refresh Token 被撤销** | 401 Unauthenticated（RefreshToken 端点） | 401 Unauthorized | N/A |
| **用户被归档** | 401 Unauthenticated（令牌验证失败） | 401 Unauthorized | 401 Unauthorized |

---

## 4. 综合决策流程图

### 4.1 浏览器（Web 应用）完整请求流程

```
浏览器发起请求 (典型: GetMemo)
    │
    ├── 1. 检查 localStorage 中的 access_token
    │       ├── 存在且未过期 → 添加 Authorization: Bearer {at}
    │       └── 不存在或已过期 → 尝试 refreshToken()
    │                          ├── Cookie 中的 RT 有效 → 获取新 AT
    │                          └── RT 无效/不存在 → 未认证状态
    │
    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    Connect RPC 层                         │
    │                                                          │
    │  AuthInterceptor:                                         │
    │  ┌────────────────────────────────────────────────────┐  │
    │  │ 有 Authorization 头？                               │  │
    │  │     │                                               │  │
    │  │   ┌─┴─────┐                                         │  │
    │  │   NO     YES                                        │  │
    │  │   │       │                                         │  │
    │  │   │  JWT 验证 → PAT 验证                            │  │
    │  │   │       │                                         │  │
    │  │   │   成功? ──Yes──→ 注入上下文                      │  │
    │  │   │       │                                         │  │
    │  │   │      No                                         │  │
    │  │   │       │                                         │  │
    │  │  检查公开端点白名单                                   │  │
    │  │  GetMemo 在白名单中 ✅                                │  │
    │  └────────────────────────────────────────────────────┘  │
    └──────────────────────────────────────────────────────────┘
    │
    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    服务层 (checkMemoReadAccess)           │
    │                                                          │
    │  memo.RowStatus == ARCHIVED?                             │
    │       │                                                  │
    │     ┌─┴─────┐                                            │
    │     YES    NO                                            │
    │     │       │                                            │
    │     │  currentUser?.ID == creatorID?                     │
    │     │       │                                            │
    │     │   ┌───┴────┐                                       │
    │     │   Yes      No                                     │
    │     │   │        │                                      │
    │     │  200    ❌ 404 NotFound                           │
    │     │                                                 │
    │  memo.Visibility != PUBLIC?                             │
    │       │                                                  │
    │     ┌─┴─────┐                                            │
    │     NO     YES                                           │
    │     │       │                                            │
    │    200  currentUser == nil?                              │
    │             │                                            │
    │          ┌──┴──┐                                         │
    │         Yes    No                                        │
    │         │      │                                         │
    │         │   memo.Visibility == PRIVATE?                  │
    │         │        │                                       │
    │         │      ┌─┴─────┐                                 │
    │         │      NO     YES                               │
    │         │      │        │                                │
    │         │     200  creatorID == currentUser.ID?         │
    │         │             │                                 │
    │         │          ┌──┴──┐                              │
    │         │         Yes    No                             │
    │         │         │      │                              │
    │         │        200  ❌ 403 PermissionDenied          │
    │         │                                               │
    │    ❌ 401 Unauthenticated                              │
    │                                                          │
    │  ⚠️ 注意：没有检查 ADMIN 角色！                           │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

### 4.2 PAT 客户端请求流程

```
PAT 客户端 (curl/python/go)
    │
    │  手动设置: Authorization: Bearer memos_pat_xxx
    │
    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    Connect RPC 层                         │
    │                                                          │
    │  AuthInterceptor:                                         │
    │  ┌────────────────────────────────────────────────────┐  │
    │  │ token.HasPrefix("memos_pat_") → YES               │  │
    │  │                                                  │  │
    │  │ AuthenticateByPAT():                              │  │
    │  │ 1. SHA-256(token) → hash                          │  │
    │  │ 2. GetUserByPATHash(hash) → user                  │  │
    │  │ 3. 检查过期时间                                     │  │
    │  │ 4. 检查用户状态 (非 ARCHIVED)                      │  │
    │  │ 5. 异步更新 LastUsedAt                             │  │
    │  │                                                  │  │
    │  │ 成功 → 注入上下文                                  │  │
    │  │ 失败 → 公开端点检查                                 │  │
    │  └────────────────────────────────────────────────────┘  │
    └──────────────────────────────────────────────────────────┘
    │
    ▼
    服务层权限检查（与浏览器相同）
```

### 4.3 匿名访客请求流程

```
匿名访客 (未登录浏览器)
    │
    │  无 Authorization 头
    │
    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    Connect RPC 层                         │
    │                                                          │
    │  AuthInterceptor:                                         │
    │  result == nil                                            │
    │  检查公开端点白名单                                         │
    │       │                                                  │
    │     ┌─┴─────┐                                            │
    │     NO     YES                                           │
    │     │       │                                            │
    │  401     继续执行（匿名上下文）                            │
    │                                                          │
    │  白名单示例: GetMemo, ListMemos, GetInstanceProfile 等   │
    └──────────────────────────────────────────────────────────┘
    │
    ▼
    服务层权限检查
    (根据资源类型返回 200/401/403/404)
```

### 4.4 MCP 调用方请求流程

```
MCP 客户端 (AI 助手 / 工具)
    │
    │  可选: Authorization: Bearer {JWT 或 PAT}
    │
    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    MCP 中间件                             │
    │                                                          │
    │  有 Authorization 头？                                   │
    │       │                                                  │
    │     ┌─┴──────────────────────────┐                       │
    │     NO                          YES                      │
    │     │                           │                        │
    │  userID = 0                 JWT → PAT                   │
    │  (匿名)                        │                        │
    │                          ┌──┴──┐                        │
    │                         Yes    No                       │
    │                         │      │                        │
    │                      注入上下文 401 Unauthorized        │
    │                                                          │
    │  ⚠️ 没有公开端点白名单                                     │
    │  所有工具都可以被调用，但工具内部有检查                      │
    └──────────────────────────────────────────────────────────┘
    │
    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    工具处理器                              │
    │                                                          │
    │  list_memos (只读):                                       │
    │  userID == 0 → 只返回 PUBLIC                             │
    │  userID != 0 → 自己的 + PUBLIC + PROTECTED              │
    │  (没有 ADMIN 特殊处理)                                    │
    │                                                          │
    │  create_memo (写):                                       │
    │  extractUserID() → userID == 0 → 错误                   │
    │                                                          │
    │  update_memo (写):                                       │
    │  extractUserID() → 检查所有权                            │
    │  (没有 ADMIN 特殊处理)                                    │
    └──────────────────────────────────────────────────────────┘
```

---

## 5. 关键代码位置速查

### 5.1 ADMIN 权限检查

| 功能 | 文件路径 | 函数/位置 | ADMIN 特殊处理 |
|-----|---------|----------|--------------|
| 读取 memo 权限 | `memo_service.go:42-71` | `checkMemoReadAccess` | ❌ 无 |
| 修改 memo 权限 | `common.go:81-83` | `canModifyMemo` | ✅ 有 |
| ListMemos 过滤 | `memo_service.go:189-238` | `ListMemos` | ❌ 无 |
| 附件权限 | `fileserver.go:636-688` | `checkAttachmentPermission` | ✅ 有 (PRIVATE) |
| MCP memo 权限 | `access.go:15-35` | `checkMemoAccess` | ❌ 无 |
| MCP 列表过滤 | `access.go:48-64` | `applyVisibilityFilter` | ❌ 无 |
| MCP 所有权 | `access.go:37-46` | `checkMemoOwnership` | ❌ 无 |

### 5.2 鉴权优先级

| 功能 | 文件路径 | 优先级逻辑 |
|-----|---------|-----------|
| Connect RPC 鉴权 | `authenticator.go:174-210` | `Authenticate()`: JWT → PAT |
| File Server 鉴权 | `authenticator.go:133-172` | `AuthenticateToUser()`: JWT → PAT → Cookie |
| MCP 鉴权 | `authenticator.go:174-210` | 同 Connect RPC |
| 刷新令牌 | `auth_service.go:435-520` | `RefreshToken`: 仅 Cookie |

### 5.3 错误码返回

| 错误类型 | 文件路径 | 代码 |
|---------|---------|------|
| 401 Unauthenticated | 多处 | `codes.Unauthenticated` |
| 403 PermissionDenied | 多处 | `codes.PermissionDenied` |
| 404 NotFound | 多处 | `codes.NotFound` |
| 归档 memo 隐藏 | `memo_service.go:53-55` | 返回 404（隐私保护） |
| File Server 401 | `fileserver.go:645, 680` | `http.StatusUnauthorized` |
| File Server 403 | `fileserver.go:648, 684` | `http.StatusForbidden` |
| File Server 404 | `fileserver.go:658` | `http.StatusNotFound` |
| MCP 401 | `mcp.go:99` | `http.StatusUnauthorized` |

---

## 6. 结论

### 6.1 ADMIN 权限边界的关键发现

1. **API 入口的"隐私优先"设计**
   - ADMIN 不能读取他人的 PRIVATE 和 ARCHIVED 备忘录
   - 这是有意的设计决策，保护用户隐私
   - 但 ADMIN 可以修改/删除任何备忘录（管理违规内容）

2. **File Server 的"管理后门"**
   - PRIVATE 备忘录的附件对 ADMIN 开放
   - 可能是为了排查问题或管理违规内容
   - 与 API 入口的行为不一致

3. **MCP 的"最小权限"设计**
   - 完全没有 ADMIN 特殊处理
   - 限制 AI 助手的权限范围
   - 可能是考虑到 AI 助手的安全性

### 6.2 鉴权优先级的统一规则

| 规则 | Connect RPC | File Server | MCP |
|-----|------------|-------------|-----|
| JWT 优先于 PAT | ✅ | ✅ | ✅ |
| Refresh Cookie 回退 | ❌（仅 Auth 端点） | ✅ | ❌ |
| 无凭证时行为 | 检查公开白名单 | 检查资源类型 | 匿名继续执行 |
| 凭证验证失败 | 401 | 401 | 401 |

### 6.3 错误码使用原则

| HTTP 码 | 使用场景 | 典型原因 |
|--------|---------|---------|
| **401** | 认证层失败 | 没有凭证、凭证过期、签名错误、用户被归档 |
| **403** | 已认证但无权 | 访问他人的 PRIVATE 资源、修改不属于自己的资源 |
| **404** | 资源不存在或隐藏 | 资源真的不存在、ARCHIVED 状态（隐私保护） |

**隐私保护的特殊处理：**
- ARCHIVED 状态返回 404 而不是 403
- 这是为了防止攻击者枚举归档内容的存在性
