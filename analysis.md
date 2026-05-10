# Memos 备忘录内容解析渲染与安全边界分析

## 1. 架构概览

Memos 采用 **后端存储原始内容 + 前端实时渲染** 的架构模式，实现了清晰的职责分离和灵活的渲染策略。

```
┌─────────────────────────────────────────────────────────────────┐
│                          数据流概览                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【用户输入】                                                     │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │ MemoEditor  │──(Markdown 原文)─────────────────────────┐      │
│  └─────────────┘                                           │      │
│       │                                                     │      │
│       ▼                                                     │      │
│  ┌─────────────────┐                                        │      │
│  │ 后端 API        │                                        │      │
│  │ - 验证内容长度  │                                        │      │
│  │ - 重建 MemoPayload │                                     │      │
│  │ - 关联附件       │                                        │      │
│  └─────────────────┘                                        │      │
│       │                                                     │      │
│       ▼                                                     │      │
│  ┌────────────────────────────────────────────────────┐     │      │
│  │ 数据库存储                                          │     │      │
│  │ ┌────────────────────────────────────────────────┐ │     │      │
│  │ │ memo:                                          │ │     │      │
│  │ │   - content: Markdown 原文                     │ │     │      │
│  │ │   - visibility: PUBLIC/PROTECTED/PRIVATE       │ │     │      │
│  │ │   - payload: 派生元数据 (tags, properties)     │ │     │      │
│  │ └────────────────────────────────────────────────┘ │     │      │
│  │                                                     │     │      │
│  │ ┌────────────────────────────────────────────────┐ │     │      │
│  │ │ attachment:                                    │ │     │      │
│  │ │   - uid, filename, type, size                  │ │     │      │
│  │ │   - memo_id (可选关联)                          │ │     │      │
│  │ │   - storage_type: LOCAL/S3/DATABASE            │ │     │      │
│  │ └────────────────────────────────────────────────┘ │     │      │
│  └────────────────────────────────────────────────────┘     │      │
│                                                              │      │
│  【渲染阶段】◄──────────────────────────────────────────────┘      │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ 前端 MemoMarkdownRenderer                                 │   │
│  │ ┌─────────────────────────────────────────────────────┐  │   │
│  │ │ ReactMarkdown + 插件组合                            │  │   │
│  │ │ - remark: GFM, Math, Mention, Tag                  │  │   │
│  │ │ - rehype: Raw + Sanitize + KaTeX                   │  │   │
│  │ └─────────────────────────────────────────────────────┘  │   │
│  │                                                          │   │
│  │ ┌─────────────────────────────────────────────────────┐  │   │
│  │ │ 自定义组件渲染                                        │  │   │
│  │ │ - Image, CodeBlock, Mention, Tag                   │  │   │
│  │ │ - TrustedIframe (白名单验证)                         │  │   │
│  │ └─────────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 后端 Markdown 解析与元数据提取

### 2.1 核心服务: `internal/markdown/markdown.go`

Memos 使用 **goldmark** (高性能 Markdown 解析器) 在后端进行元数据提取，但**不进行 HTML 渲染**。

#### 核心职责：

```go
// Service 接口定义了后端 Markdown 处理的完整能力
type Service interface {
    // 单次解析提取所有元数据（最高效）
    ExtractAll(content []byte) (*ExtractedData, error)
    
    // 提取 #tags
    ExtractTags(content []byte) ([]string, error)
    
    // 计算布尔属性（是否有链接、任务列表、代码等）
    ExtractProperties(content []byte) (*storepb.MemoPayload_Property, error)
    
    // 生成纯文本摘要（用于搜索、列表预览）
    GenerateSnippet(content []byte, maxLength int) (string, error)
    
    // 后端专用：RSS 等场景的 HTML 渲染
    RenderHTML(content []byte) (string, error)
}
```

#### `ExtractedData` 结构：

```go
type ExtractedData struct {
    Tags     []string                      // #tag 标签列表
    Mentions []string                      // @username 提及列表
    Property *storepb.MemoPayload_Property // 派生属性
}
```

#### `Property` 结构（`proto/store/memo.proto:15-22`）：

```protobuf
message Property {
    bool has_link = 1;              // 是否包含链接
    bool has_task_list = 2;         // 是否包含任务列表
    bool has_code = 3;              // 是否包含代码块
    bool has_incomplete_tasks = 4;  // 是否有未完成任务
    string title = 5;               // 从首个 H1 提取的标题
}
```

### 2.2 扩展机制

后端支持自定义 Markdown 扩展：

| 扩展 | 文件位置 | 功能 |
|------|---------|------|
| TagExtension | `internal/markdown/extensions/tag.go` | 解析 `#tag` 语法 |
| MentionExtension | `internal/markdown/extensions/mention.go` | 解析 `@username` 语法 |

