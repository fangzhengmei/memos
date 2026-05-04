# 资源文件上传与访问控制分析

本文档详细分析了 Memos 项目中资源文件的上传流程、外部存储（S3）接入方式以及资源访问控制逻辑。

---

## 一、资源文件上传流程

### 1.1 整体流程图

```
前端上传请求
    ↓
前端 uploadService.ts 处理
    ↓
gRPC API: CreateAttachment
    ↓
后端 attachment_service.go 处理
    ├── 用户身份验证
    ├── 文件类型/大小验证
    ├── EXIF 元数据剥离（图片）
    ├── SaveAttachmentBlob (存储处理)
    │       ├── 本地存储: 写入文件系统
    │       └── S3 存储: 上传到 S3 并生成预签名 URL
    └── 数据库记录创建
    ↓
返回 Attachment 信息
```

### 1.2 前端上传流程

**文件位置**: `web/src/components/MemoEditor/services/uploadService.ts`

前端上传服务核心实现：

```typescript
async uploadFiles(localFiles: LocalFile[]): Promise<Attachment[]> {
  // 逐个上传文件
  for (const localFile of localFiles) {
    const { file, motionMedia } = localFile;
    const buffer = new Uint8Array(await file.arrayBuffer());
    
    // 调用 gRPC 服务创建附件
    const attachment = await attachmentServiceClient.createAttachment({
      attachment: create(AttachmentSchema, {
        filename: file.name,
        size: BigInt(file.size),
        type: file.type,
        content: buffer,
        motionMedia: motionMedia ? create(MotionMediaSchema, motionMedia) : undefined,
      }),
    });
    attachments.push(attachment);
  }
  return attachments;
}
```

**关键点**:
- 使用 Connect RPC 协议与后端通信
- 文件内容以 `Uint8Array` 二进制格式传输
- 支持 Motion Photo 等特殊媒体类型

### 1.3 后端处理流程

**文件位置**: `server/router/api/v1/attachment_service.go`

#### 1.3.1 CreateAttachment 核心处理流程

```go
func (s *APIV1Service) CreateAttachment(ctx context.Context, request *v1pb.CreateAttachmentRequest) (*v1pb.Attachment, error) {
  // 1. 用户身份验证
  user, err := s.fetchCurrentUser(ctx)
  if err != nil {
    return nil, status.Errorf(codes.Internal, "failed to get current user: %v", err)
  }

  // 2. 基础参数验证
  if request.Attachment == nil {
    return nil, status.Errorf(codes.InvalidArgument, "attachment is required")
  }
  if !validateFilename(request.Attachment.Filename) {
    return nil, status.Errorf(codes.InvalidArgument, "filename contains invalid characters")
  }

  // 3. MIME 类型标准化
  normalizedMimeType := normalizeMimeType(request.Attachment.Type)
  if normalizedMimeType == "" {
    normalizedMimeType = "application/octet-stream"
  }

  // 4. 文件大小限制检查
  instanceStorageSetting, _ := s.Store.GetInstanceStorageSetting(ctx)
  uploadSizeLimit := int(instanceStorageSetting.UploadSizeLimitMb) * MebiByte
  if size > uploadSizeLimit {
    return nil, status.Errorf(codes.InvalidArgument, "file size exceeds the limit")
  }

  // 5. EXIF 元数据剥离（图片文件）
  if shouldStripExif(create.Type) {
    strippedBlob, err := stripImageExif(create.Blob, create.Type)
    if err == nil {
      create.Blob = strippedBlob
      create.Size = int64(len(strippedBlob))
    }
  }

  // 6. 存储 Blob 到对应存储介质
  if err := SaveAttachmentBlob(ctx, s.Profile, s.Store, create); err != nil {
    return nil, status.Errorf(codes.Internal, "failed to save attachment blob")
  }

  // 7. 创建数据库记录
  attachment, err := s.Store.CreateAttachment(ctx, create)
  
  return convertAttachmentFromStore(attachment), nil
}
```

#### 1.3.2 文件名验证逻辑

```go
func validateFilename(filename string) bool {
  // 防止路径遍历攻击
  if !filepath.IsLocal(filename) || strings.ContainsAny(filename, "/\\") {
    return false
  }
  // 防止特殊命名（如 "."、".."、空格开头/结尾）
  if strings.HasPrefix(filename, " ") || strings.HasSuffix(filename, " ") ||
     strings.HasPrefix(filename, ".") || strings.HasSuffix(filename, ".") {
    return false
  }
  return true
}
```

