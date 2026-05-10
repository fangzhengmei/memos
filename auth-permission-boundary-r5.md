# Memos 认证与权限边界详细分析 (R5 - 最终定稿)

## 修订说明

本文档是 `auth-permission-boundary-r4.md` 的最终修订版，重点解决：

1. **MCP 附件工具的认证差异**：分析 `get_attachment` vs `list_attachments` 等工具为何在匿名时返回不同状态码
2. **消除 401/403 分歧**：按具体工具和调用前提逐项拆开，给出每个场景的**唯一状态码**
3. **最终统一矩阵**：与 API、File Server 的同场景结果并排统一，避免任何双口径描述

---

## 1. MCP 附件工具的认证机制差异

### 1.1 核心发现：两种认证函数

MCP 中有**两种**获取用户 ID 的方式：

| 函数 | 实现 | 匿名行为 | 返回 |
|-----|------|---------|------|
| **`extractUserID(ctx)`** | 检查 `id == 0` | 抛出错误 | `"unauthenticated: a personal access token is required"` → **401** |
| **`auth.GetUserID(ctx)`** | 直接返回 id | 返回 0 | `userID = 0`（不报错） |

**代码对比：**

```go
// tools_memo.go:172-177
func extractUserID(ctx context.Context) (int32, error) {
    id := auth.GetUserID(ctx)
    if id == 0 {
        // 显式检查匿名，返回错误
        return 0, errors.New("unauthenticated: a personal access token is required")
    }
    return id, nil
}

// auth/context.go:15-20
func GetUserID(ctx context.Context) int32 {
    userID, ok := ctx.Value(UserIDContextKey).(int32)
    if !ok {
        return 0  // 匿名时返回 0，不报错
    }
    return userID
}
```

### 1.2 各附件工具的认证函数使用

| MCP 工具 | 使用的认证函数 | 匿名时的执行路径 | 返回码 |
|---------|---------------|-----------------|--------|
| **`list_attachments`** | `extractUserID(ctx)` | 函数直接返回错误 | **401** |
| **`get_attachment`** | `auth.GetUserID(ctx)` → `checkAttachmentAccess` | 继续执行，权限检查失败 | **403**（未关联）或根据 memo |
| **`delete_attachment`** | `extractUserID(ctx)` | 函数直接返回错误 | **401** |
| **`link_attachment_to_memo`** | `extractUserID(ctx)` | 函数直接返回错误 | **401** |

### 1.3 `get_attachment` 的详细执行流程

**核心函数：** `tools_attachment.go:209-234`

```
匿名调用 get_attachment (name="attachments/xxx")
    │
    ▼
1. userID := auth.GetUserID(ctx)
    │
    └── userID = 0（匿名，不报错）
    │
    ▼
2. 解析 UID → 获取 attachment
    │
    └── 不存在 → "attachment not found" → 404
    │
    └── 存在 → 继续
    │
    ▼
3. checkAttachmentAccess(ctx, attachment, userID=0)
    │
    ├── attachment.MemoID == nil (未关联)?
    │       │
    │       ├── Yes: "permission denied" → **403**
    │       │
    │       └── No: GetMemo({ID: attachment.MemoID})
    │                │
    │                └── memo == nil → "linked memo not found" → 404
    │                │
    │                └── memo 存在 → checkMemoAccess(memo, userID=0)
    │                         │
    │                         ├── memo.RowStatus == Archived → "permission denied" → 403
    │                         │
    │                         ├── memo.Visibility == PUBLIC → 允许 → 200
    │                         │
    │                         ├── memo.Visibility == PROTECTED → "permission denied" → 403
    │                         │
    │                         └── memo.Visibility == PRIVATE → "permission denied" → 403
```

### 1.4 `list_attachments` 的详细执行流程

**核心函数：** `tools_attachment.go:138-207`

```
匿名调用 list_attachments
    │
    ▼
1. userID, err := extractUserID(ctx)
    │
    └── userID = 0 → err = "unauthenticated: a personal access token is required"
    │
    ▼
2. 直接返回错误 → **401**
    │
    └── 不执行任何数据库查询
```

### 1.5 `get_attachment` vs `list_attachments` 的差异

| 维度 | `get_attachment` | `list_attachments` |
|-----|------------------|-------------------|
| 认证函数 | `auth.GetUserID` | `extractUserID` |
| 匿名检查位置 | `checkAttachmentAccess` | 函数入口 |
| 匿名返回码 | **403** | **401** |
| 是否查询数据库 | ✅ 是（获取 attachment，然后检查权限） | ❌ 否（直接返回错误） |
| 设计意图 | 允许匿名访问 PUBLIC memo 的附件？ | 列表是敏感操作，必须认证 |

