# Memos 深度分析（Round 3）— 事实修正与详细鉴权链路

## 问题 1：ListMemoComments 在分享模式下的真实鉴权路径

### 1.1 完整鉴权链路详解

#### 第一层：ACL 配置 — 全局放行，但服务层有二次检查

```
ACL 配置 (acl_config.go:35):
┌───────────────────────────────────────────────────────────────────┐
│ PublicMethods["/memos.api.v1.MemoService/ListMemoComments"] = {}  │
│                                                                   │
│ 含义: Connect/gRPC-Gateway 拦截器不强制要求认证                   │
│ 但: 服务层仍会做自己的权限检查                                    │
└───────────────────────────────────────────────────────────────────┘
```

#### 第二层：服务层检查主 memo 的可见性

```
ListMemoComments (memo_service.go:759-838):
┌───────────────────────────────────────────────────────────────────┐
│  Step 1: 获取主 memo                                              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ memo, err := s.Store.GetMemo(ctx, &store.FindMemo{          │ │
│  │     UID: &memoUID                                           │ │
│  │ })                                                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│                              ▼                                    │
│  Step 2: 检查主 memo 访问权限                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ if err := s.checkMemoReadAccess(ctx, memo); err != nil {   │ │
│  │     return nil, err                                         │ │
│  │ }                                                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│                              ▼                                    │
│  Step 3: 根据用户状态过滤评论                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ currentUser, err := s.fetchCurrentUser(ctx)                 │ │
│  │                                                             │ │
│  │ if currentUser == nil {  // 未登录                          │ │
│  │     memoFilter = `visibility == "PUBLIC"`                  │ │
│  │ } else {                   // 已登录                         │ │
│  │     memoFilter = `creator_id == %d ||                       │ │
│  │                  visibility in ["PUBLIC", "PROTECTED"]`     │ │
│  │ }                                                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│                              ▼                                    │
│  Step 4: 按 memoFilter 查询评论 memo                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ memoRelations, err := s.Store.ListMemoRelations(ctx,       │ │
│  │     &store.FindMemoRelation{                               │ │
│  │         RelatedMemoID: &memo.ID,                           │ │
│  │         Type:          &COMMENT,                           │ │
│  │         MemoFilter:    &memoFilter,  // ← 这里过滤          │ │
│  │     })                                                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

#### checkMemoReadAccess 逻辑 (memo_service.go:42-71)

```go
func (s *APIV1Service) checkMemoReadAccess(ctx context.Context, memo *store.Memo) error {
    // 已归档的 memo 只允许创建者访问
    if memo.RowStatus == store.Archived {
        user, err := s.fetchCurrentUser(ctx)
        if err != nil { /* ... */ }
        if user == nil || memo.CreatorID != user.ID {
            return status.Errorf(codes.NotFound, "memo not found")
        }
    }

    // 非 PUBLIC 的 memo 需要登录
    if memo.Visibility != store.Public {
        user, err := s.fetchCurrentUser(ctx)
        if err != nil { /* ... */ }
        if user == nil {
            return status.Errorf(codes.Unauthenticated, "user not authenticated")
        }
        // PRIVATE 还需要是创建者或管理员
        if memo.Visibility == store.Private && memo.CreatorID != user.ID {
            return status.Errorf(codes.PermissionDenied, "permission denied")
        }
    }
    return nil
}
```

#### fetchCurrentUser 逻辑 (auth_service.go:584-602)

```go
func (s *APIV1Service) fetchCurrentUser(ctx context.Context) (*store.User, error) {
    // 只从 context 中获取 userID，完全不看 share_token
    userID := auth.GetUserID(ctx)
    if userID == 0 {
        return nil, nil  // 未登录
    }
    user, err := s.Store.GetUser(ctx, &store.FindUser{ID: &userID})
    // ...
    return user, nil
}
```

### 1.2 场景矩阵

| 主 memo 可见性 | 用户状态 | ListMemoComments 结果 |
|---------------|---------|----------------------|
| **PUBLIC** | 未登录 | ✅ 通过 checkMemoReadAccess，返回 **PUBLIC 评论** |
| **PUBLIC** | 已登录 | ✅ 通过，返回 **PUBLIC + PROTECTED + 自己的 PRIVATE 评论** |
| **PROTECTED** | 未登录 | ❌ checkMemoReadAccess 失败 → `Unauthenticated` |
| **PROTECTED** | 已登录（任意用户） | ✅ 通过，返回 **PUBLIC + PROTECTED + 自己的 PRIVATE 评论** |
| **PRIVATE** | 未登录 | ❌ `Unauthenticated` |
| **PRIVATE** | 已登录（非创建者） | ❌ `PermissionDenied` |
| **PRIVATE** | 已登录（创建者） | ✅ 通过，返回 **所有评论**（按过滤规则） |

### 1.3 关键发现：share_token 完全不参与 ListMemoComments

```
分享模式下的实际行为:

前端 MemoDetail.tsx:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  GET /api/v1/shares/{token}  → useSharedMemo()                 │
│    └─ 使用 GetMemoByShare 接口                                 │
│    └─ 直接返回 memo 数据（无需登录，通过 share_token 认证）     │
│       └─ ✅ 可获取主 memo 的全部附件                            │
│                                                                │
│  GET /api/v1/memos/{uid}/comments  → useMemoComments()         │
│    └─ 使用 ListMemoComments 接口                               │
│    └─ ACL 放行，但 checkMemoReadAccess 检查 visibility         │
│       └─ fetchCurrentUser() 只看 context 中的 userID           │
│          └─ ❌ 完全不看 share_token！                          │
│             └─ 如果主 memo 是 PROTECTED/PRIVATE:               │
│                ├─ 未登录 → Unauthenticated                    │
│                └─ 已登录（但 share_token 登录不算）→ 可能失败   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**结论**：
- `GetMemoByShare` 是专门的分享接口，通过 share_token 授权
- `ListMemoComments` 走普通的 API 路径，不识别 share_token
- 这意味着：在分享链接页面，如果主 memo 是 PROTECTED/PRIVATE，**评论列表 API 会因为未登录而失败**

---

## 问题 2：externalLink 在三种场景下的处理与安全层分析

### 2.1 withShareAttachmentLinks 的重写逻辑

```
useMemoShareQueries.ts:95-100

export function withShareAttachmentLinks(attachments: Attachment[], token: string): Attachment[] {
  return attachments.map((a) => {
    if (a.externalLink) return a;  // ← 有 externalLink 直接返回，不修改！
    return { 
      ...a, 
      externalLink: `${window.location.origin}/file/${a.name}/${a.filename}?share_token=${encodeURIComponent(token)}` 
    };
  });
}
```

**关键逻辑**：
- 如果 `attachment.externalLink` 已存在：直接返回，**不做任何修改**
- 如果 `attachment.externalLink` 不存在：创建一个指向 `/file/...?share_token=...` 的 externalLink

### 2.2 后端设置 externalLink 的条件

```
attachment_service.go:417-435

func convertAttachmentFromStore(attachment *store.Attachment) *v1pb.Attachment {
    // ...
    
    if attachment.StorageType == storepb.AttachmentStorageType_EXTERNAL || 
       attachment.StorageType == storepb.AttachmentStorageType_S3 {
        attachmentMessage.ExternalLink = attachment.Reference
    }
    
    return attachmentMessage
}
```

**后端设置 externalLink 的条件**：
- `StorageType = EXTERNAL`：外部 URL（如 `https://example.com/image.png`）
- `StorageType = S3`：S3 预签名 URL

**不设置 externalLink 的条件**（走 /file 通路）：
- `StorageType = LOCAL`：本地文件系统
- `StorageType = DATABASE`：数据库 Blob

### 2.3 三种场景的详细分析

#### 场景 A：普通详情页（非分享模式）

```
MemoDetail.tsx:74-76

// 非分享模式：直接使用从 API 返回的 memo
const displayMemo = isShareMode
    ? { ...memo, attachments: withShareAttachmentLinks(...) }
    : memo;  // ← 非分享模式不调用重写函数
```

| 附件 StorageType | externalLink 初始值 | 是否被重写 | 最终 URL 通路 | 经过的安全层 |
|------------------|---------------------|-----------|--------------|-------------|
| EXTERNAL | `https://external.com/...` | ❌ 不重写 | 直接 externalLink | 只经过 rehype-sanitize，绕过后端 3-5 层 |
| S3 | `https://s3.amazonaws.com/...` | ❌ 不重写 | 直接 externalLink | 只经过 rehype-sanitize，绕过后端 3-5 层 |
| LOCAL | `undefined` | ❌ 不重写 | `/file/attachments/{uid}/{filename}` | ✅ 完整 5 层安全 |
| DATABASE | `undefined` | ❌ 不重写 | `/file/attachments/{uid}/{filename}` | ✅ 完整 5 层安全 |

#### 场景 B：分享详情页 — 主 memo 附件

```
MemoDetail.tsx:74-76

// 分享模式：只重写主 memo 的附件
const displayMemo = isShareMode
    ? { ...memo, attachments: withShareAttachmentLinks(memo.attachments, shareToken!) }
    : memo;
```

