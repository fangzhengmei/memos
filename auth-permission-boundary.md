# Memos 身份验证、权限分级与访问边界报告

## 1. 概述

本报告深入分析 Memos 项目中的用户身份验证机制、角色权限分级系统、访问令牌的颁发与校验流程，以及不同类型客户端在调用 API 时的权限判断边界。

Memos 采用现代化的认证架构，主要包括：
- **JWT 访问令牌**：短生命周期（15分钟），用于 API 访问
- **刷新令牌**：长生命周期（30天），存储在 HttpOnly Cookie 中
- **个人访问令牌 (PAT)**：用于程序化 API 访问
- **两级角色系统**：ADMIN 和 USER
- **多层权限检查**：拦截器层 + 服务层 + 业务逻辑层

---

## 2. 用户身份验证流程

### 2.1 认证方法概览

| 认证方式 | 存储位置 | 有效期 | 验证方式 | 适用场景 |
|---------|---------|--------|---------|---------|
| JWT 访问令牌 | 内存/localStorage | 15分钟 | 签名验证（无状态） | 常规 API 调用 |
| 刷新令牌 | HttpOnly Cookie | 30天 | 数据库查询（支持撤销） | 获取新访问令牌 |
| 个人访问令牌 (PAT) | 数据库（SHA-256 哈希） | 用户指定或永不过期 | 数据库查询 | 脚本/CLI/第三方集成 |

### 2.2 登录认证流程

#### 2.2.1 密码登录流程 (`SignIn`)

文件位置：`server/router/api/v1/auth_service.go:64-121`

1. **验证用户凭证**
   - 从请求中提取 username 和 password
   - 通过 username 查询用户 (`store.GetUser`)
   - 使用 bcrypt 比较密码哈希 (`bcrypt.CompareHashAndPassword`)

2. **检查实例设置**
   - 检查 `DisallowPasswordAuth` 配置
   - 如果禁止密码认证且用户角色为 USER，拒绝登录

3. **生成令牌**
   - 调用 `doSignIn` 方法生成双令牌
   - 刷新令牌存储在 HttpOnly Cookie 中
   - 访问令牌返回给前端

#### 2.2.2 SSO 登录流程

文件位置：`server/router/api/v1/auth_service.go:91-102, 134-217`

1. **解析 SSO 凭证**
   - 从请求中提取 IdP 名称、授权码、重定向 URI、PKCE 代码验证器

2. **获取身份提供商信息**
   - 通过 `resolveSSOIdentity` 获取 IdentityProvider 配置
   - 使用 OAuth2 客户端交换授权码获取用户信息

3. **解析用户信息**
   - 支持标识符过滤（`identifierFilter` 正则匹配）
   - 只允许符合条件的外部用户登录

4. **解析或创建本地用户**
   - 通过 `getLinkedSSOUser` 查找已绑定的本地用户
   - 如果不存在，检查实例注册设置
   - 首次登录自动创建本地用户（UUID 生成用户名）
   - 建立 `user_identity` 绑定记录

5. **处理并发竞争**
   - 使用唯一约束检查处理并发首次登录
   - 确保同一外部身份只绑定一个本地用户

### 2.3 令牌颁发流程 (`doSignIn`)

文件位置：`server/router/api/v1/auth_service.go:355-401`

```
用户登录成功
    ↓
生成刷新令牌 (JWT, tid=UUID, 30天)
    ↓
存储刷新令牌元数据 (user_setting 中的 refresh_tokens)
    - TokenId: UUID
    - ExpiresAt: 30天后
    - CreatedAt: 当前时间
    - ClientInfo: User-Agent, IP, Device, OS, Browser
    ↓
设置 HttpOnly Cookie (memos_refresh)
    - Path=/
    - HttpOnly
    - SameSite=Lax
    - Secure (HTTPS 时)
    ↓
生成访问令牌 (JWT, 15分钟)
    - Claims: user_id, username, role, status
    ↓
返回访问令牌 + 用户信息
```

### 2.4 令牌刷新流程 (`RefreshToken`)

文件位置：`server/router/api/v1/auth_service.go:435-520`

**关键特性：刷新令牌轮换 (Token Rotation)**

1. **提取旧刷新令牌**
   - 从 Cookie 中提取 `memos_refresh`
   - 使用 `AuthenticateByRefreshToken` 验证

2. **验证旧令牌**
   - JWT 签名验证
   - 检查 TokenId 是否存在于用户的刷新令牌列表中
   - 检查过期时间
   - 检查用户状态（非归档）