#### 1.3.3 EXIF 元数据剥离

**目的**: 保护用户隐私，移除图片中的 GPS 位置、相机型号等敏感信息

**支持的图片格式**:
- JPEG/JPG
- TIFF
- WebP
- HEIC/HEIF

**实现逻辑**:
```go
func stripImageExif(imageData []byte, mimeType string) ([]byte, error) {
  // 1. 解码图片（自动应用 EXIF 方向矫正）
  img, err := imaging.Decode(bytes.NewReader(imageData), imaging.AutoOrientation(true))
  
  // 2. 重新编码（清除所有元数据）
  var buf bytes.Buffer
  if mimeType == "image/png" {
    imaging.Encode(&buf, img, imaging.PNG)  // PNG 保持无损
  } else {
    // 其他格式转 JPEG，质量 95
    imaging.Encode(&buf, img, imaging.JPEG, imaging.JPEGQuality(95))
  }
  return buf.Bytes(), nil
}
```

---

## 二、外部存储（S3）接入方式

### 2.1 存储架构概览

系统支持三种存储类型：
1. **DATABASE**: 文件内容直接存储在数据库的 BLOB 字段中
2. **LOCAL**: 文件存储在本地文件系统
3. **S3**: 文件存储在兼容 S3 协议的对象存储服务

### 2.2 S3 客户端实现

**文件位置**: `internal/storage/s3/s3.go`

#### 2.2.1 客户端初始化

```go
func NewClient(ctx context.Context, s3Config *storepb.StorageS3Config) (*Client, error) {
  cfg, err := config.LoadDefaultConfig(ctx,
    config.WithCredentialsProvider(
      credentials.NewStaticCredentialsProvider(
        s3Config.AccessKeyId, 
        s3Config.AccessKeySecret, 
        "",
      ),
    ),
    config.WithRegion(s3Config.Region),
  )
  
  client := s3.NewFromConfig(cfg, func(o *s3.Options) {
    o.BaseEndpoint = aws.String(s3Config.Endpoint)
    o.UsePathStyle = s3Config.UsePathStyle  // 支持 MinIO 等兼容服务
    o.RequestChecksumCalculation = aws.RequestChecksumCalculationWhenRequired
    o.ResponseChecksumValidation = aws.ResponseChecksumValidationWhenRequired
  })
  
  return &Client{
    Client: client,
    Bucket: aws.String(s3Config.Bucket),
  }, nil
}
```

#### 2.2.2 S3 配置参数

| 参数名 | 说明 | 示例 |
|--------|------|------|
| `AccessKeyId` | 访问密钥 ID | `AKIAIOSFODNN7EXAMPLE` |
| `AccessKeySecret` | 访问密钥 | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `Region` | 区域 | `us-east-1` |
| `Endpoint` | 自定义端点（支持 MinIO 等） | `https://s3.example.com` |
| `Bucket` | 存储桶名称 | `my-memos-assets` |
| `UsePathStyle` | 是否使用路径风格寻址 | `true` (MinIO 常用) |

#### 2.2.3 核心方法

```go
// 上传对象到 S3
func (c *Client) UploadObject(ctx context.Context, key string, fileType string, content io.Reader) (string, error)

// 生成预签名 URL（有效期 5 天）
func (c *Client) PresignGetObject(ctx context.Context, key string) (string, error)

// 获取对象内容（一次性读取）
func (c *Client) GetObject(ctx context.Context, key string) ([]byte, error)

// 获取对象内容流（用于大文件/视频流式传输）
func (c *Client) GetObjectStream(ctx context.Context, key string) (io.ReadCloser, error)

// 删除对象
func (c *Client) DeleteObject(ctx context.Context, key string) error
```

### 2.3 S3 上传流程

**文件位置**: `server/router/api/v1/attachment_service.go:SaveAttachmentBlob`