### 2.3 MemoPayload 重建机制 (`server/runner/memopayload/runner.go`)

```go
// 创建/更新备忘录时调用
func RebuildMemoPayload(_ context.Context, memo *store.Memo, markdownService markdown.Service) error {
    if memo.Payload == nil {
        memo.Payload = &storepb.MemoPayload{}
    }

    // 单次解析提取所有元数据
    data, err := markdownService.ExtractAll([]byte(memo.Content))
    if err != nil {
        return errors.Wrap(err, "failed to extract markdown metadata")
    }

    // 更新派生数据
    memo.Payload.Tags = data.Tags
    memo.Payload.Property = data.Property
    return nil
}
```

**调用时机**：
- 创建备忘录时 (`memo_service.go:111`)
- 更新备忘录时（仅当内容变化时）
- 后台批处理 runner（用于数据迁移/重建）

---

## 3. 附件与图片资源管理

### 3.1 附件存储模型 (`store/attachment.go:16-41`)

```go
type Attachment struct {
    // 标识字段
    ID        int32
    UID       string                    // 资源名: attachments/{uid}
    
    // 元数据
    CreatorID int32
    Filename  string
    Type      string                    // MIME 类型
    Size      int64
    
    // 存储策略
    StorageType storepb.AttachmentStorageType  // LOCAL/S3/EXTERNAL
    Reference   string                         // 存储路径/URL
    Payload     *storepb.AttachmentPayload     // S3 配置、Motion Photo 等
    
    // 关联关系
    MemoID    *int32                   // 可选：关联的备忘录 ID
}
```

### 3.2 文件服务路由 (`server/router/fileserver/fileserver.go:121-125`)

```go
func (s *FileServerService) RegisterRoutes(echoServer *echo.Echo) {
    fileGroup := echoServer.Group("/file")
    // 附件访问路由
    fileGroup.GET("/attachments/:uid/:filename", s.serveAttachmentFile)
    // 头像访问路由
    fileGroup.GET("/users/:identifier/avatar", s.serveUserAvatar)
}
```

### 3.3 前端附件 URL 生成 (`web/src/utils/attachment.ts`)

```typescript
// 基础 URL
export const getAttachmentUrl = (attachment: Attachment) => {
  if (attachment.externalLink) {
    return attachment.externalLink;
  }
  return `${window.location.origin}/file/${attachment.name}/${attachment.filename}`;
};

// 缩略图
export const getAttachmentThumbnailUrl = (attachment: Attachment) => {
  return `${window.location.origin}/file/${attachment.name}/${attachment.filename}?thumbnail=true`;
};

// 动态照片视频
export const getAttachmentMotionClipUrl = (attachment: Attachment) => {
  return `${window.location.origin}/file/${attachment.name}/${attachment.filename}?motion=true`;
};
```

### 3.4 图片引用方式

**方式一：Markdown 内联图片**
```markdown
![alt text](https://example.com/image.png)
```
- 由前端 `MemoMarkdownRenderer` 中的 `Image` 组件渲染
- 支持自定义宽高属性
- 受 rehype-sanitize 保护