3. **生成新令牌对**
   - 生成新的刷新令牌（新 TokenId，新 30 天窗口）
   - 存储新刷新令牌元数据
   - **先添加再删除**：防止并发问题
   - 删除旧刷新令牌（使其失效）

4. **返回新访问令牌**
   - 设置新的刷新令牌 Cookie
   - 返回新的访问令牌（15分钟）

**安全优势：**
- 滑动窗口会话：活跃用户永久保持登录
- 被盗令牌在一次刷新后失效
- 可追踪每个会话的客户端信息

---

## 3. 访问令牌的校验流程

### 3.1 认证拦截器

文件位置：`server/router/api/v1/connect_interceptors.go:207-246`

**拦截器执行流程：**

1. **提取 Authorization 头**
   - 从 `req.Header().Get("Authorization")` 获取
   - 格式：`Bearer {token}`

2. **调用 Authenticator**
   - 优先尝试 JWT 访问令牌验证（无状态）
   - 失败后尝试 PAT 验证（有状态）

3. **公开端点检查**
   - 调用 `IsPublicMethod` 检查是否在白名单中
   - 非公开端点且认证失败返回 `401 Unauthenticated`

4. **设置上下文**
   - 调用 `auth.ApplyToContext` 将认证结果注入 context
   - 后续服务通过 `auth.GetUserID()`、`auth.GetUserClaims()` 获取

### 3.2 Authenticator 核心逻辑

文件位置：`server/auth/authenticator.go`

#### 3.2.1 JWT 访问令牌验证 (`AuthenticateByAccessTokenV2`)

```go
// 无状态验证 - 不查询数据库
func (a *Authenticator) AuthenticateByAccessTokenV2(accessToken string) (*UserClaims, error) {
    // 1. JWT 解析和签名验证
    // 2. 检查 Issuer = "memos"
    // 3. 检查 Audience = "user.access-token"
    // 4. 检查 Type = "access"
    // 5. 检查过期时间
    // 6. 提取 claims: user_id, username, role, status
}
```

**JWT 结构：**
- Header: `{"alg": "HS256", "kid": "v1", "typ": "JWT"}`
- Claims:
  ```json
  {
    "type": "access",
    "role": "ADMIN" | "USER",
    "status": "ACTIVE",
    "username": "steven",
    "iss": "memos",
    "aud": ["user.access-token"],
    "sub": "42",
    "iat": 1715000000,
    "exp": 1715000900
  }
  ```

#### 3.2.2 刷新令牌验证 (`AuthenticateByRefreshToken`)

```go
// 有状态验证 - 查询数据库
func (a *Authenticator) AuthenticateByRefreshToken(ctx context.Context, refreshToken string) (*store.User, string, error) {
    // 1. JWT 解析和签名验证
    // 2. 检查 Issuer = "memos"
    // 3. 检查 Audience = "user.refresh-token"
    // 4. 检查 Type = "refresh"
    // 5. 检查过期时间
    // 6. 查询数据库验证 TokenId 存在（撤销检查）
    // 7. 查询用户信息
    // 8. 检查用户状态（非归档）
}
```

**刷新令牌撤销机制：**
- 刷新令牌元数据存储在 `user_setting` 的 `refresh_tokens` 字段
- 登出或主动撤销时从列表中删除
- 数据库中的存在性检查实现撤销功能

#### 3.2.3 PAT 验证 (`AuthenticateByPAT`)

```go
// 有状态验证 - 查询数据库
func (a *Authenticator) AuthenticateByPAT(ctx context.Context, token string) (*store.User, *storepb.PersonalAccessTokensUserSetting_PersonalAccessToken, error) {
    // 1. 检查前缀 "memos_pat_"
    // 2. 计算 SHA-256 哈希
    // 3. 通过哈希查询用户
    // 4. 检查过期时间
    // 5. 检查用户状态
    // 6. 异步更新最后使用时间
}
```

**PAT 安全特性：**
- 数据库只存储 SHA-256 哈希，不存明文
- 前缀便于识别和区分
- 支持可选过期时间

### 3.3 上下文传递

文件位置：`server/auth/context.go`

认证结果通过 context 键传递：

| Context Key | 类型 | 说明 |
|------------|------|------|
| `UserIDContextKey` | `int32` | 认证用户 ID |
| `UserClaimsContextKey` | `*UserClaims` | JWT claims（含 role） |
| `AccessTokenContextKey` | `string` | 原始访问令牌 |