```go
func SaveAttachmentBlob(ctx context.Context, profile *profile.Profile, stores *store.Store, create *store.Attachment) error {
  instanceStorageSetting, _ := stores.GetInstanceStorageSetting(ctx)
  
  if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_S3 {
    s3Config := instanceStorageSetting.S3Config
    
    // 1. 创建 S3 客户端
    s3Client, err := s3.NewClient(ctx, s3Config)
    
    // 2. 应用路径模板（支持变量替换）
    filepathTemplate := instanceStorageSetting.FilepathTemplate
    filepathTemplate = replaceFilenameWithPathTemplate(filepathTemplate, create.Filename)
    
    // 3. 上传到 S3
    key, err := s3Client.UploadObject(ctx, filepathTemplate, create.Type, bytes.NewReader(create.Blob))
    
    // 4. 生成预签名 URL（有效期 5 天）
    presignURL, err := s3Client.PresignGetObject(ctx, key)
    
    // 5. 设置附件元数据
    create.Reference = presignURL           // 预签名 URL 作为引用
    create.Blob = nil                        // 释放内存
    create.StorageType = storepb.AttachmentStorageType_S3
    create.Payload = &storepb.AttachmentPayload{
      Payload: &storepb.AttachmentPayload_S3Object_{
        S3Object: &storepb.AttachmentPayload_S3Object{
          S3Config:          s3Config,      // 保存配置（后续刷新预签名用）
          Key:               key,           // 对象键
          LastPresignedTime: timestamppb.Now(),
        },
      },
    }
  }
}
```

### 2.4 路径模板变量

支持在 `FilepathTemplate` 中使用以下变量：

| 变量 | 说明 | 示例输出 |
|------|------|----------|
| `{filename}` | 原始文件名 | `photo.jpg` |
| `{timestamp}` | Unix 时间戳 | `1714888800` |
| `{year}` | 年份（4位） | `2026` |
| `{month}` | 月份（2位） | `05` |
| `{day}` | 日期（2位） | `04` |
| `{hour}` | 小时（2位） | `14` |
| `{minute}` | 分钟（2位） | `30` |
| `{second}` | 秒（2位） | `45` |
| `{uuid}` | 随机 UUID | `a1b2c3d4-...` |

**默认模板**: `assets/{timestamp}_{uuid}_{filename}`

### 2.5 预签名 URL 自动刷新

**文件位置**: `server/runner/s3presign/runner.go`

由于 S3 预签名 URL 有有效期（系统设置为 5 天），系统会定期刷新即将过期的 URL。

#### 2.5.1 定时任务配置

```go
const runnerInterval = time.Hour * 12  // 每 12 小时执行一次

func (r *Runner) Run(ctx context.Context) {
  ticker := time.NewTicker(runnerInterval)
  defer ticker.Stop()
  
  for {
    select {
    case <-ticker.C:
      r.RunOnce(ctx)
    case <-ctx.Done():
      return
    }
  }
}
```

#### 2.5.2 刷新策略

```go
func (r *Runner) CheckAndPresign(ctx context.Context) {
  // 批量处理，每批 100 个
  const batchSize = 100
  
  for {
    attachments, err := r.Store.ListAttachments(ctx, &store.FindAttachment{
      StorageType: &s3StorageType,  // 只处理 S3 存储的附件
      Limit:       &batchSize,
      Offset:      &offset,
    })
    
    for _, attachment := range attachments {
      s3ObjectPayload := attachment.Payload.GetS3Object()
      
      // 检查是否需要刷新：剩余有效期 < 4 天时刷新
      // 预签名 URL 有效期为 5 天，提前 1 天刷新
      if time.Now().Before(s3ObjectPayload.LastPresignedTime.AsTime().Add(4 * 24 * time.Hour)) {
        continue  // 还在有效期内，跳过
      }
      
      // 重新生成预签名 URL
      presignURL, err := s3Client.PresignGetObject(ctx, s3ObjectPayload.Key)
      
      // 更新数据库
      r.Store.UpdateAttachment(ctx, &store.UpdateAttachment{
        ID:        attachment.ID,
        Reference: &presignURL,
        Payload: &storepb.AttachmentPayload{
          Payload: &storepb.AttachmentPayload_S3Object_{
            S3Object: s3ObjectPayload,
          },
        },
      })
    }
    offset += len(attachments)
  }
}
```

### 2.6 本地存储 vs S3 存储对比

| 特性 | 本地存储 (LOCAL) | S3 存储 |
|------|------------------|---------|
| 存储位置 | 服务器文件系统 | 对象存储服务 |
| 路径处理 | 相对路径（基于 profile.Data） | 对象键 (Key) |
| 访问方式 | 直接文件读取 | 预签名 URL 或代理流式读取 |
| URL 有效期 | 永久（服务器决定） | 5 天（可自动刷新） |
| 扩展性 | 受服务器磁盘限制 | 无限扩展 |
| 成本 | 服务器存储成本 | 对象存储服务成本 |
| 适合场景 | 小型部署、单机环境 | 生产环境、高可用、多节点 |

