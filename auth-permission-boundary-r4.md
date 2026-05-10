# Memos 认证与权限边界详细分析 (R4)

## 修订说明

本文档是 `auth-permission-boundary-r3.md` 的补充修订版，重点补充和统一：

1. **MCP 入口状态码映射**：将 "permission denied" 等字符串错误映射到 401、403、404 等标准 HTTP 状态码，并说明映射依据
2. **统一口径**：给出 API、File Server、MCP 三个入口并排对照的状态码矩阵，确保同一场景不再出现字符串口径与数值口径混用

---

## 1. MCP 入口错误码映射机制

### 1.1 MCP 架构的两层错误处理

MCP 入口有**两层**错误处理：

```
HTTP 层 (Echo 中间件)
    │
    ├── 有 Authorization 头但验证失败 → HTTP 401
    │                                   (mcp.go:99)
    │
    └── 无 Authorization 头或验证成功 → 继续执行
            │
            ▼
    MCP 工具执行层
            │
            ├── extractUserID() 失败 → "unauthenticated: ..."
            │                            (tools_memo.go:174-176)
            │
            ├── checkMemoAccess() 失败 → "permission denied"
            │                            (access.go:19, 25, 29)
            │
            ├── checkMemoOwnership() 失败 → "permission denied"
            │                               (access.go:39)
            │
            ├── checkAttachmentAccess() 失败 → "permission denied"
            │                                  (access.go:71, checkMemoAccess)
            │
            └── 资源不存在 → "xxx not found"
                                 (tools_attachment.go:174, 222, 283, access.go:79)
```

### 1.2 MCP 字符串错误到 HTTP 状态码的语义映射

| MCP 错误消息 | 触发位置 | 语义映射 | HTTP 状态码 | 依据 |
|-------------|---------|---------|------------|------|
| **HTTP 层** | | | | |
| `"invalid or expired token"` | `mcp.go:99` | 认证失败 | **401** | `http.StatusUnauthorized` |
| `"invalid origin"` | `mcp.go:71` | CORS 拒绝 | **403** | `http.StatusForbidden` |
| **工具执行层** | | | | |
| `"unauthenticated: a personal access token is required"` | `tools_memo.go:174-176` | 需要认证 | **401** | `extractUserID()` 明确检查 `id == 0` |
| `"permission denied"` | `access.go:19, 25, 29` | 已认证但无权限 | **403** | `checkMemoAccess()` 检查 `RowStatus == Archived` 且非所有者，或 `PRIVATE` 且非所有者 |
| `"permission denied"` | `access.go:39` | 已认证但无权限 | **403** | `checkMemoOwnership()` 检查 `CreatorID != userID` |
| `"permission denied"` | `access.go:71` | 已认证但无权限 | **403** | `checkAttachmentAccess()` 未关联附件且非所有者 |
| `"memo not found"` | `tools_attachment.go:174, 283` | 资源不存在 | **404** | `GetMemo()` 返回 `nil` |
| `"attachment not found"` | `tools_attachment.go:222` | 资源不存在 | **404** | `GetAttachment()` 返回 `nil` |
| `"linked memo not found"` | `access.go:79` | 资源不存在 | **404** | `GetMemo()` 返回 `nil` |

### 1.3 关键区别：有/无 Authorization 头

```
场景 A：有 Authorization 头（Bearer token）
    │
    ▼
中间件验证
    │
    ├── 验证成功 → userID 被设置，继续执行
    │
    └── 验证失败（过期、无效签名、被撤销）
            │
            ▼
        HTTP 401 Unauthorized
        {"message": "invalid or expired token"}
        (mcp.go:99)

场景 B：无 Authorization 头
    │
    ▼
中间件跳过验证
    │
    ▼
userID = 0（匿名）
    │
    ▼
工具执行
    │
    ├── 读操作（list_memos, get_memo）→ 按可见性规则
    │   ├── PUBLIC → 200
    │   ├── PROTECTED/PRIVATE/ARCHIVED → "permission denied" → 403
    │
    └── 写操作（create_memo, update_memo）
        │
        ▼
    extractUserID() 检查
        │
        ├── id == 0 → "unauthenticated: ..." → 401
        │
        └── id > 0 → 继续执行
```

