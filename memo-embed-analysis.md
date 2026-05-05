# Memos Iframe 嵌入与权限控制分析报告

## 一、概述

Memos 支持两种与 iframe 相关的场景：
1. **在 Memo 内容中嵌入外部 iframe**：例如 YouTube、Vimeo 视频等（通过 TrustedIframe 组件）
2. **将 Memo 以 iframe 方式嵌入外部页面**：通过分享链接机制实现（MemoShare）

本文档重点分析第二种场景：**外部页面通过 iframe 嵌入 Memos 分享页面时的服务端 token 校验和内容权限控制机制**。

---

## 二、核心概念澄清：同源 vs 跨域

### 2.1 关键理解

**当外部页面通过 iframe 嵌入 Memos 分享页面时**：

```
外部页面 (https://external.com)
  └── <iframe src="https://memos.example.com/memos/shares/TOKEN">
          │
          └── 页面内的 JavaScript 运行在 https://memos.example.com 上下文中
                  │
                  └── 调用 API: https://memos.example.com/memos.api.v1.MemoService/GetMemoByShare
                          │
                          └── 这是【同源请求】！不是跨域请求
```

**关键点**：
- iframe 内的页面是从 `https://memos.example.com` 加载的
- 页面内的 JavaScript 的 `window.location.origin` 是 `https://memos.example.com`
- 发起的 API 请求目标也是 `https://memos.example.com`
- 浏览器将此视为**同源请求**，不受 CORS 限制

### 2.2 真正的跨域场景

真正需要考虑 CORS 的场景是：**外部页面的 JavaScript 直接调用 Memos API（不通过 iframe）**

```
外部页面 (https://external.com) 的 JavaScript
  └── fetch("https://memos.example.com/api/v1/shares/TOKEN")
          │
          └── 这是【跨域请求】
                  │
                  └── 受 CORS 限制
```

这种场景相对少见，因为：
- 外部页面通常无法获取有效的认证 token
- 分享功能设计为通过 iframe 嵌入整个页面使用

---

## 三、Memo 分享链接机制

### 3.1 核心数据结构

分享链接的核心数据结构定义在 `store/memo_share.go:5-13`：

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

**Token 生成方式**（`server/router/api/v1/memo_share_service.go:56-58`）：
- 使用 `shortuuid/v4` 库生成 URL 安全的 token
- 格式：base57 编码的 UUID v4，共 22 字符
- 熵值：122-bit，安全性较高

### 3.2 分享链接 URL 格式

前端生成的分享链接格式（`web/src/hooks/useMemoShareQueries.ts:82-84`）：

```
{origin}/memos/shares/{shareToken}
```

示例：
```
https://memos.example.com/memos/shares/2v9xQ8ZkLmNpR7sT3uW5y
```

### 3.3 前端 API 调用方式

前端使用 **Connect RPC** 风格调用 API，配置在 `web/src/connect.ts:184-189`：

```typescript
const transport = createConnectTransport({
    baseUrl: window.location.origin,  // 关键：使用当前页面的 origin
    useBinaryFormat: true,
    fetch: fetchWithCredentials,
    interceptors: [authInterceptor],
});
```

**请求路径**：
- Connect RPC 风格：`/memos.api.v1.MemoService/GetMemoByShare`
- 不是 gRPC-Gateway 的 REST 风格：`/api/v1/shares/{share_id}`

---

## 四、服务端 Token 校验流程

### 4.1 API 权限配置

获取分享 memo 的 API 被显式标记为公开方法（`server/router/api/v1/acl_config.go:39`）：

```go
var PublicMethods = map[string]struct{}{
    // ... 其他公开方法
    "/memos.api.v1.MemoService/GetMemoByShare": {},  // 无需认证
}
```

这意味着：
- `GetMemoByShare` API 调用**不需要** Authorization header
- 其他分享相关 API（Create/List/Delete）**需要**认证，且仅限 memo 创建者或管理员

### 4.2 Token 校验逻辑

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

### 4.3 Memo 内容获取流程

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

## 五、附件访问权限控制

### 5.1 附件 URL 重写

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

### 5.2 服务端附件权限校验

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