服务层获取当前用户：
```go
// 方法1：从 claims 获取（快速）
userID := auth.GetUserID(ctx)

// 方法2：获取完整用户对象（查询数据库）
currentUser, err := s.fetchCurrentUser(ctx)

// 方法3：获取 claims（含角色信息）
claims := auth.GetUserClaims(ctx)
role := claims.Role  // "ADMIN" 或 "USER"
```

---

## 4. 角色权限分级系统

### 4.1 角色定义

文件位置：`store/user.go:8-25`

```go
type Role string

const (
    RoleAdmin Role = "ADMIN"  // 管理员
    RoleUser  Role = "USER"   // 普通用户
)
```

**角色等级：**
```
ADMIN ──→ 拥有所有权限
  │
  └──→ 可以管理用户、系统设置
  └──→ 可以访问所有用户的资源
  └──→ 可以修改用户角色

USER ──→ 只能访问自己的资源
  └──→ 可以访问公开和受保护的资源
  └──→ 不能管理其他用户
```

### 4.2 权限检查层次

Memos 采用**多层防御**的权限检查策略：

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: 认证拦截器 (AuthInterceptor)                    │
│ - 检查是否需要认证                                        │
│ - 验证令牌有效性                                          │
│ - 设置用户上下文                                          │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: 服务层权限检查 (Service Layer)                  │
│ - 管理员专用操作检查                                      │
│ - 资源所有者检查                                          │
│ - 业务逻辑相关的权限判断                                   │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: 数据访问层过滤 (Data Access Filtering)          │
│ - 列表查询自动添加可见性过滤                               │
│ - 基于当前用户和角色的查询条件                            │
└─────────────────────────────────────────────────────────┘
```

### 4.3 管理员权限边界

#### 4.3.1 用户管理

文件位置：`server/router/api/v1/user_service.go`

| 操作 | 权限要求 | 说明 |
|-----|---------|------|
| `ListUsers` | ADMIN | 列出所有用户（含归档用户） |
| `CreateUser` (指定角色) | ADMIN | 可创建 ADMIN 或 USER |
| `UpdateUser` (修改角色) | ADMIN | 可升级/降级用户角色 |
| `UpdateUser` (修改状态) | ADMIN | 可归档/恢复用户 |
| `DeleteUser` | 本人 或 ADMIN | 可删除任意用户 |

**代码示例：**
```go
// server/router/api/v1/user_service.go:48-50
if currentUser.Role != store.RoleAdmin {
    return nil, status.Errorf(codes.PermissionDenied, "permission denied")
}
```

#### 4.3.2 实例设置

文件位置：`server/router/api/v1/instance_service.go`

| 操作 | 权限要求 |
|-----|---------|
| `GetInstanceProfile` | 公开 |
| `GetInstanceSetting` | 公开 |
| `UpdateInstanceSetting` | ADMIN |
| `SetInstanceOwner` | ADMIN |

#### 4.3.3 资源访问

文件位置：`server/router/api/v1/memo_service.go:42-71`

**Memo 可见性规则：**

| Memo 可见性 | 匿名用户 | 普通用户 | 管理员 |
|-----------|---------|---------|-------|
| PUBLIC | ✅ 可读 | ✅ 可读 | ✅ 可读/写 |
| PROTECTED | ❌ | ✅ 可读（非所有者） | ✅ 可读/写 |
| PRIVATE | ❌ | ❌（非所有者） | ✅ 可读/写 |
| ARCHIVED | ❌ | ❌（非所有者） | ❌（非所有者） |

**权限检查代码：**
```go
// server/router/api/v1/memo_service.go:42-71
func (s *APIV1Service) checkMemoReadAccess(ctx context.Context, memo *store.Memo) error {
    // 归档备忘录只对创建者可见
    if memo.RowStatus == store.Archived {
        if user == nil || memo.CreatorID != user.ID {
            return status.Errorf(codes.NotFound, "memo not found")
        }
    }
    
    // 非公开备忘录需要认证
    if memo.Visibility != store.Public {
        if user == nil {
            return status.Errorf(codes.Unauthenticated, "user not authenticated")
        }
        // 私有备忘录只对创建者可见
        if memo.Visibility == store.Private && memo.CreatorID != user.ID {
            return status.Errorf(codes.PermissionDenied, "permission denied")
        }
    }
    return nil
}
```

### 4.4 资源所有权检查

#### 4.4.1 Memo 操作权限

文件位置：`server/router/api/v1/common.go:81-83`

```go
func canModifyMemo(user *store.User, memo *store.Memo) bool {
    return user != nil && memo != nil && 
           (memo.CreatorID == user.ID || isSuperUser(user))
}
```

| 操作 | 权限要求 |
|-----|---------|
| 创建 Memo | 已认证用户 |
| 更新 Memo | 所有者 或 ADMIN |
| 删除 Memo | 所有者 或 ADMIN |

#### 4.4.2 用户设置权限

文件位置：`server/router/api/v1/user_service.go:532-535`

```go
// 只允许用户访问自己的设置
if currentUser.ID != userID {
    return nil, status.Errorf(codes.PermissionDenied, "permission denied")
}
```

| 设置类型 | 权限要求 |
|---------|---------|
| 个人设置 (GENERAL) | 本人 |
| Webhook 配置 | 本人 或 ADMIN |
| PAT 管理 | 本人 或 ADMIN |
| 链接身份 | 本人 或 ADMIN |

#### 4.4.3 附件访问权限

文件位置：`server/router/fileserver/fileserver.go:636-688`

**附件权限检查流程：**

```
附件访问请求
    ↓