### 1.4 MCP 附件访问的具体错误码映射

| 场景 | 触发函数 | 错误消息 | 映射状态码 |
|-----|---------|---------|-----------|
| 匿名访问未关联附件 | `checkAttachmentAccess` | `"permission denied"` | **403** |
| 非所有者访问未关联附件 | `checkAttachmentAccess` | `"permission denied"` | **403** |
| 关联的 memo 被删除 | `checkAttachmentAccess` | `"linked memo not found"` | **404** |
| 匿名访问 PROTECTED memo 附件 | `checkMemoAccess` | `"permission denied"` | **403** |
| 匿名访问 PRIVATE memo 附件 | `checkMemoAccess` | `"permission denied"` | **403** |
| 非所有者访问 PRIVATE memo 附件 | `checkMemoAccess` | `"permission denied"` | **403** |
| 非所有者访问 ARCHIVED memo 附件 | `checkMemoAccess` | `"permission denied"` | **403** |
| 写操作但未认证 | `extractUserID` | `"unauthenticated: ..."` | **401** |
| 工具需要认证但无 token | `extractUserID` | `"unauthenticated: ..."` | **401** |

---

## 2. 统一状态码矩阵

### 2.1 归档备忘录内容访问（三入口并排对照）

**场景：** 备忘录状态 = `ARCHIVED`

| 可见性 | 身份 | API (Memo内容) | MCP (Memo内容) | File Server (不适用) |
|-------|------|---------------|---------------|---------------------|
| **PUBLIC** | 匿名 | ❌ 404 | ❌ 403 | N/A |
| | 普通用户（非所有者） | ❌ 404 | ❌ 403 | N/A |
| | 所有者 | ✅ 200 | ✅ 200 | N/A |
| | ADMIN（非所有者） | ❌ 404 | ❌ 403 | N/A |
| **PROTECTED** | 匿名 | ❌ 404 | ❌ 403 | N/A |
| | 普通用户（非所有者） | ❌ 404 | ❌ 403 | N/A |
| | 所有者 | ✅ 200 | ✅ 200 | N/A |
| | ADMIN（非所有者） | ❌ 404 | ❌ 403 | N/A |
| **PRIVATE** | 匿名 | ❌ 404 | ❌ 403 | N/A |
| | 普通用户（非所有者） | ❌ 404 | ❌ 403 | N/A |
| | 所有者 | ✅ 200 | ✅ 200 | N/A |
| | ADMIN（非所有者） | ❌ 404 | ❌ 403 | N/A |

**映射依据说明：**

- **API 返回 404**：`checkMemoReadAccess` 显式返回 `codes.NotFound`（隐私保护，隐藏存在性）
- **MCP 返回 403**：`checkMemoAccess` 返回 `"permission denied"`，语义上是"已知道存在但无权限"

### 2.2 归档备忘录附件访问（三入口并排对照）

**场景：** 备忘录状态 = `ARCHIVED`，访问关联的附件

| 可见性 | 身份 | File Server (附件下载) | MCP (附件元数据) | API (间接：通过 memo 内容) |
|-------|------|----------------------|-----------------|-------------------------|
| **PUBLIC** | 匿名 | ✅ **200** | ❌ **403** | N/A（附件不是内容） |
| | 普通用户（非所有者） | ✅ **200** | ❌ **403** | N/A |
| | 所有者 | ✅ 200 | ✅ 200 | N/A |
| | ADMIN（非所有者） | ✅ **200** | ❌ **403** | N/A |
| | 有效 share_token | ✅ 200 | N/A（MCP 不支持） | N/A |
| **PROTECTED** | 匿名 | ❌ **401** | ❌ 403 | N/A |
| | 普通用户（非所有者） | ✅ 200 | ❌ **403** | N/A |
| | 所有者 | ✅ 200 | ✅ 200 | N/A |
| | ADMIN（非所有者） | ✅ 200 | ❌ **403** | N/A |
| | 有效 share_token | ✅ 200 | N/A | N/A |
| **PRIVATE** | 匿名 | ❌ 401 | ❌ 403 | N/A |
| | 普通用户（非所有者） | ❌ **403** | ❌ 403 | N/A |
| | 所有者 | ✅ 200 | ✅ 200 | N/A |
| | ADMIN（非所有者） | ✅ **200** | ❌ **403** | N/A |
| | 有效 share_token | ✅ 200 | N/A | N/A |