---

## 2. MCP 附件访问的唯一状态码确定

### 2.1 未关联附件 (MemoID == nil)

**场景：** 附件未关联到任何备忘录 (`MemoID == nil`)

| 工具 | 身份 | 执行路径 | 最终状态码 |
|-----|------|---------|-----------|
| **get_attachment** | 匿名 | `auth.GetUserID` → `checkAttachmentAccess` → `"permission denied"` | **403** |
| **get_attachment** | 非创建者（已认证） | `CreatorID != userID` → `"permission denied"` | **403** |
| **get_attachment** | 创建者 | `CreatorID == userID` → 允许 | **200** |
| **list_attachments** | 匿名 | `extractUserID` → `"unauthenticated"` | **401** |
| **list_attachments** | 已认证 | `CreatorID = &userID` → 只返回自己的 | **200** |
| **delete_attachment** | 匿名 | `extractUserID` → `"unauthenticated"` | **401** |
| **link_attachment_to_memo** | 匿名 | `extractUserID` → `"unauthenticated"` | **401** |

**结论：未关联附件的状态码取决于具体工具**

### 2.2 关联附件 (MemoID != nil)

**场景：** 附件关联到备忘录，不同可见性 + 状态组合

#### 2.2.1 备忘录状态 = NORMAL

| 工具 | memo可见性 | 身份 | 执行路径 | 最终状态码 |
|-----|-----------|------|---------|-----------|
| **get_attachment** | PUBLIC | 匿名 | `checkMemoAccess` → PUBLIC 允许 | **200** |
| **get_attachment** | PUBLIC | 已认证（任何） | `checkMemoAccess` → PUBLIC 允许 | **200** |
| **get_attachment** | PROTECTED | 匿名 | `checkMemoAccess` → PROTECTED + 匿名 → `"permission denied"` | **403** |
| **get_attachment** | PROTECTED | 已认证（任何） | `checkMemoAccess` → PROTECTED + 已认证 → 允许 | **200** |
| **get_attachment** | PRIVATE | 匿名 | `checkMemoAccess` → PRIVATE + 非所有者 → `"permission denied"` | **403** |
| **get_attachment** | PRIVATE | 非所有者（已认证） | `checkMemoAccess` → PRIVATE + 非所有者 → `"permission denied"` | **403** |
| **get_attachment** | PRIVATE | 所有者 | `checkMemoAccess` → PRIVATE + 所有者 → 允许 | **200** |
| **get_attachment** | PRIVATE | ADMIN（非所有者） | `checkMemoAccess` → 不检查 ADMIN → `"permission denied"` | **403** |

#### 2.2.2 备忘录状态 = ARCHIVED

| 工具 | memo可见性 | 身份 | 执行路径 | 最终状态码 |
|-----|-----------|------|---------|-----------|
| **get_attachment** | 任何（PUBLIC/PROTECTED/PRIVATE） | 非所有者（含 ADMIN） | `checkMemoAccess` → Archived + 非所有者 → `"permission denied"` | **403** |
| **get_attachment** | 任何 | 所有者 | `checkMemoAccess` → Archived + 所有者 → 允许 | **200** |
| **get_attachment** | 任何 | 匿名 | `checkMemoAccess` → Archived + 匿名 → `"permission denied"` | **403** |

**关键：ARCHIVED 状态优先于可见性检查**

```go
// access.go:17-34
func checkMemoAccess(memo *store.Memo, userID int32) error {
    // ⚠️ ARCHIVED 检查优先于可见性检查
    if memo.RowStatus == store.Archived && memo.CreatorID != userID {
        return errors.New("permission denied")
    }
    
    // 然后才检查可见性
    switch memo.Visibility {
    case store.Protected:
        if userID == 0 {
            return errors.New("permission denied")
        }
    case store.Private:
        if memo.CreatorID != userID {
            return errors.New("permission denied")
        }
    }
    return nil
}
```

---

## 3. 三入口最终统一状态码矩阵

### 3.1 未关联附件 (MemoID == nil)

**MCP 注意：状态码取决于具体工具**

