# Memos 认证与权限边界详细分析 (R3)

## 修订说明

本文档是 `auth-permission-boundary-r2.md` 的补充修订版，重点补充和统一：

1. **归档资源的附件访问边界**：当备忘录为 ARCHIVED 状态时，在三种可见性（PUBLIC、PROTECTED、PRIVATE）下，不同请求方（匿名、普通用户、ADMIN、携带 share_token）的返回码（200/401/403/404）
2. **结论依据**：明确分析底层 store 层的查询逻辑和各入口的权限检查差异
3. **统一校正**：与 API 和 MCP 入口不一致的描述已修正

---

## 1. 底层查询机制分析

### 1.1 核心发现：RowStatus 的默认过滤行为

**关键代码位置：** `store/db/sqlite/memo.go:54-144`

`ListMemos`（以及通过它实现的 `GetMemo`）的查询逻辑：

```go
func (d *DB) ListMemos(ctx context.Context, find *store.FindMemo) ([]*store.Memo, error) {
    where, args := []string{"1 = 1"}, []any{}
    
    // ... 其他过滤条件 ...
    
    // ⚠️ 关键：只有显式指定 RowStatus 时才会过滤
    if v := find.RowStatus; v != nil {
        where, args = append(where, "`memo`.`row_status` = ?"), append(args, *v)
    }
    
    // ...
}
```

**结论：**
- `GetMemo(&FindMemo{ID: &id})` → `find.RowStatus == nil` → **不会过滤 ARCHIVED**
- 这意味着归档的备忘录可以被数据库查询到

### 1.2 不同入口的查询参数对比

| 入口 | 调用位置 | 查询参数 | 是否过滤 ARCHIVED |
|-----|---------|---------|------------------|
| **File Server** | `fileserver.go:653` | `GetMemo({ID: attachment.MemoID})` | ❌ 不过滤 |
| **API (GetMemo)** | `memo_service.go:366-368` | `GetMemo({UID: memoUID})` → `checkMemoReadAccess` | ❌ 数据库不过滤，但服务层检查 |
| **MCP** | `access.go:74` | `GetMemo({ID: attachment.MemoID})` | ❌ 数据库不过滤，但服务层检查 |

---

## 2. 文件服务器中归档备忘录附件的访问边界

### 2.1 文件服务器权限检查流程

**核心函数：** `server/router/fileserver/fileserver.go:636-688`

```
附件访问请求 (GET /file/attachment/xxx)
    │
    │  1. 检查是否未关联附件 (MemoID == nil)
    │     └── 是 → 需要认证 + 所有者或 ADMIN
    │
    │  2. 查询关联的备忘录
    │     memo = GetMemo({ID: attachment.MemoID})
    │     ⚠️ 注意：不指定 RowStatus，ARCHIVED 也会返回
    │
    │  3. memo == nil?
    │     └── 是 → 404 NotFound (memo 已删除)
    │
    │  4. memo.Visibility == PUBLIC?
    │     └── 是 → 200 OK (不检查 ARCHIVED)
    │
    │  5. 检查 share_token
    │     └── 有效 → 200 OK (绕过所有检查)
    │
    │  6. 获取当前用户
    │     └── 匿名 → 401 Unauthorized
    │
    │  7. memo.Visibility == PRIVATE 且不是所有者且不是 ADMIN?
    │     └── 是 → 403 Forbidden
    │
    │  8. 其他情况 → 200 OK
```

### 2.2 关键发现：文件服务器不检查 ARCHIVED 状态

从代码流程可以看出：
- **第 4 步**：如果 `memo.Visibility == PUBLIC`，直接返回 200，不检查 `RowStatus`
- **第 5 步**：有有效 `share_token`，直接返回 200
- **第 6-7 步**：只检查 `Visibility`，不检查 `RowStatus`