---

## 三、资源访问控制逻辑

### 3.1 访问入口

资源访问有两个主要入口：

1. **gRPC API 层**: 通过 `AttachmentService` 访问附件元数据
   - `GetAttachment` - 获取单个附件信息
   - `ListAttachments` - 列出用户的附件

2. **HTTP 文件服务器**: 直接访问文件内容
   - 路由: `/file/attachments/:uid/:filename`
   - 实现: `server/router/fileserver/fileserver.go`

### 3.2 权限检查核心逻辑

#### 3.2.1 HTTP 文件访问权限检查

**文件位置**: `server/router/fileserver/fileserver.go:checkAttachmentPermission`

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, c *echo.Context, attachment *store.Attachment) error {
  // 场景 1: 未关联到 Memo 的附件
  // 规则: 只有创建者或管理员可以访问
  if attachment.MemoID == nil {
    user, err := s.getCurrentUser(ctx, c)
    if err != nil {
      return echo.NewHTTPError(http.StatusInternalServerError, "failed to get current user")
    }
    if user == nil {
      return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
    }
    // 权限检查: 创建者 或 管理员
    if user.ID != attachment.CreatorID && user.Role != store.RoleAdmin {
      return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
    }
    return nil
  }

  // 场景 2: 关联到 Memo 的附件
  // 规则: 遵循 Memo 的可见性规则
  
  // 2.1 获取关联的 Memo
  memo, err := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
  if memo == nil {
    return echo.NewHTTPError(http.StatusNotFound, "memo not found")
  }

  // 2.2 公开 Memo: 任何人都可以访问
  if memo.Visibility == store.Public {
    return nil
  }

  // 2.3 共享链接访问: 检查有效的 Share Token
  // 支持通过 URL 参数携带 share_token 访问私有/受限 Memo 的附件
  if shareToken := c.QueryParam("share_token"); shareToken != "" {
    ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
    // 检查: Token 存在 + 未过期 + 对应正确的 Memo
    if err == nil && ms != nil && !isMemoShareExpired(ms) && ms.MemoID == memo.ID {
      return nil  // 允许访问
    }
  }

  // 2.4 需要登录验证
  user, err := s.getCurrentUser(ctx, c)
  if user == nil {
    return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
  }

  // 2.5 私有 Memo: 只有创建者或管理员可以访问
  if memo.Visibility == store.Private && user.ID != memo.CreatorID && user.Role != store.RoleAdmin {
    return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
  }

  return nil
}
```

#### 3.2.2 API 层权限检查

**文件位置**: `server/router/api/v1/attachment_service.go:checkAttachmentAccess`

```go
func (s *APIV1Service) checkAttachmentAccess(ctx context.Context, attachment *store.Attachment) error {
  user, _ := s.fetchCurrentUser(ctx)

  // 未关联 Memo: 只有创建者或管理员可访问
  if attachment.MemoID == nil {
    if user == nil {
      return status.Errorf(codes.Unauthenticated, "user not authenticated")
    }
    if attachment.CreatorID != user.ID && !isSuperUser(user) {
      return status.Errorf(codes.PermissionDenied, "permission denied")
    }
    return nil
  }

  // 关联 Memo: 遵循 Memo 可见性
  memo, _ := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
  
  if memo.Visibility == store.Public {
    return nil  // 公开，任何人可访问
  }
  
  if user == nil {
    return status.Errorf(codes.Unauthenticated, "user not authenticated")
  }
  
  // 私有 Memo: 只有创建者或管理员
  if memo.Visibility == store.Private && memo.CreatorID != user.ID && !isSuperUser(user) {
    return status.Errorf(codes.PermissionDenied, "permission denied")
  }
  
  return nil
}
```

### 3.3 权限决策矩阵

| 附件类型 | 访问者身份 | Memo 可见性 | 是否有权限 |
|----------|------------|-------------|------------|
| 未关联 Memo | 未登录 | - | ❌ 未授权 |
| 未关联 Memo | 创建者本人 | - | ✅ 允许 |
| 未关联 Memo | 其他用户 | - | ❌ 禁止 |
| 未关联 Memo | 管理员 | - | ✅ 允许 |
| 关联 Memo | 任何人 | Public | ✅ 允许 |
| 关联 Memo | 未登录 | Protected/Private | ❌ 未授权 |
| 关联 Memo | 登录用户 | Protected | ✅ 允许 |
| 关联 Memo | 非创建者 | Private | ❌ 禁止 |
| 关联 Memo | 创建者本人 | Private | ✅ 允许 |
| 关联 Memo | 管理员 | Private | ✅ 允许 |
| 关联 Memo | 携带有效 Share Token | Protected/Private | ✅ 允许 |

### 3.4 用户身份认证

**文件位置**: `server/router/fileserver/fileserver.go:getCurrentUser`

```go
func (s *FileServerService) getCurrentUser(ctx context.Context, c *echo.Context) (*store.User, error) {
  // 认证优先级:
  // 1. Authorization Header (Bearer Token) - Access Token V2 或 PAT
  // 2. Cookie 中的 Refresh Token
  
  authHeader := c.Request().Header.Get(echo.HeaderAuthorization)
  cookieHeader := c.Request().Header.Get("Cookie")
  
  return s.authenticator.AuthenticateToUser(ctx, authHeader, cookieHeader)
}
```

### 3.5 公开端点配置

**文件位置**: `server/router/api/v1/acl_config.go`

系统中有专门的公开端点配置，但注意：**附件相关的 API 不在公开列表中**。

```go
var PublicMethods = map[string]struct{}{
  // Auth 相关
  "/memos.api.v1.AuthService/SignIn": {},
  "/memos.api.v1.AuthService/RefreshToken": {},
  
  // Instance 相关（登录前需要）
  "/memos.api.v1.InstanceService/GetInstanceProfile": {},
  "/memos.api.v1.InstanceService/GetInstanceSetting": {},
  
  // User 相关（公开资料）
  "/memos.api.v1.UserService/CreateUser": {},
  "/memos.api.v1.UserService/GetUser": {},
  "/memos.api.v1.UserService/GetUserAvatar": {},
  
  // Memo 相关（可见性在服务层过滤）
  "/memos.api.v1.MemoService/GetMemo": {},
  "/memos.api.v1.MemoService/ListMemos": {},
  "/memos.api.v1.MemoService/GetMemoByShare": {},
  
  // 注意: AttachmentService 的方法不在此列表中
  // 所有附件操作都需要通过服务层的权限检查
}
```

### 3.6 文件服务安全防护

#### 3.6.1 XSS 防护

对于可能执行脚本的 MIME 类型，强制下载而不是直接渲染：

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

func sanitizeContentType(mimeType string) string {
  if xssUnsafeTypes[strings.ToLower(mimeType)] {
    return "application/octet-stream"  // 强制下载
  }
  return mimeType
}
```