是否关联到 Memo? ──否──→ 只允许创建者 或 ADMIN
    │
    是
    ↓
关联的 Memo 可见性?
    ├── PUBLIC → 允许任何人
    ├── PROTECTED → 需要认证
    └── PRIVATE → 需要是创建者 或 ADMIN
    │
    ↓
是否有有效的 share_token? ──是──→ 允许访问（分享链接）
```

### 4.5 列表查询过滤

文件位置：`server/router/api/v1/memo_service.go:229-238`

**ListMemos 自动过滤：**

```go
// 未认证用户：只能看到 PUBLIC
if currentUser == nil {
    memoFind.VisibilityList = []store.Visibility{store.Public}
} else {
    // 已认证用户：自己的 + PUBLIC + PROTECTED
    if memoFind.CreatorID == nil {
        filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
        memoFind.Filters = append(memoFind.Filters, filter)
    } else if *memoFind.CreatorID != currentUser.ID {
        // 查询他人：只能看到 PUBLIC + PROTECTED
        memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
    }
}
```

---

## 5. 公开端点白名单

文件位置：`server/router/api/v1/acl_config.go:11-41`

**无需认证即可访问的端点：**

```go
var PublicMethods = map[string]struct{}{
    // 认证服务
    "/memos.api.v1.AuthService/SignIn":          {},
    "/memos.api.v1.AuthService/RefreshToken":    {},
    
    // 实例信息
    "/memos.api.v1.InstanceService/GetInstanceProfile":       {},
    "/memos.api.v1.InstanceService/GetInstanceSetting":       {},
    "/memos.api.v1.InstanceService/BatchGetInstanceSettings": {},
    
    // 用户公开信息
    "/memos.api.v1.UserService/CreateUser":       {},  // 第一个用户注册
    "/memos.api.v1.UserService/GetUser":          {},
    "/memos.api.v1.UserService/BatchGetUsers":    {},
    "/memos.api.v1.UserService/GetUserAvatar":    {},
    "/memos.api.v1.UserService/GetUserStats":     {},
    "/memos.api.v1.UserService/ListAllUserStats": {},
    
    // SSO 配置
    "/memos.api.v1.IdentityProviderService/ListIdentityProviders": {},
    
    // Memo 公开访问（服务层进一步过滤可见性）
    "/memos.api.v1.MemoService/GetMemo":              {},
    "/memos.api.v1.MemoService/ListMemos":            {},
    "/memos.api.v1.MemoService/ListMemoComments":     {},
    "/memos.api.v1.MemoService/GetLinkMetadata":      {},
    "/memos.api.v1.MemoService/BatchGetLinkMetadata": {},
    
    // 分享链接
    "/memos.api.v1.MemoService/GetMemoByShare": {},
}
```

**重要说明：**
- 列入白名单只意味着**不需要认证**，不意味着**没有权限检查**
- 例如 `ListMemos` 是公开端点，但服务层会根据用户身份过滤可见的备忘录

---

## 6. 不同类型客户端的权限边界

### 6.1 客户端类型矩阵

| 客户端类型 | 认证方式 | 令牌存储 | 权限范围 | 典型场景 |
|-----------|---------|---------|---------|---------|
| **浏览器 Web 应用** | JWT + Refresh Cookie | localStorage (AT) + HttpOnly Cookie (RT) | 完整用户权限 | 主应用界面 |
| **移动端 App** | PAT 或 JWT | 安全存储 | 完整用户权限 | iOS/Android 应用 |
| **CLI 工具** | PAT | 配置文件 | 完整用户权限 | 脚本、自动化 |
| **第三方集成** | PAT | 第三方存储 | 完整用户权限 | API 集成 |
| **匿名访客** | 无 | 无 | 公开资源 | 未登录浏览 |
| **MCP 服务** | Bearer 令牌 | - | 基于当前用户 | AI 助手访问 |

### 6.2 浏览器 Web 应用

**文件位置：** `web/src/connect.ts`, `web/src/auth-state.ts`

**认证流程：**

1. **初始化**
   ```
   页面加载
       ↓
   检查 localStorage 中的访问令牌
       ↓
   令牌有效? ──是──→ 使用令牌
       │
       否
       ↓
   有存储的令牌? ──是──→ 调用 refreshToken()
       │
       否
       ↓
   未认证状态
   ```

2. **访问令牌管理** (`auth-state.ts`)
   - 存储在 localStorage（`memos_access_token`）
   - 过期时间存储在 localStorage（`memos_token_expires_at`）
   - BroadcastChannel 同步多标签页令牌

3. **自动刷新机制** (`connect.ts`)
   - **预刷新**：令牌过期前 30 秒自动刷新
   - **反应式刷新**：收到 401 时尝试刷新
   - **标签页聚焦刷新**：聚焦时提前 2 分钟刷新
   - **去重刷新**：`tokenRefreshManager` 防止并发刷新

4. **权限特点：**
   - 依赖 HttpOnly Cookie 中的刷新令牌（无法被 JS 访问）
   - 访问令牌短生命周期（15分钟）降低泄露风险
   - 完整的用户交互权限

### 6.3 个人访问令牌 (PAT) 客户端

**文件位置：** `server/router/api/v1/user_service.go:841-988`

**PAT 生命周期：**

```
创建 PAT (CreatePersonalAccessToken)
    ↓