**重要结论：**
- 文件服务器的 `checkAttachmentPermission` **完全不检查 ARCHIVED 状态**
- 这与 API 入口（`checkMemoReadAccess`）不一致
- API 入口会额外检查 `RowStatus == Archived`

### 2.3 归档备忘录附件的访问边界矩阵

**场景：** 备忘录状态为 `ARCHIVED`，不同可见性下的返回码

| 可见性 | 匿名用户 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） | 有效 share_token |
|-------|---------|-------------------|-----------------|-----------------|-----------------|
| **PUBLIC** | ✅ **200** | ✅ **200** | ✅ **200** | ✅ **200** | ✅ **200** |
| **PROTECTED** | ❌ **401** | ✅ **200** | ✅ **200** | ✅ **200** | ✅ **200** |
| **PRIVATE** | ❌ **401** | ❌ **403** | ✅ **200** | ✅ **200** | ✅ **200** |

**结论依据：**

1. **PUBLIC, ARCHIVED → 200**
   - 代码第 661-663 行：`if memo.Visibility == store.Public { return nil }`
   - 不检查 RowStatus，直接允许访问

2. **PROTECTED, ARCHIVED, 匿名 → 401**
   - 代码第 679-681 行：`if user == nil { return 401 }`
   - 可见性非 PUBLIC 时需要认证

3. **PROTECTED, ARCHIVED, 已认证 → 200**
   - 代码第 683-685 行：只检查 PRIVATE 情况
   - PROTECTED 不触发额外检查，直接通过

4. **PRIVATE, ARCHIVED, 所有者/ADMIN → 200**
   - 代码第 683-685 行：`user.ID == memo.CreatorID || user.Role == RoleAdmin` 时通过

5. **PRIVATE, ARCHIVED, 非所有者非 ADMIN → 403**
   - 代码第 683-685 行：不满足条件时返回 403

6. **任意情况，有效 share_token → 200**
   - 代码第 668-673 行：分享令牌绕过所有权限检查

---

## 3. 与 API 入口的对比分析

### 3.1 API 入口的权限检查

**核心函数：** `server/router/api/v1/memo_service.go:42-71`

```go
func (s *APIV1Service) checkMemoReadAccess(ctx context.Context, memo *store.Memo) error {
    // ============================================
    // ARCHIVED 备忘录：只对创建者可见
    // ⚠️ 显式检查 RowStatus！
    // ============================================
    if memo.RowStatus == store.Archived {
        user, err := s.fetchCurrentUser(ctx)
        if user == nil || memo.CreatorID != user.ID {
            // ❌ 返回 404（隐私保护）
            return status.Errorf(codes.NotFound, "memo not found")
        }
    }

    // ============================================
    // 可见性检查
    // ============================================
    if memo.Visibility != store.Public {
        user, err := s.fetchCurrentUser(ctx)
        if user == nil {
            return status.Errorf(codes.Unauthenticated, "user not authenticated")
        }
        if memo.Visibility == store.Private && memo.CreatorID != user.ID {
            // ⚠️ 不检查 ADMIN！
            return status.Errorf(codes.PermissionDenied, "permission denied")
        }
    }
    return nil
}
```

### 3.2 API 入口（Memo 内容）的 ARCHIVED 访问矩阵

**注意：这是备忘录**内容**的访问，不是附件的访问。

| 可见性 | 匿名用户 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|-------|---------|-------------------|-----------------|-----------------|
| **PUBLIC, ARCHIVED** | ❌ **404** | ❌ **404** | ✅ **200** | ❌ **404** |
| **PROTECTED, ARCHIVED** | ❌ **404** | ❌ **404** | ✅ **200** | ❌ **404** |
| **PRIVATE, ARCHIVED** | ❌ **404** | ❌ **404** | ✅ **200** | ❌ **404** |

**结论依据：**
- 代码第 48-55 行：`RowStatus == Archived` 时，只允许创建者访问
- 不区分可见性，ARCHIVED 的任何备忘录都只对创建者可见
- 返回 404 而不是 403（隐私保护，隐藏存在性）

