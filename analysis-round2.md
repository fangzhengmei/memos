# Memos 深度分析（Round 2）

## 问题 1：分享场景下主 memo 附件 vs 评论附件的权限判定链路差异

### 1.1 核心问题

在分享私有/受保护的 memo 时：
- ✅ **主 memo 附件**：携带 `?share_token={token}` 可正常访问
- ❌ **评论附件**：即使携带同样的 `share_token` 也会被拒绝

### 1.2 权限判定链路对比

#### 链路 A：主 memo 附件 — 允许访问

```
请求: GET /file/attachments/{main_memo_attachment_uid}/{filename}?share_token=xyz
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ FileServerService.checkAttachmentPermission()                     │
│ ───────────────────────────────────────────────────────────────   │
│                                                                   │
│  Step 1: attachment.MemoID = 主 memo 的 ID (memo_A)              │
│                                                                   │
│  Step 2: memo = GetMemo(memo_A)                                   │
│          memo.Visibility = PROTECTED (非 PUBLIC)                  │
│                                                                   │
│  Step 3: 检查 share_token 回退机制                                │
│          ┌─────────────────────────────────────────────────┐      │
│          │ shareToken = "xyz"                             │      │
│          │ ms = GetMemoShare(uid="xyz")                   │      │
│          │   → ms.MemoID = memo_A ✅                       │      │
│          │   → !isMemoShareExpired(ms) = true ✅           │      │
│          │   → ms.MemoID == memo.ID ✅                     │      │
│          └─────────────────────────────────────────────────┘      │
│                                                                   │
│  Step 4: 所有条件满足 → return nil (授权成功)                      │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        返回文件内容 (HTTP 200)
```

**关键代码** (`fileserver.go:637-688`):

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, c *echo.Context, attachment *store.Attachment) error {
    // ...
    
    // 附件关联的 memo
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
    
    if memo.Visibility == store.Public {
        return nil  // 公开直接放行
    }

    // ⚠️ 关键：share_token 只针对 attachment 关联的 memo
    if shareToken := (*c).QueryParam("share_token"); shareToken != "" {
        ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
        if err == nil && ms != nil && 
           !isMemoShareExpired(ms) && 
           ms.MemoID == memo.ID {  // ← 这里只匹配 attachment.MemoID
            return nil  // ✅ 匹配成功
        }
    }
    // ...
}
```

#### 链路 B：评论附件 — 拒绝访问

```
请求: GET /file/attachments/{comment_attachment_uid}/{filename}?share_token=xyz
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ FileServerService.checkAttachmentPermission()                     │
│ ───────────────────────────────────────────────────────────────   │
│                                                                   │
│  Step 1: attachment.MemoID = 评论 memo 的 ID (memo_B)            │
│          （注意：评论本身是一个独立的 memo！）                       │
│                                                                   │
│  Step 2: memo = GetMemo(memo_B)                                   │
│          memo.Visibility = PROTECTED (非 PUBLIC)                  │
│                                                                   │
│  Step 3: 检查 share_token 回退机制                                │
│          ┌─────────────────────────────────────────────────┐      │
│          │ shareToken = "xyz"                             │      │
│          │ ms = GetMemoShare(uid="xyz")                   │      │
│          │   → ms.MemoID = memo_A (主 memo 的 ID)          │      │
│          │   → ms.MemoID == memo.B (memo_B) ❌ 不匹配！    │      │
│          └─────────────────────────────────────────────────┘      │
│                                                                   │
│  Step 4: share_token 回退失败                                     │
│          → getCurrentUser() 尝试认证                               │
│          → 未登录 → user == nil                                   │
│                                                                   │
│  Step 5: return HTTP 401 Unauthorized                             │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        返回 401 Unauthorized
```

**测试验证** (`fileserver_test.go:188-192`):

```go
// 明确的测试用例验证评论附件即使携带 share_token 也被拒绝
req := httptest.NewRequest(http.MethodGet, 
    fmt.Sprintf("/file/%s/%s?share_token=%s", 
        commentAttachment.Name, commentAttachment.Filename, shareToken), nil)
rec := httptest.NewRecorder()
e.ServeHTTP(rec, req)