## 六、CORS 配置分析（修正）

### 6.1 服务端有两套 API 接口

Memos 服务端提供了**两套独立的 API 接口**，分别有不同的 CORS 策略：

| 接口类型 | 路径模式 | CORS 策略 | 主要使用者 |
|---------|---------|----------|-----------|
| Connect RPC | `/memos.api.v1.*` | 严格（同源或 InstanceURL） | 浏览器前端（主应用） |
| gRPC-Gateway (REST) | `/api/v1/*` | 宽松（`AllowOrigins: ["*"]`） | 第三方集成、非浏览器客户端 |

### 6.2 Connect RPC 的 CORS 配置

Connect RPC 使用严格的 CORS 策略（`server/router/api/v1/v1.go:153-163`）：

```go
corsHandler := middleware.CORSWithConfig(middleware.CORSConfig{
    UnsafeAllowOriginFunc: func(c *echo.Context, origin string) (string, bool, error) {
        // 仅允许：
        // 1. 同源（origin 匹配请求的 Host）
        if strings.EqualFold(originURL.Host, c.Request().Host) {
            return origin, true, nil
        }
        // 2. 配置的 InstanceURL
        instanceURL, _ := url.Parse(s.Profile.InstanceURL)
        return strings.EqualFold(originURL.Scheme, instanceURL.Scheme) && 
               strings.EqualFold(originURL.Host, instanceURL.Host), true, nil
    },
    AllowCredentials: true,  // 允许携带 cookies
})
```

**设计目的**：
- 保护需要认证的 API 端点
- 防止 CSRF 攻击
- 允许携带 credentials（cookies）

**与 iframe 嵌入的关系**：
- 当 iframe 内的页面发起请求时，这是**同源请求**
- 浏览器不会发送 `Origin` 头，或发送的 `Origin` 等于目标域名
- 因此**不会触发 CORS 预检**，直接放行

### 6.3 gRPC-Gateway 的 CORS 配置

gRPC-Gateway 使用宽松的 CORS 策略（`server/router/api/v1/v1.go:130-132`）：

```go
gwGroup.Use(middleware.CORSWithConfig(middleware.CORSConfig{
    AllowOrigins: []string{"*"},  // 允许所有来源
}))
```

**设计目的**：
- 允许第三方集成（如 MCP 服务器、移动应用、命令行工具等）
- 提供 RESTful API 接口供非浏览器客户端使用

**与 iframe 嵌入的关系**：
- 前端**不使用**这套接口
- iframe 嵌入场景**不涉及**这套 CORS 配置
- 这是我之前分析的主要错误点

### 6.4 前端实际使用的接口

前端 `connect.ts` 的配置（`web/src/connect.ts:184-189`）：

```typescript
const transport = createConnectTransport({
    baseUrl: window.location.origin,  // 使用当前页面的 origin
    // ...
});
```

结合 `useMemoShareQueries.ts` 的调用（`web/src/hooks/useMemoShareQueries.ts:70`）：

```typescript
const memo = await memoServiceClient.getMemoByShare(create(GetMemoByShareRequestSchema, { shareId }));
```

**实际请求**：
- 目标 URL：`https://memos.example.com/memos.api.v1.MemoService/GetMemoByShare`
- 使用 Connect RPC 格式，不是 REST 格式
- 这是**同源请求**，不受 CORS 限制

---

## 七、Iframe 嵌入的安全考量

### 7.1 文件服务器的安全头部

文件服务器设置了严格的安全头部（`server/router/fileserver/fileserver.go:742-747`）：

```go
func setSecurityHeaders(c *echo.Context) {
    h := c.Response().Header()
    h.Set("X-Content-Type-Options", "nosniff")
    h.Set("X-Frame-Options", "DENY")  // 禁止 iframe 嵌入
    h.Set("Content-Security-Policy", "default-src 'none'; style-src 'unsafe-inline';")
}
```

**影响范围**：
- 仅影响 `/file/*` 路径下的文件访问
- **不影响**前端 SPA 页面
- 附件文件无法直接通过 iframe 嵌入，但可以通过 `<img>`、`<video>` 等标签加载