#### 3.6.2 安全响应头

所有文件响应都设置安全头：

```go
func setSecurityHeaders(c *echo.Context) {
  h := c.Response().Header()
  h.Set("X-Content-Type-Options", "nosniff")     // 防止 MIME 类型嗅探
  h.Set("X-Frame-Options", "DENY")               // 防止 Clickjacking
  h.Set("Content-Security-Policy", "default-src 'none'; style-src 'unsafe-inline';")
}
```

#### 3.6.3 缓存控制

```go
const cacheMaxAge = "public, max-age=3600"  // 1 小时缓存

func setMediaHeaders(c *echo.Context, contentType, originalType string) {
  h := c.Response().Header()
  h.Set(echo.HeaderContentType, contentType)
  h.Set(echo.HeaderCacheControl, cacheMaxAge)
  
  // 支持 HDR/广色域
  if strings.HasPrefix(originalType, "image/") || strings.HasPrefix(originalType, "video/") {
    h.Set("Color-Gamut", "srgb, p3, rec2020")
  }
}
```

---

## 四、文件服务实现细节

### 4.1 文件服务路由

**文件位置**: `server/router/fileserver/fileserver.go`

```go
func (s *FileServerService) RegisterRoutes(echoServer *echo.Echo) {
  fileGroup := echoServer.Group("/file")
  fileGroup.GET("/attachments/:uid/:filename", s.serveAttachmentFile)  // 附件文件
  fileGroup.GET("/users/:identifier/avatar", s.serveUserAvatar)         // 用户头像
}
```

### 4.2 附件文件服务流程