| 场景 | API | File Server | MCP (get_attachment) | MCP (list_attachments) |
|-----|-----|-------------|---------------------|-----------------------|
| **匿名** | N/A（无此概念） | ❌ **401** | ❌ **403** | ❌ **401** |
| **非创建者（已认证非 ADMIN）** | N/A | ❌ **403** | ❌ **403** | N/A（列表只返回自己的） |
| **非创建者（已认证 ADMIN）** | N/A | ✅ **200** | ❌ **403** | N/A |
| **创建者** | N/A | ✅ **200** | ✅ **200** | ✅ **200** |

**差异分析：**

| 场景 | API | File Server | MCP | 差异原因 |
|-----|-----|-------------|-----|---------|
| 匿名 | N/A | 401 | 403/401 | File Server 先检查认证，MCP `get_attachment` 先查资源再查权限 |
| ADMIN 访问他人的 | N/A | 200 | 403 | File Server 检查 `user.Role != RoleAdmin`，MCP 不检查 ADMIN |

### 3.2 关联附件 - 备忘录状态 = NORMAL

#### 3.2.1 可见性 = PUBLIC

| 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|------|-----------|-------------------|---------------------|
| 匿名 | ✅ 200 | ✅ 200 | ✅ 200 |
| 非所有者（已认证） | ✅ 200 | ✅ 200 | ✅ 200 |
| 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| ADMIN（非所有者） | ✅ 200 | ✅ 200 | ✅ 200 |

**结论：三入口一致，PUBLIC 资源所有人可访问。**

#### 3.2.2 可见性 = PROTECTED

| 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|------|-----------|-------------------|---------------------|
| 匿名 | ❌ **401** | ❌ **401** | ❌ **403** |
| 非所有者（已认证） | ✅ 200 | ✅ 200 | ✅ 200 |
| 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| ADMIN（非所有者） | ✅ 200 | ✅ 200 | ✅ 200 |

**差异：匿名访问时，API/FS 返回 401，MCP 返回 403**

原因：API/FS 先检查认证，MCP `get_attachment` 先查资源再查权限。

#### 3.2.3 可见性 = PRIVATE

| 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|------|-----------|-------------------|---------------------|
| 匿名 | ❌ 401 | ❌ 401 | ❌ 403 |
| 非所有者（已认证非 ADMIN） | ❌ **403** | ❌ **403** | ❌ **403** |
| 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| ADMIN（非所有者） | ❌ **403** | ✅ **200** | ❌ **403** |

**关键差异：ADMIN 访问他人的 PRIVATE 附件**
- File Server：**200**（检查 `user.Role != RoleAdmin`）
- API/MCP：**403**（不检查 ADMIN 角色）

### 3.3 关联附件 - 备忘录状态 = ARCHIVED

#### 3.3.1 可见性 = PUBLIC

| 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|------|-----------|-------------------|---------------------|
| 匿名 | ❌ **404** | ✅ **200** | ❌ **403** |
| 非所有者（已认证） | ❌ **404** | ✅ **200** | ❌ **403** |
| 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| ADMIN（非所有者） | ❌ **404** | ✅ **200** | ❌ **403** |

#### 3.3.2 可见性 = PROTECTED

| 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|------|-----------|-------------------|---------------------|
| 匿名 | ❌ **404** | ❌ **401** | ❌ **403** |
| 非所有者（已认证） | ❌ **404** | ✅ **200** | ❌ **403** |
| 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| ADMIN（非所有者） | ❌ **404** | ✅ **200** | ❌ **403** |

#### 3.3.3 可见性 = PRIVATE

| 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|------|-----------|-------------------|---------------------|
| 匿名 | ❌ **404** | ❌ **401** | ❌ **403** |
| 非所有者（已认证非 ADMIN） | ❌ **404** | ❌ **403** | ❌ **403** |
| 所有者 | ✅ 200 | ✅ 200 | ✅ 200 |
| ADMIN（非所有者） | ❌ **404** | ✅ **200** | ❌ **403** |

**ARCHIVED 状态的关键差异汇总：**

| 场景 | API | File Server | MCP | 原因 |
|-----|-----|-------------|-----|------|
| PUBLIC+ARCHIVED+匿名 | 404 | 200 | 403 | API 隐私保护（404），FS 不检查 ARCHIVED，MCP 检查但返回 403 |
| 非所有者访问任何 ARCHIVED | 404 | 视可见性而定 | 403 | API 隐私保护，FS 不检查 ARCHIVED，MCP 检查但不返回 404 |
| ADMIN 访问他人的 PRIVATE+ARCHIVED | 404 | 200 | 403 | FS 检查 ADMIN 角色，其他不检查 |