**关键差异（高亮标记）：**

| 场景 | File Server | MCP | 差异原因 |
|-----|-------------|-----|---------|
| PUBLIC+ARCHIVED+匿名 | 200 | 403 | File Server **不检查** ARCHIVED，只检查可见性 |
| PUBLIC+ARCHIVED+非所有者 | 200 | 403 | 同上 |
| PROTECTED+ARCHIVED+非所有者 | 200 | 403 | File Server 不检查 ARCHIVED |
| PRIVATE+ARCHIVED+ADMIN非所有者 | 200 | 403 | File Server **检查 ADMIN** 角色，MCP **不检查** |

### 2.3 未关联附件访问（MemoID = nil）

| 身份 | File Server (附件下载) | MCP (附件元数据) |
|------|----------------------|-----------------|
| 匿名 | ❌ 401 | ❌ 403（或 401，取决于工具） |
| 普通用户（非创建者） | ❌ 403 | ❌ 403 |
| 创建者 | ✅ 200 | ✅ 200 |
| ADMIN（非创建者） | ✅ **200** | ❌ **403** |

**差异原因：**
- File Server：`fileserver.go:647` 检查 `user.Role != RoleAdmin`
- MCP：`access.go:66-82` 只检查 `CreatorID == userID`，不检查 ADMIN

### 2.4 非归档状态对比（作为基线）

**场景：** 备忘录状态 = `NORMAL`

| 可见性 | 身份 | API (内容) | File Server (附件) | MCP (内容/附件) |
|-------|------|-----------|------------------|----------------|
| **PUBLIC** | 匿名 | ✅ 200 | ✅ 200 | ✅ 200 |
| | 普通用户（非所有者） | ✅ 200 | ✅ 200 | ✅ 200 |
| | ADMIN（非所有者） | ✅ 200 | ✅ 200 | ✅ 200 |
| **PROTECTED** | 匿名 | ❌ 401 | ❌ 401 | ❌ 403 |
| | 普通用户（非所有者） | ✅ 200 | ✅ 200 | ✅ 200 |
| | ADMIN（非所有者） | ✅ 200 | ✅ 200 | ✅ 200 |
| **PRIVATE** | 匿名 | ❌ 401 | ❌ 401 | ❌ 403 |
| | 普通用户（非所有者） | ❌ 403 | ❌ 403 | ❌ 403 |
| | 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| | ADMIN（非所有者） | ❌ **403** | ✅ **200** | ❌ **403** |

**基线差异：**
- 即使在非归档状态，ADMIN 也**不能读取** API 和 MCP 中的 PRIVATE 内容
- 但 ADMIN **可以读取** File Server 中的 PRIVATE 附件

---

## 3. 完整的错误码映射决策树

### 3.1 API 入口（Connect RPC）错误码决策树

```
请求进入 API (Connect RPC)
    │
    ▼
AuthInterceptor 检查
    │
    ├── 非公开端点且认证失败
    │       │
    │       ▼
    │   codes.Unauthenticated → HTTP 401
    │
    └── 认证通过 或 公开端点
            │
            ▼
    服务层权限检查 (checkMemoReadAccess)
            │
            ├── memo.RowStatus == Archived
            │       │
            │       └── 非所有者 → codes.NotFound → HTTP 404
            │
            ├── memo.Visibility != PUBLIC
            │       │
            │       ├── 未认证 → codes.Unauthenticated → HTTP 401
            │       │
            │       └── PRIVATE 且非所有者 → codes.PermissionDenied → HTTP 403
            │
            └── 其他 → 200 OK
```