**方式二：关联附件**
```
附件通过 memo_attachment_service 关联到 memo
前端通过 memo.attachments 列表单独渲染
位于 MemoMetadata/Attachment 组件中
```

---

## 4. 前端渲染流程与安全边界

### 4.1 渲染器架构 (`web/src/components/MemoContent/MemoMarkdownRenderer.tsx`)

```typescript
export const MemoMarkdownRenderer = ({ content, resolvedMentionUsernames }) => {
  // 插件管道
  const remarkPlugins = [
    remarkDisableSetext,      // 禁用 Setext 标题
    remarkMath,               // 数学公式支持
    remarkGfm,                // GitHub Flavored Markdown
    remarkSplitMixedTaskLists, // 任务列表处理
    remarkBreaks,             // 换行处理
    remarkMention,            // @mention 解析
    remarkTag,                // #tag 解析
    remarkPreserveType,       // 类型保留
  ];

  const rehypePlugins = [
    rehypeRaw,                // 允许原始 HTML（配合 sanitize）
    [rehypeSanitize, SANITIZE_SCHEMA],  // ⚠️ 关键：HTML 清理
    rehypeHeadingId,          // 标题 ID
    [rehypeKatex, { throwOnError: false, strict: false }], // 数学渲染
  ];

  // 自定义组件映射
  const markdownComponents: Components = {
    img: (props) => <Image {...props} />,
    iframe: TrustedIframe,    // ⚠️ 关键：白名单 iframe
    pre: CodeBlock,
    // ... 其他组件
  };

  return (
    <ReactMarkdown
      remarkPlugins={remarkPlugins}
      rehypePlugins={rehypePlugins}
      components={markdownComponents}
    >
      {content}
    </ReactMarkdown>
  );
};
```

### 4.2 安全边界：HTML Sanitize Schema (`web/src/components/MemoContent/constants.ts:48-73`)

```typescript
export const SANITIZE_SCHEMA = {
  ...defaultSchema,
  attributes: {
    ...defaultSchema.attributes,
    
    // 图片：允许宽高
    img: [...(defaultSchema.attributes?.img || []), "height", "width"],
    
    // 复选框：允许 checked
    input: INPUT_ATTRIBUTES,
    
    // 代码块：允许 KaTeX 标记类
    code: [...(defaultSchema.attributes?.code || []), 
           ["className", ...KATEX_INLINE_CLASS_NAMES, ...KATEX_BLOCK_CLASS_NAMES]],
    
    // 内联元素：允许 mention/tag 类和数据属性
    span: [...(defaultSchema.attributes?.span || []), 
           ["className", ...SPAN_CLASS_NAMES], ["aria*"], ["data*"]],
    
    // ⚠️ iframe：仅允许白名单来源
    iframe: [
      ["src", ...TRUSTED_IFRAME_SRC_PATTERNS],  // 正则白名单
      "width", "height", "frameborder", 
      "allowfullscreen", "allow", "title", 
      "referrerpolicy", "loading",
    ],
  },
  tagNames: [...(defaultSchema.tagNames || []), "iframe"],
  protocols: {
    ...defaultSchema.protocols,
    src: ["https"],  // 仅允许 HTTPS
  },
};
```

### 4.3 受信任 iframe 白名单 (`constants.ts:20-30`)

```typescript
const TRUSTED_IFRAME_SRC_PATTERNS = [
  /^https:\/\/www\.youtube\.com\/embed\/[^?#]+(?:\?.*)?$/i,
  /^https:\/\/www\.youtube-nocookie\.com\/embed\/[^?#]+(?:\?.*)?$/i,
  /^https:\/\/player\.vimeo\.com\/video\/[^?#]+(?:\?.*)?$/i,
  /^https:\/\/open\.spotify\.com\/embed\/[^?#]+(?:\?.*)?$/i,
  /^https:\/\/w\.soundcloud\.com\/player\/?(?:\?.*)?$/i,
  /^https:\/\/www\.loom\.com\/embed\/[^?#]+(?:\?.*)?$/i,
  /^https:\/\/www\.google\.com\/maps\/embed(?:\/[^?#]*)?(?:\?.*)?$/i,
  /^https:\/\/(?:app\.)?diagrams\.net\/(?:[^?#]+)?(?:\?.*)?$/i,
  /^https:\/\/(?:www\.)?draw\.io\/(?:[^?#]+)?(?:\?.*)?$/i,
];
```