```go
func (s *FileServerService) serveAttachmentFile(c *echo.Context) error {
  // 1. 解析参数
  uid := c.Param("uid")
  wantThumbnail := c.QueryParam("thumbnail") == "true"
  wantMotion := c.QueryParam("motion") == "true"
  
  // 2. 查询附件信息
  attachment, err := s.Store.GetAttachment(ctx, &store.FindAttachment{
    UID:     &uid,
    GetBlob: true,
  })
  
  // 3. 权限检查
  if err := s.checkAttachmentPermission(ctx, c, attachment); err != nil {
    return err
  }
  
  // 4. 处理特殊请求
  if wantMotion {
    return s.serveMotionClip(c, attachment)  // 动态照片的视频部分
  }
  
  // 5. 根据存储类型和文件类型选择服务方式
  contentType := s.sanitizeContentType(attachment.Type)
  
  // 视频/音频: 流式传输（支持 Range 请求）
  if isMediaType(attachment.Type) {
    return s.serveMediaStream(c, attachment, contentType)
  }
  
  // 其他文件: 静态文件服务（支持缩略图）
  return s.serveStaticFile(c, attachment, contentType, wantThumbnail)
}
```

### 4.3 不同存储类型的文件服务

#### 4.3.1 流式传输（视频/音频）

```go
func (s *FileServerService) serveMediaStream(c *echo.Context, attachment *store.Attachment, contentType string) error {
  switch attachment.StorageType {
  case storepb.AttachmentStorageType_LOCAL:
    // 本地存储: 直接使用 http.ServeFile
    filePath, _ := s.resolveLocalPath(attachment.Reference)
    http.ServeFile(c.Response(), c.Request(), filePath)
    return nil
    
  case storepb.AttachmentStorageType_S3:
    // S3 存储: 重定向到预签名 URL
    // 让客户端直接从 S3 下载，减轻服务器压力
    presignURL, _ := s.getS3PresignedURL(c.Request().Context(), attachment)
    return c.Redirect(http.StatusTemporaryRedirect, presignURL)
    
  default:
    // 数据库存储: 使用 http.ServeContent（支持 Range）
    modTime := time.Unix(attachment.UpdatedTs, 0)
    http.ServeContent(c.Response(), c.Request(), attachment.Filename, modTime, 
      bytes.NewReader(attachment.Blob))
    return nil
  }
}
```

#### 4.3.2 静态文件服务

```go
func (s *FileServerService) serveStaticFile(c *echo.Context, attachment *store.Attachment, contentType string, wantThumbnail bool) error {
  // 缩略图处理
  if wantThumbnail && thumbnailSupportedTypes[attachment.Type] {
    if thumbnailBlob, err := s.getOrGenerateThumbnail(ctx, attachment); err == nil {
      return c.Blob(http.StatusOK, "image/jpeg", thumbnailBlob)
    }
  }
  
  // 非媒体文件: 强制下载防止 XSS
  if !strings.HasPrefix(contentType, "image/") && contentType != "application/pdf" {
    c.Response().Header().Set(echo.HeaderContentDisposition, 
      fmt.Sprintf("attachment; filename=%q", attachment.Filename))
  }
  
  switch attachment.StorageType {
  case storepb.AttachmentStorageType_LOCAL:
    filePath, _ := s.resolveLocalPath(attachment.Reference)
    http.ServeFile(c.Response(), c.Request(), filePath)
    
  case storepb.AttachmentStorageType_S3:
    // S3: 代理流式读取（避免重定向的 CORS 问题）
    reader, _ := s.getAttachmentReader(ctx, attachment)
    defer reader.Close()
    return c.Stream(http.StatusOK, contentType, reader)
    
  default:
    return c.Blob(http.StatusOK, contentType, attachment.Blob)
  }
}
```

### 4.4 缩略图生成

```go
func (s *FileServerService) getOrGenerateThumbnail(ctx context.Context, attachment *store.Attachment) ([]byte, error) {
  // 1. 检查缓存
  thumbnailPath := filepath.Join(s.Profile.Data, ThumbnailCacheFolder, attachment.UID+".v2.jpeg")
  if blob, err := s.readCachedThumbnail(thumbnailPath); err == nil {
    return blob, nil
  }
  
  // 2. 并发控制（防止内存耗尽）
  s.thumbnailSemaphore.Acquire(ctx, 1)
  defer s.thumbnailSemaphore.Release(1)
  
  // 3. 再次检查缓存（双重检查锁定）
  if blob, err := s.readCachedThumbnail(thumbnailPath); err == nil {
    return blob, nil
  }
  
  // 4. 生成缩略图
  return s.generateThumbnail(ctx, attachment, thumbnailPath)
}

func (s *FileServerService) generateThumbnail(ctx context.Context, attachment *store.Attachment, thumbnailPath string) ([]byte, error) {
  reader, _ := s.getAttachmentReader(ctx, attachment)
  img, _ := imaging.Decode(reader, imaging.AutoOrientation(true))
  
  // 计算尺寸（最大边 600px）
  thumbnailWidth, thumbnailHeight := calculateThumbnailDimensions(img.Bounds().Dx(), img.Bounds().Dy())
  
  // 调整大小
  thumbnailImage := imaging.Resize(img, thumbnailWidth, thumbnailHeight, imaging.Lanczos)
  
  // 保存为 JPEG（质量 90）
  output, _ := os.Create(thumbnailPath)
  imaging.Encode(output, thumbnailImage, imaging.JPEG, imaging.JPEGQuality(90))
  
  return s.readCachedThumbnail(thumbnailPath)
}
```