### 3.2 File Server 入口错误码决策树

```
请求进入 File Server
    │
    ▼
checkAttachmentPermission
    │
    ├── 未关联附件 (MemoID == nil)
    │       │
    │       ├── 未认证 → http.StatusUnauthorized → HTTP 401
    │       │
    │       └── 非所有者且非 ADMIN → http.StatusForbidden → HTTP 403
    │
    ├── memo == nil (memo 已删除)
    │       │
    │       └── http.StatusNotFound → HTTP 404
    │
    ├── memo.Visibility == PUBLIC
    │       │
    │       └── 200 OK (不检查 ARCHIVED)
    │
    ├── 有有效 share_token
    │       │
    │       └── 200 OK (绕过所有检查)
    │
    ├── 未认证 (user == nil)
    │       │
    │       └── http.StatusUnauthorized → HTTP 401
    │
    └── memo.Visibility == PRIVATE 且非所有者且非 ADMIN
            │
            └── http.StatusForbidden → HTTP 403
    │
    └── 其他情况 → 200 OK (PROTECTED: 任何已认证用户)
```

### 3.3 MCP 入口错误码决策树（统一到 HTTP 状态码）

```
请求进入 MCP
    │
    ▼
HTTP 中间件检查
    │
    ├── Origin 不被允许 → http.StatusForbidden → HTTP 403
    │
    └── 有 Authorization 头但验证失败
            │
            ▼
        http.StatusUnauthorized → HTTP 401
    │
    └── 无 Authorization 头 或 验证成功
            │
            ▼
    MCP 工具执行层
            │
            ├── extractUserID() 失败 (写操作)
            │       │
            │       └── "unauthenticated: ..." → 语义 401
            │
            ├── checkMemoAccess() 失败
            │       │
            │       ├── ARCHIVED 且非所有者 → "permission denied" → 语义 403
            │       │
            │       ├── PROTECTED 且匿名 → "permission denied" → 语义 403
            │       │
            │       └── PRIVATE 且非所有者 → "permission denied" → 语义 403
            │
            ├── checkMemoOwnership() 失败
            │       │
            │       └── CreatorID != userID → "permission denied" → 语义 403
            │
            ├── checkAttachmentAccess() 失败
            │       │
            │       ├── 未关联且非所有者 → "permission denied" → 语义 403
            │       │
            │       └── memo 不存在 → "linked memo not found" → 语义 404
            │
            └── 资源不存在
                    │
                    └── "xxx not found" → 语义 404
```

---

## 4. 统一状态码汇总矩阵

### 4.1 归档状态 (ARCHIVED) 汇总

| 资源类型 | 可见性 | 身份 | API | File Server | MCP |
|---------|-------|------|-----|-------------|-----|
| **Memo 内容** | PUBLIC | 匿名 | 404 | N/A | 403 |
| | | 非所有者 | 404 | N/A | 403 |
| | | 所有者 | 200 | N/A | 200 |
| | | ADMIN（非所有者） | 404 | N/A | 403 |
| | PROTECTED | 匿名 | 404 | N/A | 403 |
| | | 非所有者 | 404 | N/A | 403 |
| | | 所有者 | 200 | N/A | 200 |
| | | ADMIN（非所有者） | 404 | N/A | 403 |
| | PRIVATE | 匿名 | 404 | N/A | 403 |
| | | 非所有者 | 404 | N/A | 403 |
| | | 所有者 | 200 | N/A | 200 |
| | | ADMIN（非所有者） | 404 | N/A | 403 |
| **Memo 附件** | PUBLIC | 匿名 | N/A | 200 | 403 |
| | | 非所有者 | N/A | 200 | 403 |
| | | 所有者 | N/A | 200 | 200 |
| | | ADMIN（非所有者） | N/A | 200 | 403 |
| | | 有效 share_token | N/A | 200 | N/A |
| | PROTECTED | 匿名 | N/A | 401 | 403 |
| | | 非所有者 | N/A | 200 | 403 |
| | | 所有者 | N/A | 200 | 200 |
| | | ADMIN（非所有者） | N/A | 200 | 403 |
| | | 有效 share_token | N/A | 200 | N/A |
| | PRIVATE | 匿名 | N/A | 401 | 403 |
| | | 非所有者 | N/A | 403 | 403 |
| | | 所有者 | N/A | 200 | 200 |
| | | ADMIN（非所有者） | N/A | 200 | 403 |
| | | 有效 share_token | N/A | 200 | N/A |
| **未关联附件** | N/A | 匿名 | N/A | 401 | 403/401 |
| | | 非创建者 | N/A | 403 | 403 |
| | | 创建者 | N/A | 200 | 200 |
| | | ADMIN（非创建者） | N/A | 200 | 403 |