### 4.4 TrustedIframe 组件 (`web/src/components/MemoContent/TrustedIframe.ts`)

双重验证：schema 层 + 组件层

```typescript
const TrustedIframe = (props: IframeProps) => {
  if (typeof props.src !== "string" || !isTrustedIframeSrc(props.src)) {
    return null;  // 不在白名单则不渲染
  }
  return <iframe {...props} />;
};
```

### 4.5 后端文件服务安全 (`fileserver.go:52-62`)

**XSS 防护的 MIME 类型处理**：

```go
var xssUnsafeTypes = map[string]bool{
    "text/html":                true,
    "text/javascript":          true,
    "application/javascript":   true,
    "application/x-javascript": true,
    "text/xml":                 true,
    "application/xml":          true,
    "application/xhtml+xml":    true,
}

func (s *FileServerService) sanitizeContentType(mimeType string) string {
    // 危险类型转换为 application/octet-stream（强制下载）
    if xssUnsafeTypes[strings.ToLower(mimeType)] {
        return "application/octet-stream"
    }
    return contentType
}
```

**安全响应头**：

```go
func setSecurityHeaders(c *echo.Context) {
    h := c.Response().Header()
    h.Set("X-Content-Type-Options", "nosniff")
    h.Set("X-Frame-Options", "DENY")
    h.Set("Content-Security-Policy", "default-src 'none'; style-src 'unsafe-inline';")
}
```

---

## 5. 存储原文 vs 渲染结果：职责分界

### 5.1 后端存储的内容

| 存储项 | 位置 | 类型 | 说明 |
|--------|------|------|------|
| `memo.content` | `memo.go:48` | `string` | **原始 Markdown 文本**（单一事实来源） |
| `memo.payload` | `memo.go:51` | `MemoPayload` | **派生元数据**（tags、properties） |
| `attachment.*` | `attachment.go` | 独立表 | **附件二进制**（与 memo 松散关联） |

### 5.2 明确的职责边界