| 附件 StorageType | externalLink 初始值 | 是否被重写 | 最终 URL | 经过的安全层 |
|------------------|---------------------|-----------|---------|-------------|
| **EXTERNAL** | `https://external.com/...` | **❌ 直接返回** | `https://external.com/...` | 只经过 rehype-sanitize，绕过后端 3-5 层 |
| **S3** | `https://s3.amazonaws.com/...` | **❌ 直接返回** | `https://s3.amazonaws.com/...` | 只经过 rehype-sanitize，绕过后端 3-5 层 |
| **LOCAL** | `undefined` | **✅ 重写** | `/file/attachments/{uid}/{filename}?share_token=xyz` | ✅ 完整 5 层安全 + share_token 授权 |
| **DATABASE** | `undefined` | **✅ 重写** | `/file/attachments/{uid}/{filename}?share_token=xyz` | ✅ 完整 5 层安全 + share_token 授权 |

**关键事实**：
- EXTERNAL 和 S3 类型的附件**即使在分享模式下也保持原 externalLink**
- 重写的 externalLink 实际上是指向 `/file` 通路的 URL（带 share_token）
- 重写后的 externalLink 会触发 `/file` 通路，有完整安全层

#### 场景 C：分享详情页 — 评论附件

```
MemoDetail.tsx:51-54 + 69-77

// 1. 调用 ListMemoComments 获取评论
const { data: commentsResponse } = useMemoComments(memoName, { enabled: !!memo });
const comments = commentsResponse?.memos || [];

// 2. 只重写主 memo 的附件，不重写评论附件！
const displayMemo = isShareMode
    ? { ...memo, attachments: withShareAttachmentLinks(memo.attachments, shareToken!) }
    : memo;

// 3. 评论直接传给 MemoCommentSection
<MemoCommentSection memo={displayMemo} comments={comments} parentPage={locationState?.from} />
```

| 附件 StorageType | externalLink 初始值 | 是否被重写 | 最终 URL | 经过的安全层 | 访问结果 |
|------------------|---------------------|-----------|---------|-------------|---------|
| **EXTERNAL** | `https://external.com/...` | ❌ 不重写 | `https://external.com/...` | 只经过 rehype-sanitize | ✅ 可访问（外部服务决定） |
| **S3** | `https://s3.amazonaws.com/...` | ❌ 不重写 | `https://s3.amazonaws.com/...` | 只经过 rehype-sanitize | ✅ 可访问（S3 签名验证） |
| **LOCAL**（PUBLIC 评论） | `undefined` | ❌ 不重写 | `/file/attachments/{uid}/{filename}` | ✅ 完整 5 层安全 | ✅ 可访问（visibility=PUBLIC） |
| **LOCAL**（PROTECTED 评论） | `undefined` | ❌ 不重写 | `/file/attachments/{uid}/{filename}` | ✅ 完整 5 层安全 | ❌ 401 Unauthorized（需要登录） |
| **LOCAL**（PRIVATE 评论） | `undefined` | ❌ 不重写 | `/file/attachments/{uid}/{filename}` | ✅ 完整 5 层安全 | ❌ 401/403（需要登录且是创建者） |

### 2.4 重写后的 externalLink 实际上走 /file 通路

```
前端:
┌────────────────────────────────────────────────────────────────┐
│  withShareAttachmentLinks 重写 LOCAL 附件:                      │
│                                                                │
│  原始: attachment {                                            │
│          name: "attachments/abc123",                           │
│          filename: "image.png",                                │
│          externalLink: undefined  ← 无 externalLink           │
│        }                                                       │
│                                                                │
│  重写后: attachment {                                          │
│          externalLink: "http://host/file/attachments/abc123/   │
│                          image.png?share_token=xyz"            │
│                      ─────────────────────────────              │
│                      ↑ 这是 /file 通路 URL！                    │
│        }                                                       │
└────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────────┐
│  getAttachmentUrl 调用:                                        │
│                                                                │
│  export const getAttachmentUrl = (attachment) => {             │
│    if (attachment.externalLink) {                              │
│      return attachment.externalLink;  ← 返回 /file 通路 URL    │
│    }                                                           │
│    return `/file/${attachment.name}/${attachment.filename}`;   │
│  };                                                            │
└────────────────────────────────────────────────────────────────┘
                    │
                    ▼
浏览器请求: GET /file/attachments/abc123/image.png?share_token=xyz
                    │
                    ▼
┌────────────────────────────────────────────────────────────────┐
│  后端 FileServerService:                                       │
│                                                                │
│  Step 1: checkAttachmentPermission                            │
│          ├─ attachment.MemoID = 主 memo ID                     │
│          ├─ share_token 验证通过                                │
│          └─ 授权成功                                           │
│                                                                │
│  Step 2: setSecurityHeaders                                    │
│          ├─ X-Content-Type-Options: nosniff                    │
│          ├─ X-Frame-Options: DENY                              │
│          └─ CSP: default-src 'none'                            │
│                                                                │
│  Step 3: sanitizeContentType                                   │
│          └─ 危险 MIME 类型转 octet-stream                       │
└────────────────────────────────────────────────────────────────┘
```