### 4.2 正常状态 (NORMAL) 汇总

| 资源类型 | 可见性 | 身份 | API | File Server | MCP |
|---------|-------|------|-----|-------------|-----|
| **Memo 内容** | PUBLIC | 匿名 | 200 | N/A | 200 |
| | | 非所有者 | 200 | N/A | 200 |
| | | 所有者 | 200 | N/A | 200 |
| | | ADMIN（非所有者） | 200 | N/A | 200 |
| | PROTECTED | 匿名 | 401 | N/A | 403 |
| | | 非所有者 | 200 | N/A | 200 |
| | | 所有者 | 200 | N/A | 200 |
| | | ADMIN（非所有者） | 200 | N/A | 200 |
| | PRIVATE | 匿名 | 401 | N/A | 403 |
| | | 非所有者 | 403 | N/A | 403 |
| | | 所有者 | 200 | N/A | 200 |
| | | ADMIN（非所有者） | 403 | N/A | 403 |
| **Memo 附件** | PUBLIC | 匿名 | N/A | 200 | 200 |
| | | 非所有者 | N/A | 200 | 200 |
| | | 所有者 | N/A | 200 | 200 |
| | | ADMIN（非所有者） | N/A | 200 | 200 |
| | PROTECTED | 匿名 | N/A | 401 | 403 |
| | | 非所有者 | N/A | 200 | 200 |
| | | 所有者 | N/A | 200 | 200 |
| | | ADMIN（非所有者） | N/A | 200 | 200 |
| | PRIVATE | 匿名 | N/A | 401 | 403 |
| | | 非所有者 | N/A | 403 | 403 |
| | | 所有者 | N/A | 200 | 200 |
| | | ADMIN（非所有者） | N/A | 200 | 403 |

---

## 5. 设计意图与不一致性分析

### 5.1 三个入口的设计理念对比

| 入口 | 设计理念 | 权限策略 | 返回码策略 |
|-----|---------|---------|-----------|
| **API (MemoService)** | **隐私优先** | ARCHIVED 隐藏存在性（404），PRIVATE 不允许 ADMIN 读取 | 404 优先（隐私保护） |
| **File Server** | **灵活 + 后门** | 不检查 ARCHIVED，允许 ADMIN 访问 PRIVATE 附件 | 标准权限码（401/403/404） |
| **MCP** | **最小权限 + AI 安全** | 完全没有 ADMIN 特殊处理，严格限制 AI 助手 | "permission denied" 字符串（语义 403） |

### 5.2 所有不一致点汇总