```
┌─────────────────────────────────────────────────────────────────┐
│                         职责分界图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【后端】                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ✅ 存储：原始 Markdown 文本                                │   │
│  │ ✅ 提取：标签、属性、提及（用于过滤/搜索）                   │   │
│  │ ✅ 验证：内容长度、语法合法性                               │   │
│  │ ✅ 权限：基于 memo.visibility 的访问控制                   │   │
│  │ ✅ 服务端渲染：RSS feed、邮件通知等特殊场景                 │   │
│  │                                                         │   │
│  │ ❌ 不存储：HTML 渲染结果                                  │   │
│  │ ❌ 不处理：前端 UI 样式                                   │   │
│  │ ❌ 不执行：用户自定义脚本                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  【前端】                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ✅ 渲染：Markdown → React 组件                           │   │
│  │ ✅ 清理：rehype-sanitize 防止 XSS                        │   │
│  │ ✅ 扩展：自定义组件（Mention, Tag, CodeBlock）            │   │
│  │ ✅ 交互：任务列表勾选、链接跳转等                          │   │
│  │                                                         │   │
│  │ ❌ 不修改：原始 Markdown 内容（除非用户编辑）              │   │
│  │ ❌ 不绕过：后端权限检查（通过 API 间接验证）               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 设计优势

1. **单一事实来源**：`memo.content` 始终是原始 Markdown，可随时重新解析
2. **前端灵活性**：可独立更新渲染插件、主题、组件，无需后端配合
3. **安全隔离**：HTML 清理在前端执行，后端不处理危险 HTML
4. **向后兼容**：旧数据可通过 `memopayload.Runner` 重新提取元数据

---

## 6. 私有资源的公开引用处理

### 6.1 附件权限检查机制 (`fileserver.go:637-688`)

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, c *echo.Context, attachment *store.Attachment) error {
    // ── 情况 1：未关联备忘录的附件 ──
    // 仅创建者和管理员可访问
    if attachment.MemoID == nil {
        user, err := s.getCurrentUser(ctx, c)
        if err != nil { /* 错误处理 */ }
        if user == nil {
            return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
        }
        if user.ID != attachment.CreatorID && user.Role != store.RoleAdmin {
            return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
        }
        return nil
    }

    // ── 情况 2：已关联备忘录 ──
    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
    if err != nil { /* 错误处理 */ }

    // ✅ 公开备忘录：任何人可访问
    if memo.Visibility == store.Public {
        return nil
    }

    // ── 情况 3：共享令牌回退（关键！） ──
    // 允许携带有效 share_token 的请求访问私有/保护备忘录的附件
    if shareToken := (*c).QueryParam("share_token"); shareToken != "" {
        ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
        if err == nil && ms != nil && 
           !isMemoShareExpired(ms) && 
           ms.MemoID == memo.ID {
            return nil  // ✅ 令牌有效，授权访问
        }
    }

    // ── 情况 4：需要认证的访问 ──
    user, err := s.getCurrentUser(ctx, c)
    if err != nil { /* 错误处理 */ }
    if user == nil {
        return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
    }

    // 私有备忘录：仅创建者和管理员
    if memo.Visibility == store.Private && 
       user.ID != memo.CreatorID && 
       user.Role != store.RoleAdmin {
        return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
    }

    return nil
}
```

### 6.2 MemoShare 机制 (`memo_share_service.go`)

**创建分享链接**：
```go
func (s *APIV1Service) CreateMemoShare(ctx context.Context, request *v1pb.CreateMemoShareRequest) (*v1pb.MemoShare, error) {
    // 权限检查：仅创建者或管理员可创建分享
    
    // 使用 shortuuid 生成高熵令牌（22字符，122位熵）
    ms, err := s.Store.CreateMemoShare(ctx, &store.MemoShare{
        UID:       shortuuid.New(),  // 分享令牌
        MemoID:    memo.ID,
        CreatorID: user.ID,
        ExpiresTs: expiresTs,        // 可选：过期时间
    })
    
    return convertMemoShareFromStore(ms, memo.UID), nil
}
```

**通过令牌获取备忘录（无需认证）**：
```go
func (s *APIV1Service) GetMemoByShare(ctx context.Context, request *v1pb.GetMemoByShareRequest) (*v1pb.Memo, error) {
    // 验证令牌有效性（存在且未过期）
    ms, err := s.getActiveMemoShare(ctx, request.ShareId)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "not found")  // 信息隐藏
    }

    memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: &ms.MemoID})
    // ... 返回备忘录及其附件
}
```

### 6.3 前端分享时的附件 URL 重写 (`useMemoShareQueries.ts:96-100`)

```typescript
/**
 * 为共享页面重写附件 URL，附加 share_token
 * 这样未登录用户也能访问私有/保护备忘录的附件
 */
export function withShareAttachmentLinks(attachments: Attachment[], token: string): Attachment[] {
  return attachments.map((a) => {
    if (a.externalLink) return a;
    // 注入 share_token 查询参数
    return { 
      ...a, 
      externalLink: `${window.location.origin}/file/${a.name}/${a.filename}?share_token=${encodeURIComponent(token)}` 
    };
  });
}
```

### 6.4 权限矩阵