### 2.5 三种场景的安全层对比矩阵

| 场景 | 附件类型 | URL 通路 | Layer 1 URL 生成 | Layer 2 rehype-sanitize | Layer 3 后端鉴权 | Layer 4 安全响应头 | Layer 5 MIME 清理 | 访问控制 |
|------|---------|---------|-----------------|------------------------|-----------------|-------------------|-------------------|---------|
| **普通详情** | EXTERNAL/S3 | externalLink | ✅ | ✅ | ❌ 绕过 | ❌ 绕过 | ❌ 绕过 | 外部服务 |
| **普通详情** | LOCAL/DATABASE | /file | ✅ | ✅ | ✅ | ✅ | ✅ | visibility + 登录 |
| **分享详情 主 memo** | EXTERNAL/S3 | externalLink（原） | ✅ | ✅ | ❌ 绕过 | ❌ 绕过 | ❌ 绕过 | 外部服务 |
| **分享详情 主 memo** | LOCAL/DATABASE | /file（重写为 externalLink） | ✅ | ✅ | ✅ | ✅ | ✅ | share_token |
| **分享详情 评论** | EXTERNAL/S3 | externalLink | ✅ | ✅ | ❌ 绕过 | ❌ 绕过 | ❌ 绕过 | 外部服务 |
| **分享详情 评论** | LOCAL/DATABASE | /file（无 share_token） | ✅ | ✅ | ✅ | ✅ | ✅ | visibility（未登录只 PUBLIC） |

---

## 事实偏差修正总结

### 之前的不准确表述

1. **ListMemoComments 的鉴权**：之前笼统说是"需要认证"，实际是：
   - ACL 层面是 public（不强制认证）
   - 但服务层通过 `checkMemoReadAccess` 检查**主 memo** 的 visibility
   - 对评论的过滤基于 `fetchCurrentUser` 返回的用户状态
   - **share_token 完全不参与该接口**

2. **externalLink 的处理**：之前说"重写后的 externalLink 是外部 URL"，实际是：
   - `withShareAttachmentLinks` 对已有 `externalLink` 的附件**直接返回不修改**
   - 对没有 `externalLink` 的附件（LOCAL/DATABASE），重写后指向 `/file` 通路 URL
   - 重写后的 externalLink 实际上走 `/file` 通路，**有完整安全层**

### 关键澄清

| 问题 | 之前理解 | 修正后的事实 |
|------|---------|------------|
| 分享链接的评论列表 | 可访问 | 取决于主 memo 的 visibility：PUBLIC 可访问，PROTECTED/PRIVATE 需要登录 |
| 主 memo 的 S3/EXTERNAL 附件在分享模式 | 走 /file 通路 | 保持原 externalLink，绕过 Memos 后端安全层 |
| 主 memo 的 LOCAL 附件在分享模式 | 直接 externalLink | 重写为指向 `/file` 的 externalLink，有完整安全层 |
| share_token 对评论 API 的作用 | 隐式参与 | 完全不参与，ListMemoComments 不识别 |

---

## 关键代码索引

| 分析点 | 文件位置 | 行号范围 |
|-------|---------|---------|
| ACL 公共方法定义 | `server/router/api/v1/acl_config.go` | 35 |
| ListMemoComments 实现 | `server/router/api/v1/memo_service.go` | 759-910 |
| checkMemoReadAccess | `server/router/api/v1/memo_service.go` | 42-71 |
| fetchCurrentUser | `server/router/api/v1/auth_service.go` | 584-602 |
| withShareAttachmentLinks | `web/src/hooks/useMemoShareQueries.ts` | 95-100 |
| convertAttachmentFromStore | `server/router/api/v1/attachment_service.go` | 417-435 |
| getAttachmentUrl | `web/src/utils/attachment.ts` | 3-9 |
| MemoDetail 分享逻辑 | `web/src/pages/MemoDetail.tsx` | 73-77 |
| AttachmentCard URL 选择 | `web/src/components/MemoMetadata/Attachment/AttachmentCard.tsx` | 13 |