require.Equal(t, http.StatusUnauthorized, rec.Code)  // ✅ 预期是 401
```

### 1.3 深层原因分析

#### 评论的本质：独立 memo + MemoRelation 关联

从 `memo_service.proto:75-87` 和 `memo_service.go:759-910` 可以看出：

```
┌─────────────────────────────────────────────────────────────────┐
│                        评论数据模型                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  主 Memo (memo_A)                                               │
│  ┌────────────────────────┐                                     │
│  │ UID: "main-memo"       │                                     │
│  │ Visibility: PROTECTED  │                                     │
│  │ Parent: null           │                                     │
│  └───────────┬────────────┘                                     │
│              │ MemoRelation.COMMENT                              │
│              ▼                                                  │
│  评论 Memo (memo_B) ── 评论本身也是独立的 memo！                  │
│  ┌────────────────────────┐                                     │
│  │ UID: "comment-123"     │                                     │
│  │ Visibility: PROTECTED  │  ← 有自己独立的 visibility           │
│  │ Parent: "memos/main-memo" │  ← 指向主 memo                  │
│  └───────────┬────────────┘                                     │
│              │                                                  │
│              ▼                                                  │
│  评论附件                                                       │
│  ┌────────────────────────┐                                     │
│  │ UID: "comment-attach"  │                                     │
│  │ MemoID: memo_B.ID      │  ← 关联到评论 memo，不是主 memo     │
│  └────────────────────────┘                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 分享 token 的作用范围

```
MemoShare 记录:
┌─────────────────────────────────────────┐
│ UID: "xyz"         ← share_token       │
│ MemoID: memo_A.ID   ← 只绑定主 memo    │
│ ExpiresTs: null                         │
└─────────────────────────────────────────┘
```

**share_token 只对特定 memo_id 有效**，而评论附件关联的是评论 memo（不是主 memo），所以 token 验证失败。

#### 前端分享时的代码路径

从 `MemoDetail.tsx:51-77` 可以看到：

```typescript
const MemoDetail = () => {
  const shareToken = params.token;
  const isShareMode = !!shareToken;
  
  // 1. 获取主 memo（使用 GetMemoByShare，无需认证）
  const { data: memoFromShare } = useSharedMemo(shareToken ?? "", { enabled: isShareMode });
  
  // 2. ⚠️ 获取评论（使用 ListMemoComments，需要认证）
  const { data: commentsResponse } = useMemoComments(memoName, { enabled: !!memo });
  const comments = commentsResponse?.memos || [];
  
  // 3. ✅ 只重写主 memo 的附件 URL
  const displayMemo = isShareMode
    ? { ...memo, attachments: withShareAttachmentLinks(memo.attachments as Attachment[], shareToken!) }
    : memo;
  
  // 4. ❌ 评论附件没有被重写！
  //    且 ListMemoComments 在未认证时只能返回 PUBLIC 评论
};
```

### 1.4 权限矩阵（分享场景）

| 资源类型 | 关联 memo | share_token 匹配 | 结果 |
|---------|----------|-----------------|------|
| 主 memo 附件 | memo_A (主) | `ms.MemoID == memo_A` ✅ | 200 OK |
| 评论附件（PUBLIC） | memo_B (评论) | `ms.MemoID == memo_B` ❌，但 memo.Visibility=PUBLIC | 200 OK |
| 评论附件（PROTECTED） | memo_B (评论) | `ms.MemoID == memo_B` ❌，且非登录 | 401 Unauthorized |
| 评论附件（PRIVATE） | memo_B (评论) | `ms.MemoID == memo_B` ❌，且非登录 | 401 Unauthorized |
| 未关联附件 | null | 只允许创建者访问 | 401/403 |

### 1.5 设计意图与边界

这种设计是**有意的安全边界**：

1. **最小权限原则**：分享 token 只授予访问特定 memo 资源的权限，不应该自动授予访问所有相关内容的权限
2. **评论独立权限**：评论有自己独立的 `visibility`，评论创建者可能不希望自己的评论随着主 memo 一起被分享
3. **信息泄露防护**：防止通过分享链接间接访问到其他用户的私有评论

---

## 问题 2：externalLink 与 /file 两条通路的鉴权与安全头差异