---

## 4. 完整的决策流程与状态码映射

### 4.1 API 入口决策树

```
请求进入 API (Connect RPC)
    │
    ▼
AuthInterceptor
    │
    ├── 非公开端点且认证失败 → codes.Unauthenticated → **401**
    │
    └── 认证通过或公开端点
            │
            ▼
    checkMemoReadAccess
            │
            ├── memo.RowStatus == Archived
            │       │
            │       └── 非所有者 → codes.NotFound → **404**
            │
            ├── memo.Visibility != PUBLIC
            │       │
            │       ├── 未认证 → codes.Unauthenticated → **401**
            │       │
            │       └── PRIVATE 且非所有者 → codes.PermissionDenied → **403**
            │
            └── 其他 → **200**
```

### 4.2 File Server 入口决策树

```
请求进入 File Server
    │
    ▼
checkAttachmentPermission
    │
    ├── MemoID == nil (未关联)
    │       │
    │       ├── 未认证 → http.StatusUnauthorized → **401**
    │       │
    │       └── 非所有者且非 ADMIN → http.StatusForbidden → **403**
    │
    ├── memo == nil (memo 已删除)
    │       │
    │       └── http.StatusNotFound → **404**
    │
    ├── memo.Visibility == PUBLIC
    │       │
    │       └── **200** (⚠️ 不检查 ARCHIVED)
    │
    ├── 有有效 share_token
    │       │
    │       └── **200**
    │
    ├── 未认证
    │       │
    │       └── http.StatusUnauthorized → **401**
    │
    ├── memo.Visibility == PRIVATE 且非所有者且非 ADMIN
    │       │
    │       └── http.StatusForbidden → **403**
    │
    └── 其他 (PROTECTED 且已认证) → **200**
```

### 4.3 MCP 入口决策树 (get_attachment 工具)

```
匿名调用 get_attachment
    │
    ▼
userID := auth.GetUserID(ctx) = 0
    │
    ▼
获取 attachment
    │
    ├── 不存在 → "attachment not found" → **404**
    │
    └── 存在 → checkAttachmentAccess
            │
            ├── CreatorID == userID (0) → **200** (不可能，userID=0 不是有效用户)
            │
            ├── MemoID == nil (未关联)
            │       │
            │       └── "permission denied" → **403**
            │
            ├── memo == nil → "linked memo not found" → **404**
            │
            └── checkMemoAccess(memo, userID=0)
                    │
                    ├── memo.RowStatus == Archived → "permission denied" → **403**
                    │
                    ├── memo.Visibility == PUBLIC → **200**
                    │
                    ├── memo.Visibility == PROTECTED → "permission denied" → **403**
                    │
                    └── memo.Visibility == PRIVATE → "permission denied" → **403**
```

### 4.4 MCP 入口决策树 (list_attachments 工具)

```
匿名调用 list_attachments
    │
    ▼
extractUserID(ctx)
    │
    └── userID == 0 → "unauthenticated: a personal access token is required" → **401**
    │
    └── 不执行任何数据库查询
```

---

## 5. 最终统一状态码汇总表

### 5.1 归档状态 (ARCHIVED) 完整矩阵

| 资源类型 | memo可见性 | 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|---------|-----------|------|-----------|-------------------|---------------------|
| **Memo 内容** | PUBLIC | 匿名 | ❌ 404 | N/A | ❌ 403 |
| | | 非所有者 | ❌ 404 | N/A | ❌ 403 |
| | | 所有者 | ✅ 200 | N/A | ✅ 200 |
| | | ADMIN（非所有者） | ❌ 404 | N/A | ❌ 403 |
| | PROTECTED | 匿名 | ❌ 404 | N/A | ❌ 403 |
| | | 非所有者 | ❌ 404 | N/A | ❌ 403 |
| | | 所有者 | ✅ 200 | N/A | ✅ 200 |
| | | ADMIN（非所有者） | ❌ 404 | N/A | ❌ 403 |
| | PRIVATE | 匿名 | ❌ 404 | N/A | ❌ 403 |
| | | 非所有者 | ❌ 404 | N/A | ❌ 403 |
| | | 所有者 | ✅ 200 | N/A | ✅ 200 |
| | | ADMIN（非所有者） | ❌ 404 | N/A | ❌ 403 |
| **Memo 附件** | PUBLIC | 匿名 | N/A | ✅ **200** | ❌ **403** |
| | | 非所有者 | N/A | ✅ **200** | ❌ **403** |
| | | 所有者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非所有者） | N/A | ✅ **200** | ❌ **403** |
| | PROTECTED | 匿名 | N/A | ❌ **401** | ❌ **403** |
| | | 非所有者 | N/A | ✅ **200** | ❌ **403** |
| | | 所有者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非所有者） | N/A | ✅ **200** | ❌ **403** |
| | PRIVATE | 匿名 | N/A | ❌ 401 | ❌ 403 |
| | | 非所有者（非 ADMIN） | N/A | ❌ 403 | ❌ 403 |
| | | 所有者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非所有者） | N/A | ✅ **200** | ❌ **403** |
| **未关联附件** | N/A | 匿名 | N/A | ❌ **401** | ❌ **403** |
| | | 非创建者（非 ADMIN） | N/A | ❌ 403 | ❌ 403 |
| | | 创建者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非创建者） | N/A | ✅ **200** | ❌ **403** |