### 3.3 API 入口 vs File Server：关键差异对比

| 维度 | API 入口 (checkMemoReadAccess) | File Server (checkAttachmentPermission) |
|-----|-------------------------------|--------------------------------------|
| **检查 ARCHIVED** | ✅ 显式检查 `RowStatus == Archived` | ❌ **不检查** |
| **PUBLIC + ARCHIVED** | ❌ 404（仅所有者可访问） | ✅ 200（任何人可访问） |
| **ARCHIVED 的 ADMIN 访问** | ❌ 404（仅所有者可访问） | ✅ PRIVATE: 200（ADMIN 可访问） |
| **返回码类型** | 404（隐私保护） | 401/403（标准权限） |
| **share_token 支持** | N/A (MemoService 不处理分享) | ✅ 支持，绕过所有检查 |
| **未关联附件** | N/A | 所有者或 ADMIN 可访问 |

---

## 4. 与 MCP 入口的对比分析

### 4.1 MCP 入口的权限检查

**核心函数：** `server/router/mcp/access.go:15-35`

```go
func checkMemoAccess(memo *store.Memo, userID int32) error {
    // userID == 0 表示匿名
    
    // ============================================
    // ARCHIVED：只对创建者可见
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
        // PRIVATE：只对创建者可见
        // ⚠️ 不检查 ADMIN！
        if memo.CreatorID != userID {
            return errors.New("permission denied")
        }
    default:
        // PUBLIC：允许
    }
    return nil
}
```

### 4.2 MCP 入口的 ARCHIVED 访问矩阵

| 可见性 | 匿名 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|-------|------|-------------------|-----------------|-----------------|
| **PUBLIC, ARCHIVED** | ❌ "permission denied" | ❌ "permission denied" | ✅ 成功 | ❌ "permission denied" |
| **PROTECTED, ARCHIVED** | ❌ "permission denied" | ❌ "permission denied" | ✅ 成功 | ❌ "permission denied" |
| **PRIVATE, ARCHIVED** | ❌ "permission denied" | ❌ "permission denied" | ✅ 成功 | ❌ "permission denied" |

**结论依据：**
- 代码第 18-20 行：`RowStatus == Archived` 且 `CreatorID != userID` 时拒绝
- 不区分可见性，ARCHIVED 的任何备忘录都只对创建者可见
- 返回简单错误消息 "permission denied"，不是结构化错误码
- **与 API 相同：ARCHIVED 只对所有者可见**
- **与 API 相同：不检查 ADMIN 角色**

### 4.3 MCP 附件访问

**核心函数：** `server/router/mcp/access.go:66-82`

```go
func (s *MCPService) checkAttachmentAccess(ctx context.Context, attachment *store.Attachment, userID int32) error {
    // 1. 所有者可访问
    if attachment.CreatorID == userID {
        return nil
    }
    
    // 2. 未关联附件：只有所有者可访问
    if attachment.MemoID == nil {
        return errors.New("permission denied")
    }
    
    // 3. 已关联附件：检查 memo 权限
    memo, err := s.store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
    if memo == nil {
        return errors.New("linked memo not found")
    }
    return checkMemoAccess(memo, userID)  // 调用上面的函数
}
```

**MCP 附件的 ARCHIVED 访问矩阵：**

| 可见性 | 匿名 | 普通用户（非所有者） | 普通用户（所有者） | ADMIN（非所有者） |
|-------|------|-------------------|-----------------|-----------------|
| **PUBLIC, ARCHIVED** | ❌ "permission denied" | ❌ "permission denied" | ✅ 成功 | ❌ "permission denied" |
| **PROTECTED, ARCHIVED** | ❌ "permission denied" | ❌ "permission denied" | ✅ 成功 | ❌ "permission denied" |
| **PRIVATE, ARCHIVED** | ❌ "permission denied" | ❌ "permission denied" | ✅ 成功 | ❌ "permission denied" |