### 2.1 通路概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          附件 URL 生成逻辑                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  前端: getAttachmentUrl()  (`attachment.ts:3-9`)                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ if (attachment.externalLink) {                                  │   │
│  │   return attachment.externalLink;  // ← 路径 A: externalLink   │   │
│  │ } else {                                                         │   │
│  │   return /file/{name}/{filename}; // ← 路径 B: /file 通路      │   │
│  │ }                                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 路径 A：externalLink — 外部 URL 直通

#### 什么情况下会走 externalLink？

从 `attachment_service.go:417-435`：

```go
func convertAttachmentFromStore(attachment *store.Attachment) *v1pb.Attachment {
    // ...
    if attachment.StorageType == storepb.AttachmentStorageType_EXTERNAL || 
       attachment.StorageType == storepb.AttachmentStorageType_S3 {
        attachmentMessage.ExternalLink = attachment.Reference
    }
    return attachmentMessage
}
```

**触发条件**：
- `StorageType = EXTERNAL`：外部链接（如 `https://example.com/image.png`）
- `StorageType = S3`：S3 存储（`attachment.Reference` 是 S3 预签名 URL）

#### 通路链路

```
externalLink (直接访问外部 URL)
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│                       前端浏览器                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ GET https://s3.amazonaws.com/bucket/key?X-Amz-Signature=… │  │
│  │         ↓                                                 │  │
│  │ ┌─────────────────────────────────────────────────────┐   │  │
│  │ │ ⚠️ 不经过 Memos 后端的任何处理                      │   │  │
│  │ │ ⚠️ 不检查 Memos 的权限                              │   │  │
│  │ │ ⚠️ 不添加 Memos 的安全响应头                        │   │  │
│  │ │ ✅ 但依赖外部服务自身的安全机制                      │   │  │
│  │ └─────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  S3 存储的 externalLink 实际上是预签名 URL：                     │
│  - 有过期时间（由 presign 逻辑控制）                             │
│  - 签名验证由 S3 服务端完成                                      │
│  - 过期后 URL 自动失效                                           │
│                                                                 │
└──────────────────────────────────────────────────────────────────┘
```

#### externalLink 的两个子场景

| 场景 | StorageType | Reference 内容 | 鉴权机制 | 过期控制 |
|------|-------------|----------------|---------|---------|
| 外部链接 | `EXTERNAL` | `https://example.com/image.png` | 完全由外部服务决定 | 无 |
| S3 预签名 | `S3` | `https://s3.../key?X-Amz-...` | S3 签名验证 | 预签名过期时间 |

### 2.3 路径 B：/file 通路 — Memos 本地文件服务

#### 触发条件

当 `StorageType = LOCAL` 或 `StorageType = DATABASE` 时，不设置 `ExternalLink`，前端走 `/file` 通路。

#### 通路链路

