# Memos Iframe 嵌入与权限控制分析报告

## 一、概述

Memos 支持两种与 iframe 相关的场景：
1. **在 Memo 内容中嵌入外部 iframe**：例如 YouTube、Vimeo 视频等（通过 TrustedIframe 组件）
2. **将 Memo 以 iframe 方式嵌入外部页面**：通过分享链接机制实现（MemoShare）

本文档重点分析第二种场景：**跨域 iframe 嵌入时的服务端 token 校验和内容权限控制机制**。

---

## 二、Memo 分享链接机制

### 2.1 核心数据结构

分享链接的核心数据结构定义在 `store/memo_share.go`：

```go
type MemoShare struct {
    ID        int32
    UID       string      // 分享 token，使用 shortuuid 生成
    MemoID    int32       // 关联的 memo ID
    CreatorID int32       // 创建者 ID
    CreatedTs int64       // 创建时间戳
    ExpiresTs *int64      // 过期时间戳，nil 表示永不过期
}
```

**Token 生成方式**：
- 使用 `shortuuid/v4` 库生成 URL 安全的 token
- 格式：base57 编码的 UUID v4，共 22 字符
- 熵值：122-bit，安全性较高

### 2.2 分享链接 URL 格式

前端生成的分享链接格式（`web/src/hooks/useMemoShareQueries.ts:82-84`）：

```
{origin}/memos/shares/{shareToken}
```

示例：
```
https://memos.example.com/memos/shares/2v9xQ8ZkLmNpR7sT3uW5y
```

---

## 三、服务端 Token 校验流程

### 3.1 API 权限配置

首先，获取分享 memo 的 API 被显式标记为公开方法（`server/router/api/v1/acl_config.go:39`）：

```go
var PublicMethods = map[string]struct{}{
    // ... 其他公开方法
    "/memos.api.v1.MemoService/GetMemoByShare": {},  // 无需认证
}
```

这意味着：
- `GetMemoByShare` API 调用**不需要** Authorization header
- 其他分享相关 API（Create/List/Delete）**需要**认证，且仅限 memo 创建者或管理员

### 3.2 Token 校验逻辑

核心校验逻辑在 `server/router/api/v1/memo_share_service.go:199-208`：

```go
func (s *APIV1Service) getActiveMemoShare(ctx context.Context, shareID string) (*store.MemoShare, error) {
    // 1. 从数据库查询 share 记录
    ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareID})
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get memo share")
    }
    // 2. 检查是否存在且未过期
    if ms == nil || isMemoShareExpired(ms) {
        return nil, status.Errorf(codes.NotFound, "not found")
    }
    return ms, nil
}

func isMemoShareExpired(ms *store.MemoShare) bool {
    return ms.ExpiresTs != nil && time.Now().Unix() > *ms.ExpiresTs
}
```

**校验要点**：
1. **Token 存在性检查**：数据库中必须存在对应的 `MemoShare` 记录
2. **过期检查**：如果设置了 `ExpiresTs`，当前时间必须小于过期时间
3. **信息泄露防护**：无效或过期的 token 都返回 `NOT_FOUND`，不区分具体原因

### 3.3 Memo 内容获取流程

完整的 `GetMemoByShare` 方法（`server/router/api/v1/memo_share_service.go:151-192`）：

```go
func (s *APIV1Service) GetMemoByShare(ctx context.Context, request *v1pb.GetMemoByShareRequest) (*v1pb.Memo, error) {
    // 1. 校验 share token
    ms, err := s.getActiveMemoShare(ctx, request.ShareId)
    if err != nil {
        return nil, err
    }

    // 2. 获取关联的 memo
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: &ms.MemoID})
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get memo")
    }
    
    // 3. 检查 memo 状态：已归档或不存在的 memo 同样返回 NOT_FOUND
    if memo == nil || memo.RowStatus == store.Archived {
        return nil, status.Errorf(codes.NotFound, "not found")
    }

    // 4. 加载关联数据：reactions、attachments、relations
    reactions, _ := s.Store.ListReactions(ctx, ...)
    attachments, _ := s.Store.ListAttachments(ctx, ...)
    relations, _ := s.batchConvertMemoRelations(ctx, ...)

    // 5. 转换为 protobuf 消息返回
    return s.convertMemoFromStore(ctx, memo, reactions, attachments, relations)
}
```

**关键设计**：
- 已归档的 memo 无法通过分享链接访问
- 返回内容包括：memo 正文、评论、附件、关联关系等

---

## 四、附件访问权限控制

### 4.1 附件 URL 重写