生成随机字符串: memos_pat_ + 32字符随机
    ↓
计算 SHA-256 哈希
    ↓
存储到 user_setting.personal_access_tokens:
    - TokenId: UUID
    - TokenHash: SHA-256
    - Description: 用户描述
    - ExpiresAt: 可选过期时间
    - CreatedAt: 创建时间
    ↓
返回完整令牌（仅一次！）
    ↓
使用时验证:
    请求头: Authorization: Bearer memos_pat_xxx
        ↓
    计算哈希 → 数据库查询 → 验证过期
        ↓
    成功: 更新 LastUsedAt（异步）
```

**PAT 权限特点：**
- ✅ 与用户登录具有**相同的权限**
- ✅ 绕过双令牌刷新机制（直接验证）
- ✅ 永不过期（除非用户设置）
- ❌ 无法通过用户界面查看令牌值（只能看到元数据）
- ❌ 泄露风险高（长生命周期）

**安全建议：**
- 为不同用途创建不同 PAT
- 设置合理的过期时间
- 定期轮换 PAT
- 不再使用时立即删除

### 6.4 MCP (Model Context Protocol) 服务

**文件位置：** `server/router/mcp/access.go`

**MCP 权限检查：**

```go
// 检查 Memo 访问权限
func checkMemoAccess(memo *store.Memo, userID int32) error {
    // 归档：只允许创建者
    if memo.RowStatus == store.Archived && memo.CreatorID != userID {
        return errors.New("permission denied")
    }
    
    switch memo.Visibility {
    case store.Protected:
        if userID == 0 {
            return errors.New("permission denied")  // 需要认证
        }
    case store.Private:
        if memo.CreatorID != userID {
            return errors.New("permission denied")  // 只允许创建者
        }
    }
    return nil
}

// 列表过滤
func applyVisibilityFilter(find *store.FindMemo, userID int32, rowStatus *store.RowStatus) {
    if userID == 0 {
        find.VisibilityList = []store.Visibility{store.Public}
    } else {
        find.Filters = append(find.Filters, 
            "creator_id == "+userID+` || visibility in ["PUBLIC", "PROTECTED"]`)
    }
}
```

**MCP 权限特点：**
- 使用与主 API 相同的认证方式（Bearer 令牌）
- 遵循相同的可见性规则
- 额外的 Origin 检查（防止 CSRF）
- 工具级别权限检查

### 6.5 文件服务器

**文件位置：** `server/router/fileserver/fileserver.go`

**文件访问权限：**

| 文件类型 | 访问权限 |
|---------|---------|
| 用户头像 | 公开（任何人可访问） |
| 公开备忘录附件 | 公开 |
| 受保护备忘录附件 | 需要认证 |
| 私有备忘录附件 | 只允许创建者 或 ADMIN |
| 未关联附件 | 只允许创建者 或 ADMIN |
| 分享链接中的附件 | 有有效 share_token 即可 |

**特殊安全措施：**
```go
// XSS 防护：危险 MIME 类型作为附件下载
var xssUnsafeTypes = map[string]bool{
    "text/html":                true,
    "text/javascript":          true,
    "application/javascript":   true,
    // ...
}