| 不一致场景 | API | File Server | MCP | 影响 |
|-----------|-----|-------------|-----|------|
| **ARCHIVED 检查** | ✅ 检查 → 404 | ❌ **不检查** → 按可见性处理 | ✅ 检查 → 语义 403 | 归档备忘录的附件可能泄露 |
| **ADMIN 读取 PRIVATE** | ❌ 不允许 → 403 | ✅ 允许 → 200 | ❌ 不允许 → 语义 403 | ADMIN 可通过文件服务器访问私有附件 |
| **ADMIN 访问未关联附件** | N/A | ✅ 允许 → 200 | ❌ 不允许 → 语义 403 | ADMIN 可管理孤立文件 |
| **返回码策略** | 404（隐私保护） | 标准 HTTP 码 | 字符串错误（语义映射） | 客户端需要不同处理逻辑 |
| **share_token 支持** | N/A | ✅ 绕过所有检查 | ❌ 不支持 | 分享链接只在文件下载时有效 |
| **Refresh Cookie 回退** | ❌ 仅 Auth 端点 | ✅ 支持 | ❌ 不支持 | 浏览器访问文件时可自动刷新登录 |

### 5.3 401 vs 403 语义差异

| HTTP 码 | 语义 | 典型场景 |
|--------|------|---------|
| **401 Unauthorized** | "你是谁？" — 需要认证 | 没有 token、token 过期、token 无效、匿名访问需要认证的资源 |
| **403 Forbidden** | "我知道你是谁，但你不能这样做" — 已认证但无权限 | 非所有者访问 PRIVATE、非所有者访问 ARCHIVED、ADMIN 在 MCP 中无特殊权限 |
| **404 Not Found** | "不存在" — 资源真的不存在或被隐藏 | 资源已删除、ARCHIVED 且非所有者（API 隐私保护） |

**MCP 的特殊情况：**
- MCP 使用 `extractUserID()` 区分 401 和 403
- 匿名调用写操作 → `extractUserID()` 失败 → `"unauthenticated"` → 语义 **401**
- 已认证但无权限 → `"permission denied"` → 语义 **403**

---

## 6. 关键代码位置速查

### 6.1 错误码返回位置

| 功能 | 文件路径 | 函数/位置 | 返回码 |
|-----|---------|----------|--------|
| **API 入口** | | | |
| ARCHIVED 非所有者 | `memo_service.go:53-55` | `checkMemoReadAccess` | `codes.NotFound` → 404 |
| 未认证访问非 PUBLIC | `memo_service.go:62-64` | `checkMemoReadAccess` | `codes.Unauthenticated` → 401 |
| PRIVATE 非所有者 | `memo_service.go:67-69` | `checkMemoReadAccess` | `codes.PermissionDenied` → 403 |
| **File Server 入口** | | | |
| 未关联附件未认证 | `fileserver.go:644-646` | `checkAttachmentPermission` | `http.StatusUnauthorized` → 401 |
| 未关联附件非所有者非 ADMIN | `fileserver.go:647-649` | `checkAttachmentPermission` | `http.StatusForbidden` → 403 |
| Memo 不存在 | `fileserver.go:657-659` | `checkAttachmentPermission` | `http.StatusNotFound` → 404 |
| 非 PUBLIC 未认证 | `fileserver.go:679-681` | `checkAttachmentPermission` | `http.StatusUnauthorized` → 401 |
| PRIVATE 非所有者非 ADMIN | `fileserver.go:683-685` | `checkAttachmentPermission` | `http.StatusForbidden` → 403 |
| **MCP 入口** | | | |
| Token 无效（HTTP 层） | `mcp.go:99` | 中间件 | `http.StatusUnauthorized` → 401 |
| Origin 不允许 | `mcp.go:71` | 中间件 | `http.StatusForbidden` → 403 |
| 未认证调用写操作 | `tools_memo.go:174-176` | `extractUserID` | `"unauthenticated: ..."` → 语义 401 |
| ARCHIVED 非所有者 | `access.go:18-20` | `checkMemoAccess` | `"permission denied"` → 语义 403 |
| PROTECTED 匿名 | `access.go:24-26` | `checkMemoAccess` | `"permission denied"` → 语义 403 |
| PRIVATE 非所有者 | `access.go:28-30` | `checkMemoAccess` | `"permission denied"` → 语义 403 |
| Memo 所有权检查 | `access.go:38-40` | `checkMemoOwnership` | `"permission denied"` → 语义 403 |
| 未关联附件非所有者 | `access.go:70-72` | `checkAttachmentAccess` | `"permission denied"` → 语义 403 |
| Memo 不存在 | `access.go:78-80` | `checkAttachmentAccess` | `"linked memo not found"` → 语义 404 |