**结论：MCP 附件访问与 MCP memo 访问规则一致，ARCHIVED 只对所有者可见。**

---

## 5. 完整的三入口对比矩阵

### 5.1 归档备忘录内容访问矩阵

| 可见性 | 入口 | 匿名 | 普通用户（非所有者） | 所有者 | ADMIN（非所有者） |
|-------|-----|------|-------------------|-------|-----------------|
| **PUBLIC, ARCHIVED** | API (内容) | 404 | 404 | 200 | 404 |
| | MCP (内容) | "permission denied" | "permission denied" | 成功 | "permission denied" |
| | File Server | N/A（内容不是文件） | N/A | N/A | N/A |
| **PROTECTED, ARCHIVED** | API (内容) | 404 | 404 | 200 | 404 |
| | MCP (内容) | "permission denied" | "permission denied" | 成功 | "permission denied" |
| | File Server | N/A | N/A | N/A | N/A |
| **PRIVATE, ARCHIVED** | API (内容) | 404 | 404 | 200 | 404 |
| | MCP (内容) | "permission denied" | "permission denied" | 成功 | "permission denied" |
| | File Server | N/A | N/A | N/A | N/A |

### 5.2 归档备忘录附件访问矩阵

| 可见性 | 入口 | 匿名 | 普通用户（非所有者） | 所有者 | ADMIN（非所有者） | share_token |
|-------|-----|------|-------------------|-------|-----------------|-------------|
| **PUBLIC, ARCHIVED** | File Server | ✅ 200 | ✅ 200 | ✅ 200 | ✅ 200 | ✅ 200 |
| | MCP (附件) | ❌ "denied" | ❌ "denied" | ✅ 成功 | ❌ "denied" | N/A |
| **PROTECTED, ARCHIVED** | File Server | ❌ 401 | ✅ 200 | ✅ 200 | ✅ 200 | ✅ 200 |
| | MCP (附件) | ❌ "denied" | ❌ "denied" | ✅ 成功 | ❌ "denied" | N/A |
| **PRIVATE, ARCHIVED** | File Server | ❌ 401 | ❌ 403 | ✅ 200 | ✅ 200 | ✅ 200 |
| | MCP (附件) | ❌ "denied" | ❌ "denied" | ✅ 成功 | ❌ "denied" | N/A |

**关键差异（用颜色标记）：**

- **PUBLIC, ARCHIVED, 匿名**：File Server 返回 200，MCP 返回拒绝
- **PUBLIC, ARCHIVED, 非所有者**：File Server 返回 200，MCP 返回拒绝
- **PRIVATE, ARCHIVED, ADMIN 非所有者**：File Server 返回 200，MCP 返回拒绝

### 5.3 未关联附件（MemoID == nil）访问矩阵

| 入口 | 匿名 | 普通用户（非创建者） | 创建者 | ADMIN（非创建者） |
|-----|------|-------------------|-------|-----------------|
| **File Server** | 401 | 403 | 200 | 200 |
| **MCP** | "permission denied" | "permission denied" | 成功 | "permission denied" |

**差异：**
- File Server 允许 ADMIN 访问任何未关联附件
- MCP 只允许创建者访问未关联附件

---

## 6. 设计意图与不一致性分析

### 6.1 各入口的设计理念

| 入口 | 设计理念 | 重点 |
|-----|---------|------|
| **API (MemoService)** | **隐私优先** | ARCHIVED 状态隐藏存在性（返回 404），保护用户隐私 |
| **MCP** | **最小权限** | 完全没有 ADMIN 特殊处理，限制 AI 助手的权限范围 |
| **File Server** | **灵活+后门** | 不检查 ARCHIVED，允许 ADMIN 访问 PRIVATE 附件，方便排查问题 |

### 6.2 不一致性汇总