// 这些类型会被转换为 application/octet-stream
// 并强制下载 (Content-Disposition: attachment)
```

### 6.6 匿名访客

**权限范围：**

| 操作 | 权限 |
|-----|------|
| 查看实例信息 | ✅ |
| 查看公开用户信息 | ✅ |
| 查看公开备忘录 | ✅ |
| 查看受保护备忘录 | ❌ |
| 查看私有备忘录 | ❌ |
| 创建备忘录 | ❌ |
| 修改任何内容 | ❌ |
| 访问用户设置 | ❌ |
| 第一个用户注册 | ✅（如果没有用户） |

---

## 7. 认证与权限架构图

### 7.1 整体认证架构

```
                    ┌─────────────────────────┐
                    │     Client (Web/App)    │
                    │  Authorization: Bearer  │
                    │  Cookie: memos_refresh  │
                    └───────────┬─────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │   Echo HTTP Server       │
                    └───────────┬─────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
   │  File Server   │  │ Connect RPC    │  │ MCP Server     │
   │  (fileserver)  │  │  (api/v1)      │  │  (mcp)         │
   └────────┬───────┘  └────────┬───────┘  └────────┬───────┘
            │                   │                   │
            ▼                   ▼                   ▼
   ┌──────────────────────────────────────────────────────────┐
   │              Authenticator (server/auth)                 │
   │  ┌──────────────────────────────────────────────────┐   │
   │  │  1. Extract Bearer token / Cookie                │   │
   │  │  2. Priority: JWT AT → PAT → Refresh Token       │   │
   │  │  3. Validate signature / database lookup        │   │
   │  │  4. Set context: UserID, Claims, AccessToken    │   │
   │  └──────────────────────────────────────────────────┘   │
   └──────────────────────────┬───────────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │              Service Layer Authorization                 │
   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
   │  │  Role Check  │  │ Owner Check  │  │ Visibility   │   │
   │  │  (ADMIN?)   │  │  (creator?)  │  │  (PUBLIC?)   │   │
   │  └──────────────┘  └──────────────┘  └──────────────┘   │
   └──────────────────────────┬───────────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │              Data Access Layer (store)                   │
   │  - Query filtering by visibility/role/user_id           │
   │  - User cache (TTL 10min, max 1000)                     │
   └──────────────────────────────────────────────────────────┘