### 6.2 MCP 错误码映射规则

| MCP 层 | 错误类型 | 错误消息 | 映射 HTTP 码 |
|-------|---------|---------|------------|
| **HTTP 中间件** | 认证失败 | `"invalid or expired token"` | **401** |
| **HTTP 中间件** | CORS 拒绝 | `"invalid origin"` | **403** |
| **工具执行** | 需要认证 | `"unauthenticated: ..."` | **401** |
| **工具执行** | 权限不足 | `"permission denied"` | **403** |
| **工具执行** | 资源不存在 | `"xxx not found"` | **404** |

---

## 7. 结论

### 7.1 统一结论

经过对三个入口的深入分析和统一口径，结论如下：

#### 归档备忘录附件的实际行为

| 入口 | ARCHIVED 检查 | ADMIN 特殊处理 | 典型返回码 |
|-----|--------------|---------------|-----------|
| **File Server** | ❌ **不检查** | ✅ 可访问 PRIVATE | 200/401/403/404 |
| **API** | ✅ 检查 → 404 | ❌ 不能读取 PRIVATE | 200/401/403/404 |
| **MCP** | ✅ 检查 → 语义 403 | ❌ 无任何特殊处理 | 语义 200/401/403/404 |

#### 关键不一致性

1. **File Server 的宽松设计**：
   - PUBLIC + ARCHIVED 的附件可以被**任何人**访问（200）
   - PRIVATE + ARCHIVED 的附件可以被**ADMIN**访问（200）
   - 这是"后门"或"管理便利"的设计

2. **API 的隐私优先设计**：
   - 任何 ARCHIVED 备忘录对非所有者返回 **404**（隐私保护）
   - 即使 ADMIN 也不能读取他人的 PRIVATE 内容

3. **MCP 的最小权限设计**：
   - 完全没有 ADMIN 特殊处理
   - 所有错误以字符串形式返回，需要语义映射到 HTTP 状态码

### 7.2 MCP 状态码映射的最终确定

| MCP 错误消息 | 语义 | 映射 HTTP 码 |
|-------------|------|------------|
| `"unauthenticated: a personal access token is required"` | 需要认证 | **401** |
| `"invalid or expired token"` (HTTP 层) | 认证失败 | **401** |
| `"permission denied"` (ARCHIVED) | 已认证但无权限 | **403** |
| `"permission denied"` (PRIVATE) | 已认证但无权限 | **403** |
| `"permission denied"` (所有权) | 已认证但无权限 | **403** |
| `"xxx not found"` | 资源不存在 | **404** |

### 7.3 设计建议（如果追求一致性）

如果希望三个入口保持一致，可以考虑：

1. **File Server 添加 ARCHIVED 检查**：
   - 在 `checkAttachmentPermission` 中添加对 `memo.RowStatus == Archived` 的检查
   - 对非所有者返回 404（与 API 一致）

2. **MCP 明确返回结构化错误码**：
   - 当前 MCP 使用字符串错误，客户端需要自己映射
   - 可以考虑在 MCP 工具结果中包含错误类型字段

3. **统一 ADMIN 权限策略**：
   - 当前 ADMIN 在 File Server 有特殊权限，但在 API 和 MCP 没有
   - 需要明确 ADMIN 的职责范围（内容隐私 vs 管理便利）

**注意：这是设计决策，不是 bug。当前行为可能是有意的。**

---

## 附录：完整的状态码对照表