### 5.2 正常状态 (NORMAL) 完整矩阵

| 资源类型 | memo可见性 | 身份 | API (内容) | File Server (附件) | MCP (get_attachment) |
|---------|-----------|------|-----------|-------------------|---------------------|
| **Memo 内容** | PUBLIC | 匿名 | ✅ 200 | N/A | ✅ 200 |
| | | 非所有者 | ✅ 200 | N/A | ✅ 200 |
| | | 所有者 | ✅ 200 | N/A | ✅ 200 |
| | | ADMIN（非所有者） | ✅ 200 | N/A | ✅ 200 |
| | PROTECTED | 匿名 | ❌ **401** | N/A | ❌ **403** |
| | | 非所有者 | ✅ 200 | N/A | ✅ 200 |
| | | 所有者 | ✅ 200 | N/A | ✅ 200 |
| | | ADMIN（非所有者） | ✅ 200 | N/A | ✅ 200 |
| | PRIVATE | 匿名 | ❌ 401 | N/A | ❌ 403 |
| | | 非所有者（非 ADMIN） | ❌ 403 | N/A | ❌ 403 |
| | | 所有者 | ✅ 200 | N/A | ✅ 200 |
| | | ADMIN（非所有者） | ❌ **403** | N/A | ❌ **403** |
| **Memo 附件** | PUBLIC | 匿名 | N/A | ✅ 200 | ✅ 200 |
| | | 非所有者 | N/A | ✅ 200 | ✅ 200 |
| | | 所有者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非所有者） | N/A | ✅ 200 | ✅ 200 |
| | PROTECTED | 匿名 | N/A | ❌ **401** | ❌ **403** |
| | | 非所有者 | N/A | ✅ 200 | ✅ 200 |
| | | 所有者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非所有者） | N/A | ✅ 200 | ✅ 200 |
| | PRIVATE | 匿名 | N/A | ❌ 401 | ❌ 403 |
| | | 非所有者（非 ADMIN） | N/A | ❌ 403 | ❌ 403 |
| | | 所有者 | N/A | ✅ 200 | ✅ 200 |
| | | ADMIN（非所有者） | N/A | ✅ **200** | ❌ **403** |

---

## 6. 关键代码位置速查

### 6.1 MCP 附件工具认证函数

| 工具 | 文件路径 | 函数/位置 | 认证函数 |
|-----|---------|----------|---------|
| **list_attachments** | `tools_attachment.go:138-207` | `handleListAttachments` | `extractUserID(ctx)` → 401 |
| **get_attachment** | `tools_attachment.go:209-234` | `handleGetAttachment` | `auth.GetUserID(ctx)` → 403 |
| **delete_attachment** | `tools_attachment.go:236-250` | `handleDeleteAttachment` | `extractUserID(ctx)` → 401 |
| **link_attachment_to_memo** | `tools_attachment.go:252-327` | `handleLinkAttachmentToMemo` | `extractUserID(ctx)` → 401 |

### 6.2 认证函数实现

| 函数 | 文件路径 | 匿名行为 |
|-----|---------|---------|
| `extractUserID(ctx)` | `tools_memo.go:172-177` | 返回 `"unauthenticated: ..."` → **401** |
| `auth.GetUserID(ctx)` | `auth/context.go:15-20` | 返回 `0`（不报错） |
| `checkAttachmentAccess` | `mcp/access.go:66-82` | 未关联 → `"permission denied"` → **403** |
| `checkMemoAccess` | `mcp/access.go:17-34` | 非所有者 → `"permission denied"` → **403** |

### 6.3 权限检查函数