```

### 7.2 令牌生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                        登录阶段 (SignIn)                            │
├─────────────────────────────────────────────────────────────────────┤
│  User ── username/password ──→ AuthService.SignIn()                │
│                                    │                                │
│                                    ▼                                │
│                          ┌────────────────┐                         │
│                          │ bcrypt 验证密码 │                         │
│                          └────────┬───────┘                         │
│                                   │                                  │
│                    ┌──────────────┴──────────────┐                  │
│                    ▼                             ▼                  │
│          ┌────────────────┐          ┌────────────────┐            │
│          │ Refresh Token  │          │ Access Token   │            │
│          │ (JWT, 30天)    │          │ (JWT, 15分钟)  │            │
│          │ tid=UUID       │          │ role/status    │            │
│          └────────┬───────┘          └────────┬───────┘            │
│                   │                           │                    │
│                   ▼                           ▼                    │
│          ┌────────────────┐          ┌────────────────┐            │
│          │ HttpOnly Cookie│          │ Response Body  │            │
│          │ memos_refresh  │          │ access_token   │            │
│          └────────────────┘          └────────────────┘            │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      使用阶段 (API 调用)                             │
├─────────────────────────────────────────────────────────────────────┤
│  Client ── Bearer AT ──→ API Endpoint                               │
│                              │                                      │
│                              ▼                                      │
│                    ┌────────────────┐                               │
│                    │ AuthInterceptor│                               │
│                    │ 验证 JWT 签名   │                               │
│                    └────────┬───────┘                               │
│                             │                                       │
│              ┌──────────────┴──────────────┐                        │
│              ▼                             ▼                        │
│      ┌────────────────┐          ┌────────────────┐                │
│      │  验证通过       │          │  验证失败       │                │
│      │ 设置 Context   │          │ 401 Unauth     │                │
│      └────────┬───────┘          └────────────────┘                │
│               │                                                     │
│               ▼                                                     │
│      ┌────────────────┐                                             │
│      │ Service Logic  │                                             │
│      │ 权限检查        │                                             │
│      └────────────────┘                                             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      刷新阶段 (Token Refresh)                        │
├─────────────────────────────────────────────────────────────────────┤
│  条件：AT 过期 或 401 响应                                          │
│                              │                                      │
│                              ▼                                      │
│  Client (Cookie: RT) ──→ AuthService.RefreshToken()                │
│                              │                                      │
│                              ▼                                      │
│                    ┌────────────────┐                               │
│                    │ 验证旧 RT      │                               │
│                    │ - JWT 签名     │                               │
│                    │ - 数据库存在性  │                               │
│                    │ - 过期时间     │                               │
│                    └────────┬───────┘                               │
│                             │                                       │
│              ┌──────────────┴──────────────┐                        │
│              ▼                             ▼                        │
│      ┌────────────────┐          ┌────────────────┐                │
│      │ 生成新 RT (轮换)│          │ 生成新 AT      │                │
│      │ 新 tid, 新 30天 │          │ 新 15分钟      │                │
│      └────────┬───────┘          └────────┬───────┘                │
│               │                           │                        │
│               ▼                           ▼                        │
│      ┌────────────────┐          ┌────────────────┐                │
│      │ 删除旧 RT      │          │ Response: AT   │                │
│      │ (数据库)       │          │                │                │
│      └────────────────┘          └────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. 安全特性总结

### 8.1 认证安全

| 特性 | 实现位置 | 说明 |
|-----|---------|------|
| **JWT 签名验证** | `token.go:206-217` | HS256 + kid 验证 |
| **刷新令牌轮换** | `auth_service.go:471-502` | 每次刷新生成新令牌，旧令牌失效 |
| **HttpOnly Cookie** | `auth_service.go:562-582` | 刷新令牌无法被 JS 访问 |
| **PAT 哈希存储** | `token.go:199-203` | SHA-256，不存明文 |
| **令牌撤销支持** | `authenticator.go:61-99` | 数据库存在性检查 |
| **过期时间** | `token.go:39-43` | AT=15min, RT=30天 |

### 8.2 授权安全

| 特性 | 实现位置 | 说明 |
|-----|---------|------|
| **多层检查** | 拦截器 + 服务层 | 深度防御 |
| **列表自动过滤** | `memo_service.go:229-238` | 防止越权查询 |
| **所有权检查** | `common.go:81-83` | 资源只能被所有者/管理员修改 |
| **可见性控制** | `memo_service.go:42-71` | PUBLIC/PROTECTED/PRIVATE |
| **归档隐藏** | 多处 | 归档资源只对所有者可见 |
| **分享令牌** | `fileserver.go:665-673` | 临时访问私有资源 |

### 8.3 前端安全

| 特性 | 实现位置 | 说明 |
|-----|---------|------|
| **访问令牌预刷新** | `connect.ts:136-147` | 过期前自动刷新 |
| **并发刷新去重** | `connect.ts:30-49` | 防止令牌竞争 |
| **多标签同步** | `auth-state.ts:12-46` | BroadcastChannel 共享令牌 |
| **响应式刷新** | `connect.ts:163-178` | 401 时自动重试 |

---

## 9. 关键代码位置索引

| 功能模块 | 文件路径 | 关键函数/类型 |
|---------|---------|-------------|
| **认证核心** | | |
| 令牌生成/解析 | `server/auth/token.go` | `GenerateAccessTokenV2`, `GenerateRefreshToken`, `ParseAccessTokenV2` |
| 认证器 | `server/auth/authenticator.go` | `Authenticator`, `Authenticate`, `AuthenticateByPAT` |
| 上下文管理 | `server/auth/context.go` | `UserClaims`, `GetUserID`, `ApplyToContext` |
| 令牌提取 | `server/auth/extract.go` | `ExtractBearerToken`, `ExtractRefreshTokenFromCookie` |
| **API 服务** | | |
| 认证服务 | `server/router/api/v1/auth_service.go` | `SignIn`, `RefreshToken`, `doSignIn` |
| 用户服务 | `server/router/api/v1/user_service.go` | 角色检查、PAT 管理 |
| 备忘录服务 | `server/router/api/v1/memo_service.go` | `checkMemoReadAccess`, `ListMemos` |
| 拦截器 | `server/router/api/v1/connect_interceptors.go` | `AuthInterceptor` |
| ACL 配置 | `server/router/api/v1/acl_config.go` | `PublicMethods` |
| **其他服务** | | |
| 文件服务器 | `server/router/fileserver/fileserver.go` | `checkAttachmentPermission` |
| MCP 服务 | `server/router/mcp/access.go` | `checkMemoAccess`, `applyVisibilityFilter` |
| **前端** | | |
| 令牌状态 | `web/src/auth-state.ts` | `getAccessToken`, `setAccessToken` |
| API 客户端 | `web/src/connect.ts` | `authInterceptor`, `refreshAccessToken` |

---

## 10. 权限决策流程示例

### 10.1 场景：普通用户访问他人的私有备忘录

```
请求: GET /api/v1/memos/xxx (PRIVATE, 他人创建)
    │
    ▼