### 归档状态 (ARCHIVED) 完整矩阵

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    ARCHIVED 状态 — 三入口状态码并排对照                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  可见性: PUBLIC                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 身份          │ API (内容) │ File Server (附件) │ MCP (内容/附件)       │   │
│  ├───────────────┼───────────┼─────────────────────┼───────────────────────┤   │
│  │ 匿名          │ ❌ 404    │ ✅ 200              │ ❌ 403                │   │
│  │ 非所有者      │ ❌ 404    │ ✅ 200              │ ❌ 403                │   │
│  │ 所有者        │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ ADMIN (非所有)│ ❌ 404    │ ✅ 200              │ ❌ 403                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  可见性: PROTECTED                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 身份          │ API (内容) │ File Server (附件) │ MCP (内容/附件)       │   │
│  ├───────────────┼───────────┼─────────────────────┼───────────────────────┤   │
│  │ 匿名          │ ❌ 404    │ ❌ 401              │ ❌ 403                │   │
│  │ 非所有者      │ ❌ 404    │ ✅ 200              │ ❌ 403                │   │
│  │ 所有者        │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ ADMIN (非所有)│ ❌ 404    │ ✅ 200              │ ❌ 403                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  可见性: PRIVATE                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 身份          │ API (内容) │ File Server (附件) │ MCP (内容/附件)       │   │
│  ├───────────────┼───────────┼─────────────────────┼───────────────────────┤   │
│  │ 匿名          │ ❌ 404    │ ❌ 401              │ ❌ 403                │   │
│  │ 非所有者      │ ❌ 404    │ ❌ 403              │ ❌ 403                │   │
│  │ 所有者        │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ ADMIN (非所有)│ ❌ 404    │ ✅ 200              │ ❌ 403                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  差异汇总:                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 场景                          │ File Server │ MCP │ 差异原因              │   │
│  ├───────────────────────────────┼─────────────┼─────┼───────────────────────┤   │
│  │ PUBLIC+ARCHIVED+匿名          │ 200         │ 403 │ FS 不检查 ARCHIVED    │   │
│  │ PROTECTED+ARCHIVED+非所有者    │ 200         │ 403 │ FS 不检查 ARCHIVED    │   │
│  │ PRIVATE+ARCHIVED+ADMIN非所有者 │ 200         │ 403 │ FS 检查 ADMIN 角色    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 正常状态 (NORMAL) 完整矩阵

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     NORMAL 状态 — 三入口状态码并排对照                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  可见性: PUBLIC                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 身份          │ API (内容) │ File Server (附件) │ MCP (内容/附件)       │   │
│  ├───────────────┼───────────┼─────────────────────┼───────────────────────┤   │
│  │ 匿名          │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ 非所有者      │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ 所有者        │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ ADMIN (非所有)│ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  可见性: PROTECTED                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 身份          │ API (内容) │ File Server (附件) │ MCP (内容/附件)       │   │
│  ├───────────────┼───────────┼─────────────────────┼───────────────────────┤   │
│  │ 匿名          │ ❌ 401    │ ❌ 401              │ ❌ 403                │   │
│  │ 非所有者      │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ 所有者        │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ ADMIN (非所有)│ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  可见性: PRIVATE                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 身份          │ API (内容) │ File Server (附件) │ MCP (内容/附件)       │   │
│  ├───────────────┼───────────┼─────────────────────┼───────────────────────┤   │
│  │ 匿名          │ ❌ 401    │ ❌ 401              │ ❌ 403                │   │
│  │ 非所有者      │ ❌ 403    │ ❌ 403              │ ❌ 403                │   │
│  │ 所有者        │ ✅ 200    │ ✅ 200              │ ✅ 200                │   │
│  │ ADMIN (非所有)│ ❌ 403    │ ✅ 200              │ ❌ 403                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  差异汇总:                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 场景                          │ File Server │ MCP │ 差异原因              │   │
│  ├───────────────────────────────┼─────────────┼─────┼───────────────────────┤   │
│  │ PRIVATE+NORMAL+ADMIN非所有者  │ 200         │ 403 │ FS 检查 ADMIN 角色    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```