| 功能 | 文件路径 | 返回码策略 |
|-----|---------|-----------|
| API: `checkMemoReadAccess` | `memo_service.go:42-71` | ARCHIVED → 404, PRIVATE非所有者 → 403 |
| File Server: `checkAttachmentPermission` | `fileserver.go:636-688` | 不检查 ARCHIVED, ADMIN 可访问 PRIVATE |
| MCP: `checkAttachmentAccess` | `mcp/access.go:66-82` | 检查 ARCHIVED, 不检查 ADMIN |
| MCP: `checkMemoAccess` | `mcp/access.go:17-34` | 检查 ARCHIVED, 不检查 ADMIN |

---

## 7. 最终结论

### 7.1 设计意图总结

| 入口 | 设计理念 | 关键特征 |
|-----|---------|---------|
| **API** | **隐私优先** | ARCHIVED 返回 404（隐藏存在性），ADMIN 不能读取他人 PRIVATE |
| **File Server** | **灵活 + 后门** | 不检查 ARCHIVED，ADMIN 可访问 PRIVATE 附件，支持 share_token |
| **MCP** | **最小权限 + AI 安全** | 不检查 ADMIN，ARCHIVED 返回 403（不隐藏存在性） |

### 7.2 所有不一致点（最终定稿）

| 不一致场景 | API | File Server | MCP (get_attachment) | 影响 |
|-----------|-----|-------------|---------------------|------|
| **ARCHIVED 检查** | ✅ 检查 → 404 | ❌ 不检查 → 按可见性 | ✅ 检查 → 403 | 归档附件可能泄露 |
| **ARCHIVED 返回码** | 404（隐私保护） | 不适用 | 403（明确拒绝） | 隐私保护程度不同 |
| **ADMIN 访问 PRIVATE** | ❌ 403 | ✅ 200 | ❌ 403 | ADMIN 权限范围不一致 |
| **匿名访问未关联附件** | N/A | 401 | 403 | 认证检查时机不同 |
| **匿名访问 PROTECTED** | 401 | 401 | 403 | 认证检查时机不同 |
| **MCP 工具差异** | N/A | N/A | `get_attachment`→403, `list_attachments`→401 | 不同工具行为不一致 |

### 7.3 MCP 工具差异的设计意图推测

| 工具 | 认证函数 | 匿名返回码 | 推测原因 |
|-----|---------|-----------|---------|
| **get_attachment** | `auth.GetUserID` | 403 | 允许匿名访问 PUBLIC memo 的附件（与 API/FS 一致） |
| **list_attachments** | `extractUserID` | 401 | 列表操作更敏感，强制认证（列出自己的文件） |
| **delete_attachment** | `extractUserID` | 401 | 写操作，必须认证 |
| **link_attachment_to_memo** | `extractUserID` | 401 | 写操作，必须认证 |

**注意：`get_attachment` 返回 403 而不是 401，可能是为了让匿名用户也能访问 PUBLIC memo 的附件。但实际上在 `checkMemoAccess` 中，PUBLIC memo 的附件确实可以被匿名访问（返回 200）。**

---

## 附录：状态码对照表

### 401 vs 403 语义对比

| HTTP 码 | 语义 | API/FS 触发条件 | MCP 触发条件 |
|--------|------|----------------|-------------|
| **401** | "你是谁？" — 需要认证 | 没有 token、token 无效、匿名访问需要认证的资源 | `extractUserID` 失败（list_attachments, delete_attachment） |
| **403** | "我知道你是谁，但你不能" — 已认证但无权限 | 非所有者访问 PRIVATE、ADMIN 限制 | `checkAttachmentAccess`/`checkMemoAccess` 失败（get_attachment） |
| **404** | "不存在" — 真不存在或被隐藏 | 资源已删除、ARCHIVED 且非所有者（API） | 资源不存在 |

### 各入口的 401/403 触发策略

| 入口 | 401 触发时机 | 403 触发时机 |
|-----|------------|------------|
| **API** | 拦截器检查认证失败、checkMemoReadAccess 中 `user == nil` | checkMemoReadAccess 中 PRIVATE 非所有者 |
| **File Server** | checkAttachmentPermission 中 `user == nil` | checkAttachmentPermission 中非所有者非 ADMIN |
| **MCP (list/delete/link)** | `extractUserID` 中 `id == 0` | 不使用这些函数触发，直接 401 |
| **MCP (get_attachment)** | 不会触发 401 | `checkAttachmentAccess` 中任何权限不足 |