| 不一致点 | API | MCP | File Server |
|---------|-----|-----|-------------|
| **ARCHIVED 检查** | ✅ 检查 | ✅ 检查 | ❌ 不检查 |
| **ADMIN 读取 PRIVATE** | ❌ 不允许 | ❌ 不允许 | ✅ 允许 |
| **ADMIN 修改资源** | ✅ 允许 (canModifyMemo) | ❌ 不允许 | N/A |
| **ARCHIVED 返回码** | 404 (隐私保护) | "permission denied" | 不检查，按可见性返回 |
| **share_token 支持** | N/A | N/A | ✅ 支持 |
| **未关联附件 ADMIN 访问** | N/A | ❌ 不允许 | ✅ 允许 |

### 6.3 潜在安全影响

**File Server 的宽松设计可能导致：**

1. **ARCHIVED 附件泄露**：
   - PUBLIC 且 ARCHIVED 的备忘录附件可以被匿名访问
   - 这意味着归档操作并不能完全"隐藏"附件
   - 用户期望归档后内容不可访问，但实际上附件仍然公开

2. **ADMIN 可访问 PRIVATE 附件**：
   - File Server 允许 ADMIN 访问任何人的 PRIVATE 附件
   - 这可能是有意的（排查违规内容）
   - 但与 API 入口的行为不一致（API 中 ADMIN 不能读取 PRIVATE 内容）

3. **share_token 绕过所有检查**：
   - 分享令牌创建后，即使备忘录被归档，令牌仍然有效
   - 这是有意的设计（分享链接应该持续有效）
   - 但需要注意：分享的是"快照"还是"实时"状态

---

## 7. 关键代码位置速查

### 7.1 归档状态检查

| 功能 | 文件路径 | 函数/位置 | 是否检查 ARCHIVED |
|-----|---------|----------|------------------|
| Memo 内容读取 (API) | `memo_service.go:42-71` | `checkMemoReadAccess` | ✅ 第 48-55 行 |
| Memo 内容读取 (MCP) | `access.go:15-35` | `checkMemoAccess` | ✅ 第 18-20 行 |
| 附件读取 (File Server) | `fileserver.go:636-688` | `checkAttachmentPermission` | ❌ 不检查 |
| 附件读取 (MCP) | `access.go:66-82` | `checkAttachmentAccess` | ✅ 通过 checkMemoAccess |
| ListMemos (API) | `memo_service.go:189-238` | `ListMemos` | ✅ 第 199-207 行 |
| ListMemos (MCP) | `access.go:48-64` | `applyVisibilityFilter` | ✅ 第 49-58 行 |

### 7.2 ADMIN 权限检查

| 功能 | 文件路径 | 函数/位置 | 是否考虑 ADMIN |
|-----|---------|----------|--------------|
| Memo 读取 (API) | `memo_service.go:42-71` | `checkMemoReadAccess` | ❌ 不考虑 |
| Memo 读取 (MCP) | `access.go:15-35` | `checkMemoAccess` | ❌ 不考虑 |
| Memo 修改 (API) | `common.go:81-83` | `canModifyMemo` | ✅ 第 82 行 |
| 附件读取 (File Server) | `fileserver.go:636-688` | `checkAttachmentPermission` | ✅ 第 647, 683 行 |
| 附件读取 (MCP) | `access.go:66-82` | `checkAttachmentAccess` | ❌ 不考虑 |
| 未关联附件 (File Server) | `fileserver.go:636-688` | `checkAttachmentPermission` | ✅ 第 647 行 |
| 未关联附件 (MCP) | `access.go:66-82` | `checkAttachmentAccess` | ❌ 不考虑 |

### 7.3 Store 层查询

| 功能 | 文件路径 | 函数/位置 | 默认过滤 ARCHIVED |
|-----|---------|----------|------------------|
| GetMemo | `store/memo.go:120-131` | `GetMemo` | ❌ 不指定 RowStatus 不过滤 |
| ListMemos (DB) | `store/db/sqlite/memo.go:54-144` | `ListMemos` | ❌ 只有显式指定才过滤 |
| FindMemo.RowStatus | `store/memo.go:57-82` | `FindMemo` | 默认 nil，不过滤 |