拦截器: 认证通过 (currentUser = userA)
    │
    ▼
服务层 checkMemoReadAccess:
    ├── memo.Visibility = PRIVATE (非公开)
    ├── currentUser != nil (已认证) ✓
    └── memo.CreatorID != currentUser.ID (不是所有者)
        │
        ▼
    返回: 403 PermissionDenied
```

### 10.2 场景：管理员访问归档备忘录

```
请求: GET /api/v1/memos/xxx (ARCHIVED, 他人创建)
    │
    ▼
拦截器: 认证通过 (currentUser = admin, role=ADMIN)
    │
    ▼
服务层 checkMemoReadAccess:
    ├── memo.RowStatus = ARCHIVED
    ├── memo.CreatorID != currentUser.ID
    │
    ▼
    返回: 404 Not Found (归档只对所有者可见，即使是管理员)
```

### 10.3 场景：匿名用户访问公开备忘录

```
请求: GET /api/v1/memos/xxx (PUBLIC)
    │
    ▼
拦截器: 检查 PublicMethods → ListMemos 在白名单中
    │
    ▼
无认证，继续执行
    │
    ▼
服务层 checkMemoReadAccess:
    ├── memo.Visibility = PUBLIC ✓
    │
    ▼
    允许访问
```

### 10.4 场景：PAT 调用 API

```
请求: Authorization: Bearer memos_pat_xxx
    │
    ▼
拦截器 Authenticator:
    ├── 检测到前缀 "memos_pat_"
    ├── 计算 SHA-256 哈希
    ├── 数据库查询用户
    ├── 检查过期时间
    ├── 检查用户状态
    │
    ▼
认证通过，设置上下文
    │
    ▼
服务层权限检查: 与正常登录用户相同
```

---

## 11. 结论

Memos 的认证和权限系统设计具有以下特点：

### 优点

1. **现代化令牌架构**：双令牌系统（AT + RT）结合安全性和用户体验
2. **刷新令牌轮换**：有效降低令牌泄露风险
3. **多层权限检查**：拦截器 + 服务层 + 数据层，深度防御
4. **灵活的可见性控制**：三级备忘录可见性（PUBLIC/PROTECTED/PRIVATE）
5. **PAT 支持**：便于脚本和第三方集成
6. **详细的客户端信息**：可追踪登录设备，便于安全审计

### 设计原则

1. **白名单优先**：默认需要认证，仅明确列出的端点公开
2. **最小权限**：普通用户只能访问必要资源
3. **防御性编程**：即使是公开端点也在服务层进行权限检查
4. **无状态优先**：JWT 访问令牌无状态验证，提高性能
5. **可追溯性**：所有令牌操作都有元数据记录

### 安全边界总结

| 边界类型 | 控制机制 | 实施位置 |
|---------|---------|---------|
| 认证边界 | 令牌验证 + Cookie | `AuthInterceptor` |
| 角色边界 | ADMIN/USER 检查 | 各服务层 |
| 所有权边界 | creator_id 比较 | `canModifyMemo` 等 |
| 可见性边界 | 备忘录 visibility 字段 | `checkMemoReadAccess` |
| 列表过滤 | 查询条件自动添加 | `ListMemos`, `applyVisibilityFilter` |
| 撤销边界 | 数据库存在性检查 | `AuthenticateByRefreshToken` |

本报告全面分析了 Memos 的认证和权限系统，为理解、维护和扩展该系统提供了详细参考。