### 4.5 动态照片（Motion Photo）支持

系统支持 Android Motion Photo 和 Apple Live Photo：

```go
func (s *FileServerService) serveMotionClip(c *echo.Context, attachment *store.Attachment) error {
  motionMedia := attachment.Payload.GetMotionMedia()
  if motionMedia == nil || motionMedia.Family != storepb.MotionMediaFamily_ANDROID_MOTION_PHOTO {
    return echo.NewHTTPError(http.StatusBadRequest, "attachment does not have motion clip")
  }
  
  // 提取或获取缓存的视频片段
  clipBlob, err := s.getOrExtractMotionClip(c.Request().Context(), attachment)
  
  // 作为 MP4 提供
  modTime := time.Unix(attachment.UpdatedTs, 0)
  http.ServeContent(c.Response(), c.Request(), attachment.UID+".mp4", modTime, bytes.NewReader(clipBlob))
  return nil
}
```

---

## 五、附件数据模型

### 5.1 Attachment 结构体

**文件位置**: `store/attachment.go`

```go
type Attachment struct {
  // 系统字段
  ID        int32   // 内部主键
  UID       string  // 对外唯一标识（URL 友好）
  
  // 标准字段
  CreatorID int32   // 创建者用户 ID
  CreatedTs int64   // 创建时间戳
  UpdatedTs int64   // 更新时间戳
  
  // 业务字段
  Filename    string                        // 文件名
  Blob        []byte                        // 文件内容（仅 DATABASE 存储类型）
  Type        string                        // MIME 类型
  Size        int64                         // 文件大小（字节）
  StorageType storepb.AttachmentStorageType // 存储类型: DATABASE/LOCAL/S3
  Reference   string                        // 引用路径/URL
  Payload     *storepb.AttachmentPayload    // 额外载荷（S3 配置、Motion Media 等）
  
  // 关联字段
  MemoID  *int32   // 关联的 Memo ID（可选）
  MemoUID *string  // 关联的 Memo UID（组合字段）
}
```

### 5.2 AttachmentPayload 结构

```protobuf
message AttachmentPayload {
  oneof payload {
    S3Object s3_object = 1;
    MotionMedia motion_media = 2;
  }
}

message S3Object {
  StorageS3Config s3_config = 1;      // S3 配置（用于刷新预签名 URL）
  string key = 2;                      // 对象键
  google.protobuf.Timestamp last_presigned_time = 3;  // 上次预签名时间
}

message MotionMedia {
  MotionMediaFamily family = 1;        // ANDROID_MOTION_PHOTO / APPLE_LIVE_PHOTO
  MotionMediaRole role = 2;            // CONTAINER / STILL / VIDEO
  string group_id = 3;                  // 分组 ID（用于关联静态图和视频）
  int64 presentation_timestamp_us = 4;  // 展示时间戳
  bool has_embedded_video = 5;          // 是否包含内嵌视频
}
```

### 5.3 存储类型枚举

```protobuf
enum AttachmentStorageType {
  ATTACHMENT_STORAGE_TYPE_UNSPECIFIED = 0;
  DATABASE = 1;  // 存储在数据库 BLOB
  LOCAL = 2;     // 存储在本地文件系统
  S3 = 3;        // 存储在 S3 兼容对象存储
  EXTERNAL = 4;  // 外部链接（用户直接提供 URL）
}
```

---

## 六、附件删除流程

### 6.1 删除逻辑

**文件位置**: `store/attachment.go`