| 场景 | 附件状态 | 备忘录可见性 | 访问方式 | 结果 |
|------|---------|-------------|---------|------|
| 1 | 未关联任何 memo | - | 未登录 | ❌ 401 Unauthorized |
| 2 | 未关联任何 memo | - | 非创建者登录 | ❌ 403 Forbidden |
| 3 | 关联 memo | PUBLIC | 任何人 | ✅ 允许 |
| 4 | 关联 memo | PROTECTED | 未登录 | ❌ 401 |
| 5 | 关联 memo | PROTECTED | 任意登录用户 | ✅ 允许 |
| 6 | 关联 memo | PRIVATE | 非创建者登录 | ❌ 403 |
| 7 | 关联 memo | PRIVATE | 创建者/管理员 | ✅ 允许 |
| 8 | 关联 memo | PRIVATE/PROTECTED | 携带有效 share_token | ✅ 允许 |

---

## 7. 关键安全实践总结

### 7.1 多层防御策略

```
┌─────────────────────────────────────────────────────────────────┐
│                      XSS 多层防御体系                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: 输入验证（后端）                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ memo_service.go:104-110                                   │  │
│  │ - 内容长度限制                                             │  │
│  │ - Markdown 语法验证                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  Layer 2: 前端渲染清理                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ MemoMarkdownRenderer.tsx                                  │  │
│  │ - rehype-raw + rehype-sanitize 组合                       │  │
│  │ - 自定义 SANITIZE_SCHEMA                                  │  │
│  │ - 危险属性（style, on*）被 strip                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  Layer 3: 文件服务防护                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ fileserver.go                                             │  │
│  │ - MIME 类型清理（HTML→octet-stream）                      │  │
│  │ - 安全响应头（CSP, X-Frame-Options, nosniff）             │  │
│  │ - 权限检查（visibility + share_token）                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  Layer 4: iframe 白名单                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ constants.ts + TrustedIframe.tsx                          │  │
│  │ - SANITIZE_SCHEMA 中的 src 正则验证                       │  │
│  │ - TrustedIframe 组件二次检查                              │  │
│  │ - 仅允许 HTTPS 协议                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 安全测试覆盖 (`web/tests/memo-content-security.test.tsx`)

```typescript
describe("memo content sanitization", () => {
  // 测试 1: 内联样式被 strip
  it("strips user-controlled inline styles from raw HTML spans", () => {
    const html = renderMemoContent('<span style="position:fixed;inset:0;z-index:99999">overlay</span>');
    expect(html).toMatch(/<span>overlay<\/span>/);
    expect(html).not.toMatch(/style=/);
  });

  // 测试 2: KaTeX 功能不受影响
  it("still renders KaTeX output after sanitizing math marker classes", () => {
    const html = renderMemoContent("$L$");
    expect(html).toMatch(/class="katex"/);
  });

  // 测试 3: iframe 白名单
  it("drops untrusted iframe embeds during rendering", () => {
    const trusted = renderMemoContent('<iframe src="https://www.youtube.com/embed/abc123"></iframe>');
    const untrusted = renderMemoContent('<iframe src="https://evil.example/embed/abc123"></iframe>');
    
    expect(trusted).toMatch(/<iframe/);
    expect(untrusted).not.toMatch(/<iframe/);
  });
});
```

---

## 8. 关键文件索引

| 功能模块 | 关键文件 |
|---------|---------|
| 后端 Markdown 解析 | `internal/markdown/markdown.go` |
| 标签扩展 | `internal/markdown/extensions/tag.go` |
| 提及扩展 | `internal/markdown/extensions/mention.go` |
| MemoPayload 重建 | `server/runner/memopayload/runner.go` |
| 文件服务/权限 | `server/router/fileserver/fileserver.go` |
| 备忘录服务 | `server/router/api/v1/memo_service.go` |
| 分享服务 | `server/router/api/v1/memo_share_service.go` |
| 前端渲染器 | `web/src/components/MemoContent/MemoMarkdownRenderer.tsx` |
| 安全 Schema | `web/src/components/MemoContent/constants.ts` |
| 附件 URL 生成 | `web/src/utils/attachment.ts` |
| 分享查询 | `web/src/hooks/useMemoShareQueries.ts` |
| 安全测试 | `web/tests/memo-content-security.test.tsx` |