前端在分享模式下会重写附件 URL，添加 `share_token` 参数（`web/src/hooks/useMemoShareQueries.ts:96-100`）：

```typescript
export function withShareAttachmentLinks(attachments: Attachment[], token: string): Attachment[] {
    return attachments.map((a) => {
        if (a.externalLink) return a;
        return { 
            ...a, 
            externalLink: `${window.location.origin}/file/${a.name}/${a.filename}?share_token=${encodeURIComponent(token)}` 
        };
    });
}
```

**重写后的 URL 格式**：
```
{origin}/file/attachments/{attachmentUID}/{filename}?share_token={shareToken}
```

### 4.2 服务端附件权限校验

文件服务器的权限校验逻辑（`server/router/fileserver/fileserver.go:636-688`）：

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, c *echo.Context, attachment *store.Attachment) error {
    // 情况1：未关联到任何 memo 的附件 → 仅创建者或管理员可访问
    if attachment.MemoID == nil {
        user, err := s.getCurrentUser(ctx, c)
        if err != nil || user == nil {
            return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
        }
        if user.ID != attachment.CreatorID && user.Role != store.RoleAdmin {
            return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
        }
        return nil
    }

    // 获取关联的 memo
    memo, _ := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
    if memo == nil {
        return echo.NewHTTPError(http.StatusNotFound, "memo not found")
    }

    // 情况2：memo 是公开的 → 任何人可访问
    if memo.Visibility == store.Public {
        return nil
    }

    // 情况3：非公开 memo，但提供了有效的 share_token
    if shareToken := (*c).QueryParam("share_token"); shareToken != "" {
        ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
        // 检查：token 存在、未过期、且属于该 memo
        if err == nil && ms != nil && !isMemoShareExpired(ms) && ms.MemoID == memo.ID {
            return nil
        }
    }

    // 情况4：需要用户认证
    user, err := s.getCurrentUser(ctx, c)
    if err != nil || user == nil {
        return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
    }

    // 私有 memo → 仅创建者或管理员可访问
    if memo.Visibility == store.Private && user.ID != memo.CreatorID && user.Role != store.RoleAdmin {
        return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
    }

    return nil
}
```

**权限决策树**：
```
附件访问请求
    │
    ├─── 未关联 memo? ──→ 仅创建者/管理员可访问
    │
    └─── 已关联 memo
            │
            ├─── memo 是公开的? ──→ 允许访问
            │
            ├─── 提供了 share_token?
            │       │
            │       ├─── token 有效且属于该 memo? ──→ 允许访问
            │       │
            │       └─── 无效或不属于 ──→ 继续检查
            │
            └─── 需要用户认证
                    │
                    ├─── 未认证 ──→ 401 Unauthorized
                    │
                    └─── 已认证
                            │
                            ├─── memo 是私有的
                            │       │
                            │       ├─── 是创建者或管理员? ──→ 允许访问
                            │       │
                            │       └─── 否则 ──→ 403 Forbidden
                            │
                            └─── memo 是 protected (非公开非私有) ──→ 允许访问