```
/file/attachments/{uid}/{filename}
        │
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                    FileServerService (Echo 路由)                   │
│  fileserver.go:121-124:                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ fileGroup := echoServer.Group("/file")                      │  │
│  │ fileGroup.GET("/attachments/:uid/:filename",                │  │
│  │             s.serveAttachmentFile)                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                           │                                       │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Step 1: 鉴权 checkAttachmentPermission()                   │  │
│  │   ┌─────────────────────────────────────────────────────┐  │  │
│  │   │ 1. 检查 attachment.MemoID                          │  │  │
│  │   │ 2. 检查 memo.Visibility                            │  │  │
│  │   │ 3. 检查 share_token 回退                           │  │  │
│  │   │ 4. 检查用户认证 + 角色                              │  │  │
│  │   └─────────────────────────────────────────────────────┘  │  │
│  │                           │                                 │  │
│  │                           ▼                                 │  │
│  │ ┌───────────────────────────────────────────────────────┐  │  │
│  │ │ Step 2: 读取文件内容                                   │  │  │
│  │ │ - LOCAL: 读取本地文件                                 │  │  │
│  │ │ - S3: 从 S3 下载（如果不用预签名 URL）                  │  │  │
│  │ │ - DATABASE: 从数据库 Blob 读取                        │  │  │
│  │ └───────────────────────────────────────────────────────┘  │  │
│  │                           │                                 │  │
│  │                           ▼                                 │  │
│  │ ┌───────────────────────────────────────────────────────┐  │  │
│  │ │ Step 3: MIME 类型清理 sanitizeContentType()           │  │  │
│  │ │ - text/html → application/octet-stream               │  │  │
│  │ │ - text/javascript → application/octet-stream         │  │  │
│  │ └───────────────────────────────────────────────────────┘  │  │
│  │                           │                                 │  │
│  │                           ▼                                 │  │
│  │ ┌───────────────────────────────────────────────────────┐  │  │
│  │ │ Step 4: 设置安全响应头 setSecurityHeaders()          │  │  │
│  │ │ - X-Content-Type-Options: nosniff                   │  │  │
│  │ │ - X-Frame-Options: DENY                             │  │  │
│  │ │ - Content-Security-Policy: default-src 'none'        │  │  │
│  │ └───────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 2.4 两条通路的详细对比

| 维度 | externalLink 通路 | /file 通路 |
|------|------------------|-----------|
| **触发条件** | StorageType = EXTERNAL 或 S3 | StorageType = LOCAL/DATABASE 或 S3（不使用预签名） |
| **请求目标** | 外部 URL（第三方服务器） | Memos 后端 `/file/attachments/:uid/:filename` |
| **Memos 后端鉴权** | ❌ 不经过 | ✅ `checkAttachmentPermission()` 完整检查 |
| **实际鉴权机制** | 外部服务决定（或 S3 预签名） | Memos 后端：visibility + share_token + 用户角色 |
| **MIME 类型清理** | ❌ 外部服务决定 | ✅ `sanitizeContentType()` — 危险类型转 octet-stream |
| **安全响应头** | ❌ 外部服务决定 | ✅ `setSecurityHeaders()` — CSP, X-Frame-Options, nosniff |
| **分享 token 支持** | ❌ 不支持 | ✅ `?share_token=...` 查询参数 |
| **缩略图支持** | ❌ 外部服务决定 | ✅ `?thumbnail=true` |
| **动态视频提取** | ❌ 外部服务决定 | ✅ `?motion=true` (Live Photo) |
| **Range 请求** | ❌ 外部服务决定 | ✅ 媒体流支持 |

### 2.5 公开引用的风险边界

#### 风险矩阵

| 场景 | 通路 | 风险级别 | 说明 |
|------|------|---------|------|
| **主 memo 附件（PUBLIC）** | /file | 🔵 低 | 后端检查 visibility=PUBLIC 后放行，有安全响应头 |
| **主 memo 附件（PROTECTED/PRIVATE）** | /file | 🔵 低 | 需要认证或有效的 share_token，有安全响应头 |
| **评论附件（PUBLIC）** | /file | 🔵 低 | visibility 检查放行 |
| **评论附件（PROTECTED/PRIVATE）** | /file | 🔵 低 | 即使主 memo 已分享，share_token 无效，强制认证 |
| **外部链接附件** | externalLink | 🟡 中 | 不经过 Memos 安全层，依赖外部服务 |
| **S3 预签名 URL** | externalLink | 🟡 中 | 有过期时间，但在有效期内 URL 对任何人都有效 |
| **Markdown 内联外部图片** | 直接 URL | 🟠 中高 | 由前端 Image 组件渲染，经过 rehype-sanitize，但无后端访问控制 |
| **Markdown 内联 /file 图片** | /file | 🔵 低 | 走文件服务通路，有完整鉴权 |

#### 关键风险边界说明

**边界 1：externalLink 绕过 Memos 安全层**

```
StorageType = EXTERNAL 的附件:

前端: <img src="https://evil.com/image.png" />
           │
           ▼
    浏览器直接请求 evil.com
           │
    ┌────────────────────────────────┐
    │ ❌ 不经过 Memos /file 路由      │
    │ ❌ 不检查 Memos 权限            │
    │ ❌ 不添加 CSP/X-Frame-Options   │
    │ ❌ MIME 类型由 evil.com 决定     │
    │    (可能返回 text/html)          │
    └────────────────────────────────┘
    
但仍受 rehype-sanitize 限制：
    ✅ 协议只能是 https:
    ✅ 不能包含内联脚本