### 7.2 前端 SPA 页面的嵌入能力

前端页面由 `frontend` 服务提供（`server/router/frontend/frontend.go`），**没有设置**：
- `X-Frame-Options` 头部
- `Content-Security-Policy: frame-ancestors` 指令

**这意味着**：
- 分享页面**可以**被任意域名的 iframe 嵌入
- 这是设计选择，因为分享功能就是为了让外部页面可以嵌入

### 7.3 点击劫持（Clickjacking）风险分析

**潜在风险**：
```
外部页面 (https://malicious.com)
  ├─── 透明层（覆盖在 iframe 上方）
  │         └─── "点击领取红包" 按钮（实际位置对应 iframe 内的某个操作）
  │
  └─── <iframe src="https://memos.example.com/memos/shares/TOKEN">
                └─── 分享页面（只读展示）
```

**实际风险评估**：
- 分享页面是**只读**的，没有敏感操作（如删除、修改、授权等）
- 用户无法在分享页面执行危险操作
- 因此**点击劫持风险较低**

**如果未来分享页面添加了交互功能**：
- 需要考虑添加 `X-Frame-Options: SAMEORIGIN` 或 `Content-Security-Policy: frame-ancestors 'self'`
- 或允许用户配置是否允许 iframe 嵌入

---

## 八、前端实现分析

### 8.1 路由配置

分享页面的路由定义（`web/src/router/index.tsx:94`）：

```tsx
{ path: "memos/shares/:token", element: <MemoDetail /> }
```

**注意**：该路由**不在** `RequireAuthRoute` 守卫下，访问时不需要登录。

### 8.2 MemoDetail 页面的分享模式

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

### 8.3 分享模式下的错误处理

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

## 九、权限控制架构总结（修正）