---

## 8. 结论与建议

### 8.1 统一结论

**归档备忘录附件访问的实际行为：**

1. **File Server（实际文件下载）**：
   - PUBLIC + ARCHIVED：**任何人可访问**（200）
   - PROTECTED + ARCHIVED：**已认证用户可访问**（401 → 200）
   - PRIVATE + ARCHIVED：**所有者或 ADMIN 可访问**（401/403 → 200）
   - **完全不检查 ARCHIVED 状态**

2. **API（内容读取）**：
   - 任何可见性 + ARCHIVED：**只对创建者可见**（404）
   - ADMIN 也不能读取他人的 ARCHIVED 备忘录内容

3. **MCP（AI 助手）**：
   - 任何可见性 + ARCHIVED：**只对创建者可见**（"permission denied"）
   - 完全没有 ADMIN 特殊处理

### 8.2 建议（如果需要一致性）

如果希望 File Server 与 API/MCP 保持一致，建议修改 `checkAttachmentPermission`：

```go
// 建议在 checkAttachmentPermission 开头添加：
func (s *FileServerService) checkAttachmentPermission(...) error {
    // ... 未关联附件检查 ...
    
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
    // ...
    
    // 新增：ARCHIVED 备忘录的附件只对创建者或 ADMIN 可见
    if memo.RowStatus == store.Archived {
        user, err := s.getCurrentUser(ctx, c)
        if err != nil {
            return echo.NewHTTPError(http.StatusInternalServerError, "...")
        }
        if user == nil || (memo.CreatorID != user.ID && user.Role != store.RoleAdmin) {
            return echo.NewHTTPError(http.StatusNotFound, "memo not found")
        }
    }
    
    // ... 原有逻辑 ...
}
```

**注意：这是设计决策，不是 bug。当前行为可能是有意的。**

### 8.3 关键代码位置对比总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARCHIVED 附件访问：三入口对比                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  File Server: checkAttachmentPermission (fileserver.go:636-688)        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 流程:                                                            │   │
│  │ 1. MemoID == nil → 所有者或 ADMIN 可访问                        │   │
│  │ 2. GetMemo({ID: ...}) ← 不指定 RowStatus，ARCHIVED 会返回       │   │
│  │ 3. memo.Visibility == PUBLIC → 200 (不检查 ARCHIVED)            │   │
│  │ 4. share_token 有效 → 200                                       │   │
│  │ 5. 匿名 → 401                                                  │   │
│  │ 6. PRIVATE 且非所有者非 ADMIN → 403                             │   │
│  │ 7. 其他 → 200                                                  │   │
│  │                                                                 │   │
│  │ 结论: 不检查 ARCHIVED，按可见性处理                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  API: checkMemoReadAccess (memo_service.go:42-71)                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 流程:                                                            │   │
│  │ 1. memo.RowStatus == Archived → 只允许创建者 → 否则 404          │   │
│  │ 2. memo.Visibility != PUBLIC → 匿名 401                         │   │
│  │ 3. PRIVATE 且非所有者 → 403 (不检查 ADMIN)                       │   │
│  │                                                                 │   │
│  │ 结论: ARCHIVED 只对所有者可见，不检查 ADMIN                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  MCP: checkMemoAccess (access.go:15-35)                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 流程:                                                            │   │
│  │ 1. memo.RowStatus == Archived 且非所有者 → "permission denied"  │   │
│  │ 2. PROTECTED 且匿名 → "permission denied"                        │   │
│  │ 3. PRIVATE 且非所有者 → "permission denied"                      │   │
│  │                                                                 │   │
│  │ 结论: ARCHIVED 只对所有者可见，完全没有 ADMIN 特殊处理            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