```go
func (s *Store) DeleteAttachment(ctx context.Context, delete *DeleteAttachment) error {
  // 1. 获取附件信息
  attachment, err := s.GetAttachment(ctx, &FindAttachment{ID: &delete.ID})
  
  // 2. 删除存储中的文件（本地文件或 S3 对象）
  if err := s.DeleteAttachmentStorage(ctx, attachment); err != nil {
    if attachment.StorageType == storepb.AttachmentStorageType_LOCAL {
      return errors.Wrap(err, "failed to delete local file")
    }
    // S3 删除失败仅记录警告，不阻断数据库删除
    slog.Warn("Failed to delete attachment storage", slog.Any("err", err))
  }
  
  // 3. 删除数据库记录
  return s.driver.DeleteAttachment(ctx, delete)
}
```

### 6.2 S3 对象删除

```go
func (s *Store) deleteAttachmentStorageImpl(ctx context.Context, attachment *Attachment, instanceStorageSetting *storepb.InstanceStorageSetting) error {
  if attachment.StorageType == storepb.AttachmentStorageType_S3 {
    s3ObjectPayload := attachment.Payload.GetS3Object()
    
    // 1. 获取 S3 配置
    // - 优先使用 attachment 中保存的配置
    // - 否则使用当前实例配置（兼容旧数据）
    s3Config := s3ObjectPayload.S3Config
    if s3Config == nil {
      if instanceStorageSetting == nil {
        instanceStorageSetting, _ = s.GetInstanceStorageSetting(ctx)
      }
      s3Config = instanceStorageSetting.S3Config
    }
    
    // 2. 创建客户端并删除
    s3Client, _ := s3.NewClient(ctx, s3Config)
    s3Client.DeleteObject(ctx, s3ObjectPayload.Key)
  }
  
  // 3. 删除衍生缓存（缩略图、动态照片视频）
  s.deleteAttachmentDerivedCaches(attachment)
  return nil
}
```

---

## 七、关键配置项

### 7.1 实例存储配置

**文件位置**: `store/instance_setting.go`

```go
type InstanceStorageSetting struct {
  StorageType          InstanceStorageSetting_StorageType  // LOCAL / S3
  UploadSizeLimitMb    int32                               // 上传大小限制（MB）
  FilepathTemplate     string                              // 路径模板
  S3Config             *StorageS3Config                    // S3 配置（仅 S3 模式）
}

// 默认值
const (
  defaultInstanceStorageType       = storepb.InstanceStorageSetting_LOCAL
  defaultInstanceUploadSizeLimitMb = 30
  defaultInstanceFilepathTemplate  = "assets/{timestamp}_{uuid}_{filename}"
)
```

### 7.2 上传缓冲区

```go
const (
  MaxUploadBufferSizeBytes = 32 << 20  // 32 MiB 内存缓冲区
  MebiByte                 = 1024 * 1024
)
```

---

## 八、总结

### 8.1 架构亮点

1. **多存储抽象**: 通过统一的 `StorageType` 和 `Reference` 字段，透明支持数据库、本地文件、S3 三种存储方式
2. **预签名 URL 自动刷新**: S3 模式下，后台任务定期刷新即将过期的预签名 URL，对用户无感知
3. **灵活的权限模型**: 附件权限与关联的 Memo 可见性绑定，同时支持 Share Token 临时访问
4. **隐私保护**: 图片自动剥离 EXIF 元数据，防止 GPS 位置等敏感信息泄露
5. **安全防护**: XSS 类型强制下载、安全响应头、路径遍历防护

### 8.2 数据流概览

```
上传流程:
前端 → gRPC CreateAttachment → 验证 → EXIF剥离 → SaveAttachmentBlob (本地/S3) → 数据库记录

访问流程:
/file/attachments/:uid → 权限检查 → 按存储类型服务
  - LOCAL: 直接读取文件
  - S3: 重定向预签名URL 或 代理流式读取
  - DATABASE: 从 BLOB 字段读取

权限决策:
未关联 Memo → 仅创建者/管理员
已关联 Memo → Public: 任何人
              Protected: 登录用户
              Private: 创建者/管理员
              + Share Token: 临时授权访问
```

### 8.3 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 前端上传服务 | `web/src/components/MemoEditor/services/uploadService.ts` |
| 附件 API 服务 | `server/router/api/v1/attachment_service.go` |
| HTTP 文件服务器 | `server/router/fileserver/fileserver.go` |
| S3 客户端实现 | `internal/storage/s3/s3.go` |
| S3 预签名刷新 | `server/runner/s3presign/runner.go` |
| 附件存储层 | `store/attachment.go` |
| 实例配置管理 | `store/instance_setting.go` |
| 公开端点 ACL | `server/router/api/v1/acl_config.go` |