```

**边界 2：S3 预签名 URL 的时间窗口**

```
S3 预签名 URL 特点:

┌─────────────────────────────────────────────────┐
│ https://s3.amazonaws.com/bucket/key?            │
│   X-Amz-Algorithm=AWS4-HMAC-SHA256&              │
│   X-Amz-Date=20260510T120000Z&                  │
│   X-Amz-Expires=900&    ← 15 分钟有效期          │
│   X-Amz-Signature=...                           │
└─────────────────────────────────────────────────┘

风险:
- 在 15 分钟内，任何拿到 URL 的人都能访问
- URL 可能被中间人攻击截获（虽然走 HTTPS）
- 过期后失效，但有效期内无法撤销
```

**边界 3：分享不延伸到评论**

```
分享 token 的作用域是精确的 memo_id:

分享 memo_A:
┌─────────────────────────────────────┐
│ MemoShare:                          │
│   UID: xyz                          │
│   MemoID: memo_A                    │
└─────────────────────────────────────┘

可访问:
  ✅ memo_A 的附件（/file/...?share_token=xyz）
  
不可访问:
  ❌ 评论 memo_B 的附件（MemoID = memo_B ≠ memo_A）
  ❌ 其他关联 memo 的附件
```

### 2.6 安全防护的多层覆盖

```
┌────────────────────────────────────────────────────────────────────┐
│                      附件访问的多层防护                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Layer 1: 前端 URL 生成（选择通路）                                  │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ attachment.ts: getAttachmentUrl()                            │ │
│  │ ├─ externalLink? → 直接使用外部 URL                           │ │
│  │ └─ 否则 → /file/{name}/{filename}                            │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│                              ▼                                     │
│  Layer 2: 前端渲染（rehype-sanitize）                              │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ constants.ts: SANITIZE_SCHEMA                                │ │
│  │ ├─ img.src: 只允许 https: 协议                                │ │
│  │ └─ iframe.src: 白名单正则验证                                 │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│                              ▼                                     │
│  Layer 3: 后端鉴权（仅 /file 通路）                                 │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ fileserver.go: checkAttachmentPermission()                   │ │
│  │ ├─ attachment.MemoID 关联检查                                │ │
│  │ ├─ memo.Visibility 检查                                      │ │
│  │ ├─ share_token 回退验证                                      │ │
│  │ └─ 用户认证 + 角色检查                                        │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│                              ▼                                     │
│  Layer 4: 后端安全响应头（仅 /file 通路）                           │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ fileserver.go: setSecurityHeaders()                          │ │
│  │ ├─ X-Content-Type-Options: nosniff                           │ │
│  │ ├─ X-Frame-Options: DENY                                     │ │
│  │ └─ CSP: default-src 'none'                                   │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│                              ▼                                     │
│  Layer 5: MIME 类型清理（仅 /file 通路）                            │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ fileserver.go: sanitizeContentType()                         │ │
│  │ ├─ text/html → application/octet-stream                      │ │
│  │ ├─ text/javascript → application/octet-stream                │ │
│  │ └─ ... (强制下载，防止内联执行)                                │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

externalLink 通路只经过 Layer 1 + Layer 2，
完全绕过 Layer 3、4、5！
```

---

## 关键代码索引

| 分析点 | 文件位置 | 行号范围 |
|-------|---------|---------|
| 附件权限检查核心逻辑 | `server/router/fileserver/fileserver.go` | 637-688 |
| 评论附件拒绝访问测试 | `server/router/fileserver/fileserver_test.go` | 188-192 |
| externalLink 转换条件 | `server/router/api/v1/attachment_service.go` | 417-435 |
| 安全响应头设置 | `server/router/fileserver/fileserver.go` | 741-747 |
| MIME 类型清理 | `server/router/fileserver/fileserver.go` | 707-718 |
| 分享评论列表 API | `server/router/api/v1/memo_service.go` | 759-910 |
| 前端分享模式附件 URL 重写 | `web/src/pages/MemoDetail.tsx` | 73-77 |
| 前端附件 URL 生成 | `web/src/utils/attachment.ts` | 3-17 |
| 公共 API 白名单 | `server/router/api/v1/acl_config.go` | 35 |