```
┌─────────────────────────────────────────────────────────────────┐
│                        外部页面 (iframe 嵌入)                     │
│  <iframe src="https://memos.example.com/memos/shares/TOKEN">   │
│  域名: https://external.com                                       │
└────────────────────────────────────┬────────────────────────────┘
                                     │
                                     │ iframe 加载
                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Memos 前端 SPA (分享页面)                      │
│  域名: https://memos.example.com                                  │
│  路由: /memos/shares/:token                                      │
│  - 无需登录                                                        │
│  - 调用 Connect RPC API (同源请求，不受 CORS 限制)                │
│  - 附件 URL 自动添加 share_token 参数                             │
└────────────────────────────────────┬────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   Connect RPC       │  │   附件访问 (/file/*) │  │   其他 API (如评论)  │
│   /memos.api.v1.*   │  │                     │  │                     │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│ CORS: 严格          │  │ CORS: 宽松          │  │ CORS: 严格          │
│ (同源或 InstanceURL) │  │ (AllowOrigins: ["*"]│  │ (同源或 InstanceURL) │
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

## 十、安全特性与建议

### 10.1 已实现的安全特性

| 特性 | 实现位置 | 说明 |
|------|----------|------|
| Token 高熵值 | `shortuuid.New()` | 122-bit 熵，暴力破解困难 |
| 过期机制 | `ExpiresTs` 字段 | 支持临时分享链接 |
| 信息泄露防护 | 统一返回 `NOT_FOUND` | 不区分"token 不存在"和"已过期" |
| 归档 memo 不可访问 | `RowStatus == Archived` 检查 | 已删除的内容无法通过分享链接访问 |
| 附件权限校验 | `checkAttachmentPermission` | 非公开 memo 的附件需要 token |
| 文件防嵌入 | `X-Frame-Options: DENY` | 附件文件无法被 iframe 直接嵌入 |
| Connect CORS 严格 | `UnsafeAllowOriginFunc` | 保护需要认证的 API，防止 CSRF |

### 10.2 潜在风险与建议

#### 风险 1：分享页面可被任意域名 iframe 嵌入
- **现状**：前端页面没有设置 `X-Frame-Options` 或 CSP `frame-ancestors`
- **评估**：低风险，因为分享页面是只读的
- **建议**：
  - 如果未来添加交互功能，考虑添加 `X-Frame-Options: SAMEORIGIN`
  - 或允许用户在实例设置中配置允许的嵌入域名

#### 风险 2：gRPC-Gateway CORS 过于宽松
- **现状**：`/api/v1/*` 路径允许所有来源
- **评估**：中等风险，但这是设计选择
- **分析**：
  - 公开 API（如 `GetMemoByShare`）本来就不需要认证
  - 需要认证的 API 在服务层有额外校验
  - 主要用于第三方集成（MCP、移动应用等）
- **建议**：
  - 保持现状，但需了解其影响
  - 如果需要更严格的控制，可考虑添加 Referer 检查或自定义 CORS 策略

#### 风险 3：分享链接无访问次数限制
- **现状**：一个有效的 token 可以被无限次访问
- **评估**：中等风险
- **建议**：
  - 可考虑添加访问次数限制（可选功能）
  - 或添加访问日志以便追踪滥用

---

## 十一、关键代码文件索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| MemoShare 数据结构 | `store/memo_share.go` | 5-13 |
| 公开 API 配置 | `server/router/api/v1/acl_config.go` | 39 |
| GetMemoByShare 实现 | `server/router/api/v1/memo_share_service.go` | 151-192 |
| Token 校验逻辑 | `server/router/api/v1/memo_share_service.go` | 199-208 |
| 附件权限校验 | `server/router/fileserver/fileserver.go` | 636-688 |
| Connect CORS 配置 | `server/router/api/v1/v1.go` | 153-163 |
| gRPC-Gateway CORS 配置 | `server/router/api/v1/v1.go` | 130-132 |
| 文件服务器安全头部 | `server/router/fileserver/fileserver.go` | 742-747 |
| 前端分享路由 | `web/src/router/index.tsx` | 94 |
| 前端分享模式检测 | `web/src/pages/MemoDetail.tsx` | 26-76 |
| 附件 URL 重写 | `web/src/hooks/useMemoShareQueries.ts` | 96-100 |
| 分享链接生成 | `web/src/hooks/useMemoShareQueries.ts` | 82-84 |
| 前端 Connect 配置 | `web/src/connect.ts` | 184-189 |

---

## 十二、总结

### 12.1 核心修正

我之前的分析存在以下关键错误，现已修正：

| 错误点 | 修正后 |
|--------|--------|
| iframe 内的请求是跨域请求 | iframe 内的请求是**同源请求**，因为页面运行在 memos 域名上下文中 |
| 前端使用 gRPC-Gateway API | 前端使用 **Connect RPC** 格式的 API（`/memos.api.v1.*`） |
| CORS 配置与 iframe 嵌入相关 | CORS 配置主要用于**第三方集成**，iframe 嵌入场景不涉及 |
| 所有 API 路径 CORS 相同 | 有**两套独立**的 API 接口，CORS 策略不同 |

### 12.2 Iframe 嵌入的实际机制

当外部页面通过 iframe 嵌入 Memos 分享页面时：

1. **页面加载**：iframe 从 `https://memos.example.com` 加载分享页面
2. **API 调用**：页面内的 JavaScript 发起**同源请求**到 Connect RPC 接口
3. **Token 校验**：服务端通过 `GetMemoByShare` 校验 token 的有效性和过期状态
4. **附件访问**：附件 URL 携带 `share_token` 参数，服务端进行额外权限校验
5. **CORS 不参与**：整个过程是同源请求，CORS 配置不生效

### 12.3 权限控制的核心边界

服务端的权限控制与是否跨域**无关**，核心是：

1. **Token 校验**：检查 token 是否存在、未过期
2. **Memo 状态**：检查 memo 是否存在、未归档
3. **附件权限**：检查是否有有效的 share_token
4. **可见性规则**：根据 memo 的 visibility 字段决定访问权限

### 12.4 设计优点

- **简单可靠**：基于 token 的分享机制，无需复杂的跨域处理
- **细粒度控制**：每个分享链接只授予特定 memo 的只读访问
- **安全性**：高熵 token、过期机制、信息泄露防护
- **灵活性**：gRPC-Gateway 允许第三方集成，Connect RPC 保护浏览器客户端