```

---

## 五、跨域 (CORS) 处理

### 5.1 API 层 CORS 配置

API 层有两套不同的 CORS 策略（`server/router/api/v1/v1.go:130-165`）：

#### gRPC-Gateway（/api/v1/* 路由）

```go
gwGroup.Use(middleware.CORSWithConfig(middleware.CORSConfig{
    AllowOrigins: []string{"*"},  // 允许所有来源
}))
```

**特点**：
- `Access-Control-Allow-Origin: *`
- 适用于 `/api/v1/*` 路由
- 这意味着 **GetMemoByShare API 可以从任何域名调用**

#### Connect 处理器（/memos.api.v1.* 路由）

```go
corsHandler := middleware.CORSWithConfig(middleware.CORSConfig{
    UnsafeAllowOriginFunc: func(c *echo.Context, origin string) (string, bool, error) {
        // 仅允许同源或配置的 InstanceURL
        if strings.EqualFold(originURL.Host, c.Request().Host) {
            return origin, true, nil
        }
        // 检查是否匹配配置的 InstanceURL
        instanceURL, _ := url.Parse(s.Profile.InstanceURL)
        return strings.EqualFold(originURL.Scheme, instanceURL.Scheme) && 
               strings.EqualFold(originURL.Host, instanceURL.Host), true, nil
    },
    AllowCredentials: true,
})
```

**特点**：
- 仅允许同源或配置的 `InstanceURL`
- 允许携带 credentials（cookies）
- 浏览器客户端主要使用这套

### 5.2 前端 SPA 页面的 CORS

前端 SPA 页面（包括分享页面 `/memos/shares/:token`）由 `frontend` 服务提供，**没有显式的 CORS 限制**。

这意味着：
- 外部页面可以通过 `<iframe src="https://memos.example.com/memos/shares/xxx">` 嵌入分享页面
- 但嵌入后，iframe 内的 API 调用会受 API 层 CORS 策略影响

---

## 六、安全头部与 iframe 嵌入限制

### 6.1 文件服务器的安全头部

文件服务器设置了严格的安全头部（`server/router/fileserver/fileserver.go:742-747`）：

```go
func setSecurityHeaders(c *echo.Context) {
    h := c.Response().Header()
    h.Set("X-Content-Type-Options", "nosniff")
    h.Set("X-Frame-Options", "DENY")  // 关键：禁止 iframe 嵌入
    h.Set("Content-Security-Policy", "default-src 'none'; style-src 'unsafe-inline';")
}
```

**影响**：
- `X-Frame-Options: DENY` 会阻止浏览器在 iframe 中渲染文件服务器的响应
- 这意味着 **附件文件无法直接通过 iframe 嵌入**
- 但 SPA 前端页面不受此限制（该头部仅在文件服务中设置）

### 6.2 实际嵌入场景

可行的嵌入方式：
1. **嵌入整个分享页面**（推荐）：
   ```html
   <iframe src="https://memos.example.com/memos/shares/2v9xQ8ZkLmNpR7sT3uW5y"></iframe>
   ```
   - 这会加载完整的 SPA 应用
   - 前端会调用 `GetMemoByShare` API 获取内容
   - 附件通过带 `share_token` 的 URL 加载

2. **直接调用 API**（适合自定义 UI）：
   ```javascript
   // 从任意域名调用（受 gRPC-Gateway CORS 策略允许）
   const response = await fetch('https://memos.example.com/api/v1/memos:byShare?shareId=xxx');
   const memo = await response.json();
   ```

---

## 七、前端实现分析

### 7.1 路由配置

分享页面的路由定义（`web/src/router/index.tsx:94`）：

```tsx
{ path: "memos/shares/:token", element: <MemoDetail /> }
```

**注意**：该路由**不在** `RequireAuthRoute` 守卫下，访问时不需要登录。

### 7.2 MemoDetail 页面的分享模式

页面会自动检测是否为分享模式（`web/src/pages/MemoDetail.tsx:26-76`）：

```tsx
const MemoDetail = () => {
    const params = useParams();
    const shareToken = params.token;
    const isShareMode = !!shareToken;

    // 根据模式选择不同的查询方式
    const memoNameFromParams = params.uid ? `${memoNamePrefix}${params.uid}` : "";
    const { data: memoFromDirect, ... } = useMemo(memoNameFromParams, { enabled: !isShareMode });
    const { data: memoFromShare, ... } = useSharedMemo(shareToken ?? "", { enabled: isShareMode });

    const memo = isShareMode ? memoFromShare : memoFromDirect;

    // 分享模式下重写附件 URL
    const displayMemo = isShareMode
        ? { ...memo, attachments: withShareAttachmentLinks(memo.attachments, shareToken!) }
        : memo;
    
    // ...
}
```

### 7.3 分享模式下的错误处理

如果 token 无效或过期，前端会重定向到 404（`web/src/pages/MemoDetail.tsx:62-67`）：

```tsx
if (isShareMode) {
    const isNotFound = error instanceof ConnectError && 
        (error.code === Code.NotFound || error.code === Code.Unauthenticated);
    if (isNotFound || (!isLoading && !memo)) {
        return <Navigate to="/404" replace />;
    }
}
```

---

## 八、权限控制架构总结

```
┌─────────────────────────────────────────────────────────────────┐
│                        外部页面 (iframe 嵌入)                     │
│  <iframe src="https://memos.example.com/memos/shares/TOKEN">   │
└────────────────────────────────────┬────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Memos 前端 SPA (分享页面)                      │
│  路由: /memos/shares/:token                                      │
│  - 无需登录                                                        │
│  - 调用 GetMemoByShare API (公开)                                 │
│  - 附件 URL 自动添加 share_token 参数                             │
└────────────────────────────────────┬────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   GetMemoByShare    │  │   附件访问 (/file/*) │  │   其他 API (如评论)  │
│   /api/v1/memos:    │  │                     │  │                     │
│   byShare           │  │                     │  │                     │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│ CORS: AllowOrigins  │  │ CORS: AllowOrigins  │  │ 需认证，受 Connect   │
│: ["*"] (任意域名)    │  │: ["*"] (任意域名)    │  │ CORS 策略限制        │
├─────────────────────┤  ├─────────────────────┤  └─────────────────────┘
│ 权限校验:            │  │ 权限校验:            │
│ 1. Token 存在性      │  │ 1. 若 memo 是公开   │
│ 2. Token 未过期      │  │    → 允许访问        │
│ 3. Memo 未归档       │  │ 2. 否则检查          │
│                     │  │    share_token       │
│ 返回: Memo 完整数据   │  │                     │
└─────────────────────┘  └─────────────────────┘
```

---

## 九、安全特性与建议

### 9.1 已实现的安全特性

| 特性 | 实现位置 | 说明 |
|------|----------|------|
| Token 高熵值 | `shortuuid.New()` | 122-bit 熵，暴力破解困难 |
| 过期机制 | `ExpiresTs` 字段 | 支持临时分享链接 |
| 信息泄露防护 | 统一返回 `NOT_FOUND` | 不区分"token 不存在"和"已过期" |
| 归档 memo 不可访问 | `RowStatus == Archived` 检查 | 已删除的内容无法通过分享链接访问 |
| 附件权限校验 | `checkAttachmentPermission` | 非公开 memo 的附件需要 token |
| API 层 CORS | gRPC-Gateway 允许 `*` | 方便外部集成，但需注意 |
| 文件防嵌入 | `X-Frame-Options: DENY` | 附件文件无法被 iframe 嵌入 |

### 9.2 潜在风险与建议

#### 风险 1：分享链接可被猜测？
- **现状**：使用 22 字符 shortuuid，熵值足够
- **评估**：低风险，2^122 种组合，暴力破解不可行

#### 风险 2：无撤销后立即失效机制？
- **现状**：撤销是删除数据库记录，立即生效
- **评估**：安全，删除后立即无法访问

#### 风险 3：CORS 策略过于宽松？
- **现状**：gRPC-Gateway 允许所有来源调用 `GetMemoByShare`
- **评估**：中等风险，需注意：
  - 这是设计选择，方便外部集成
  - 但恶意网站可以在用户不知情的情况下访问已知的分享 token
  - 建议：如果需要更严格的控制，可考虑添加 Referer 检查或自定义 CORS 策略

#### 风险 4：分享页面无 Clickjacking 防护？
- **现状**：SPA 页面未设置 `X-Frame-Options` 或 CSP `frame-ancestors`
- **评估**：中等风险
- **建议**：考虑在前端服务中添加安全头部，或允许用户配置是否允许 iframe 嵌入

---

## 十、关键代码文件索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| MemoShare 数据结构 | `store/memo_share.go` | 5-13 |
| 公开 API 配置 | `server/router/api/v1/acl_config.go` | 39 |
| GetMemoByShare 实现 | `server/router/api/v1/memo_share_service.go` | 151-192 |
| Token 校验逻辑 | `server/router/api/v1/memo_share_service.go` | 199-208 |
| 附件权限校验 | `server/router/fileserver/fileserver.go` | 636-688 |
| API 层 CORS 配置 | `server/router/api/v1/v1.go` | 130-165 |
| 文件服务器安全头部 | `server/router/fileserver/fileserver.go` | 742-747 |
| 前端分享路由 | `web/src/router/index.tsx` | 94 |
| 前端分享模式检测 | `web/src/pages/MemoDetail.tsx` | 26-76 |
| 附件 URL 重写 | `web/src/hooks/useMemoShareQueries.ts` | 96-100 |
| 分享链接生成 | `web/src/hooks/useMemoShareQueries.ts` | 82-84 |

---

## 十一、总结

Memos 的 iframe 嵌入机制基于 **分享链接 (MemoShare)** 设计，核心特点：

1. **Token 驱动**：每个分享链接使用高熵的 shortuuid 作为 token
2. **按需过期**：支持设置过期时间，临时分享更安全
3. **权限隔离**：
   - 分享链接仅授予对特定 memo 的只读访问
   - 附件访问需要额外的 token 校验
   - 已归档或删除的内容无法访问
4. **跨域友好**：
   - gRPC-Gateway 层允许任意域名调用 API
   - 前端 SPA 页面可被 iframe 嵌入
5. **信息安全**：
   - 无效/过期 token 统一返回 404，防止信息泄露
   - 附件文件设置 `X-Frame-Options: DENY` 防止被直接嵌入

这种设计平衡了**易用性**（方便外部嵌入和集成）和**安全性**（细粒度的权限控制、高熵 token、过期机制）。
