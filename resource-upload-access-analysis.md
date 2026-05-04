# 资源文件上传与访问控制分析

本文档详细分析了 Memos 项目中资源文件的上传流程、外部存储（S3）接入方式、资源访问控制逻辑、附件与备忘录绑定的权限变化、不同入口的鉴权边界差异，以及删除权限的差异。

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

系统支持四种存储类型：
1. **DATABASE**: 文件内容直接存储在数据库的 BLOB 字段中
2. **LOCAL**: 文件存储在本地文件系统
3. **S3**: 文件存储在兼容 S3 协议的对象存储服务
4. **EXTERNAL**: 外部链接（用户直接提供 URL）

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

## 三、附件与备忘录绑定的权限变化链路

### 3.1 绑定前的权限状态

**文件位置**: `server/router/api/v1/attachment_service.go:checkAttachmentAccess` 和 `server/router/fileserver/fileserver.go:checkAttachmentPermission`

附件在**未绑定到备忘录**时（`MemoID == nil`），权限规则非常严格：

```go
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
```

**绑定前权限总结**：

| 访问者 | 权限 |
|--------|------|
| 未登录用户 | ❌ Unauthenticated |
| 附件创建者 | ✅ 允许访问 |
| 其他登录用户 | ❌ PermissionDenied |
| 管理员 | ✅ 允许访问 |

### 3.2 绑定过程的权限控制

**文件位置**: `server/router/api/v1/memo_attachment_service.go`

绑定操作通过 `SetMemoAttachments` API 完成，该过程有多层权限检查：

#### 3.2.1 第一层：Memo 修改权限

```go
func (s *APIV1Service) SetMemoAttachments(ctx context.Context, request *v1pb.SetMemoAttachmentsRequest) (*emptypb.Empty, error) {
  user, err := s.fetchCurrentUser(ctx)
  // ...
  
  memo, err := s.Store.GetMemo(ctx, &store.FindMemo{UID: &memoUID})
  // ...
  
  // 关键检查：用户必须能修改 Memo
  if !canModifyMemo(user, memo) {
    return nil, status.Errorf(codes.PermissionDenied, "permission denied")
  }
  // ...
}
```

**`canModifyMemo` 规则**（通常定义在备忘录服务中）：
- Memo 创建者可以修改
- 管理员可以修改任意 Memo

#### 3.2.2 第二层：附件归属检查

在 `normalizeMemoAttachmentRequest` 中，检查要绑定的附件是否属于当前用户：

```go
func (s *APIV1Service) normalizeMemoAttachmentRequest(
  ctx context.Context,
  user *store.User,
  currentAttachments []*store.Attachment,
  requestAttachments []*v1pb.Attachment,
) ([]*store.Attachment, error) {
  for _, requestAttachment := range requestAttachments {
    attachmentUID, _ := ExtractAttachmentUIDFromName(requestAttachment.Name)
    attachment, _ := s.Store.GetAttachment(ctx, &store.FindAttachment{UID: &attachmentUID})
    
    // 关键检查：不能绑定其他用户的附件
    if attachment.CreatorID != user.ID && !isSuperUser(user) {
      return nil, status.Errorf(codes.PermissionDenied, "cannot attach another user's attachment")
    }
  }
  // ...
}
```

#### 3.2.3 第三层：解绑时的权限检查

在 `setMemoAttachmentsInternal` 中，对于不再绑定的附件（需要解绑），同样有权限检查：

```go
func (s *APIV1Service) setMemoAttachmentsInternal(ctx context.Context, user *store.User, memo *store.Memo, requestAttachments []*v1pb.Attachment) error {
  // ...
  
  // 删除不在请求中的附件（解绑）
  for _, attachment := range currentAttachments {
    if !requestedIDs[attachment.ID] {
      // 关键检查：不能移除其他用户的附件
      if attachment.CreatorID != user.ID && !isSuperUser(user) {
        return status.Errorf(codes.PermissionDenied, "cannot remove another user's attachment")
      }
      // 执行删除/解绑
      s.Store.DeleteAttachment(ctx, &store.DeleteAttachment{
        ID:     int32(attachment.ID),
        MemoID: &memo.ID,
      })
    }
  }
  
  // 绑定新附件：更新 MemoID
  for index, attachment := range normalizedAttachments {
    updatedTs := time.Now().Unix() + int64(index)
    s.Store.UpdateAttachment(ctx, &store.UpdateAttachment{
      ID:        attachment.ID,
      MemoID:    &memo.ID,  // 绑定到 Memo
      UpdatedTs: &updatedTs,
    })
  }
  // ...
}
```

### 3.3 绑定后的权限状态

附件绑定到备忘录后（`MemoID != nil`），权限**完全遵循备忘录的可见性规则**：

```go
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
```

### 3.4 权限变化链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        附件权限状态转换链路                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐                                                           │
│  │  新建附件    │  MemoID = nil                                             │
│  │  (未绑定)    │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         │ 权限规则:                                                         │
│         │  - 必须登录                                                       │
│         │  - 仅创建者/管理员可访问                                          │
│         │  - 其他用户即使登录也无法访问                                      │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    SetMemoAttachments (绑定过程)                       │  │
│  │  权限检查:                                                             │  │
│  │  1. 当前用户必须能修改目标 Memo (canModifyMemo)                        │  │
│  │  2. 要绑定的附件必须属于当前用户 (或管理员)                              │  │
│  │  3. 要解绑的附件必须属于当前用户 (或管理员)                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │  已绑定附件  │  MemoID = memo.ID                                        │
│  │              │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         │ 权限规则: 遵循 Memo 可见性                                        │
│         │                                                                   │
│         ├─────────────────┬─────────────────┬─────────────────┐            │
│         ▼                 ▼                 ▼                 ▼            │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐    │
│  │  Public  │      │Protected │      │ Private  │      │ + Share  │    │
│  │  Memo    │      │  Memo    │      │  Memo    │      │  Token   │    │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘    │
│       │                 │                 │                 │            │
│       ▼                 ▼                 ▼                 ▼            │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  任何人可访问        登录用户可访问     仅创建者/管理员     临时授权  │    │
│  │  (无需登录)                                                      │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.5 绑定前后权限对比表

| 维度 | 绑定前 (MemoID = nil) | 绑定后 (MemoID != nil) |
|------|----------------------|------------------------|
| **权限依据** | 附件创建者身份 | 关联 Memo 的可见性 |
| **未登录用户** | ❌ 拒绝 | Public: ✅ 允许<br>其他: ❌ 拒绝 |
| **创建者本人** | ✅ 允许 | ✅ 始终允许 |
| **其他登录用户** | ❌ 拒绝 | Public: ✅ 允许<br>Protected: ✅ 允许<br>Private: ❌ 拒绝 |
| **管理员** | ✅ 允许 | ✅ 始终允许 |
| **Share Token** | ❌ 不支持 | ✅ 支持临时访问 |
| **访问方式** | 只能通过 API 操作 | 可通过 API + 文件服务访问 |

### 3.6 特殊场景：Motion Photo 分组绑定

**文件位置**: `server/router/api/v1/memo_attachment_service.go:normalizeMemoAttachmentRequest`

对于动态照片（Motion Photo / Live Photo），系统会自动处理分组：

```go
// Motion Photo 通常包含两个文件：静态图 + 视频
// 它们通过 GroupID 关联

requestGroups := make(map[string][]*store.Attachment)
for _, attachment := range requestedAttachments {
  motion := getAttachmentMotionMedia(attachment)
  if motion == nil || motion.GroupId == "" {
    continue
  }
  // 按 GroupID 分组
  requestGroups[motion.GroupId] = append(requestGroups[motion.GroupId], attachment)
}

// 绑定逻辑：如果是多成员分组，需要所有成员都被请求才会绑定
if isMultiMemberMotionGroup(currentGroup) && !allGroupMembersRequested(currentGroup, requestNamesByGroup[groupID]) {
  appendedGroups[groupID] = true
  continue  // 跳过，不绑定部分成员
}
```

**设计意图**：防止 Motion Photo 的静态图和视频被分开绑定到不同的 Memo，保持媒体完整性。

---

## 四、多入口鉴权边界差异分析

### 4.1 系统入口架构

系统有三个主要的 API 入口，每个入口有不同的鉴权策略：

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              客户端请求                                      │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
│  Connect RPC      │     │  gRPC-Gateway     │     │  HTTP 文件服务    │
│  (浏览器前端)      │     │  (API 网关)       │     │  (文件下载)        │
│                   │     │                   │     │                   │
│  Path:            │     │  Path:            │     │  Path:            │
│  /memos.api.v1.*  │     │  /api/v1/*        │     │  /file/*          │
│  /memos.api.v1.*  │     │  /file/*          │     │                   │
└─────────┬─────────┘     └─────────┬─────────┘     └─────────┬─────────┘
          │                           │                           │
          ▼                           ▼                           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         鉴权策略差异                                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Connect RPC / gRPC-Gateway:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. 网关层中间件检查: IsPublicMethod() 白名单                        │   │
│  │  2. 不在白名单 → 必须携带有效凭证                                     │   │
│  │  3. 附件服务不在白名单 → 所有请求必须先认证                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  HTTP 文件服务:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. 网关层放行（无法确定 RPC 方法名）                                 │   │
│  │  2. 完全在服务层 checkAttachmentPermission() 中检查                  │   │
│  │  3. 支持 Share Token 临时访问（API 层不支持）                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 入口一：Connect RPC（浏览器前端）

**文件位置**: `server/router/api/v1/v1.go:166`

```go
// 路径匹配
connectGroup.Any("/memos.api.v1.*", echo.WrapHandler(http.MaxBytesHandler(connectMux, maxAPIRequestBytes)))

// 中间件链
connectInterceptors := connect.WithInterceptors(
  NewMetadataInterceptor(),
  NewLoggingInterceptor(logStacktraces),
  NewRecoveryInterceptor(logStacktraces),
  NewAuthInterceptor(s.Store, s.Secret),  // 认证拦截器
)
```

**鉴权逻辑**：`NewAuthInterceptor` 使用 `PublicMethods` 白名单

```go
// server/router/api/v1/acl_config.go
var PublicMethods = map[string]struct{}{
  // Auth 相关
  "/memos.api.v1.AuthService/SignIn": {},
  "/memos.api.v1.AuthService/RefreshToken": {},
  
  // Instance 相关
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
  
  // 注意: AttachmentService 的方法不在此列表中！
}
```

**关键结论**：
- `AttachmentService` 的所有方法（`GetAttachment`, `CreateAttachment`, `DeleteAttachment` 等）**不在白名单中**
- 所有附件 API 请求**必须携带有效认证凭证**才能通过网关层
- 未认证请求会直接返回 `16 Unauthenticated`

### 4.3 入口二：gRPC-Gateway（API 网关）

**文件位置**: `server/router/api/v1/v1.go:129-138`

```go
// 路径匹配
gwGroup.Any("/api/v1/*", handler)
gwGroup.Any("/file/*", handler)  // 注意：/file/* 也走这个网关

// 网关认证中间件
gatewayAuthMiddleware := func(next runtime.HandlerFunc) runtime.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request, pathParams map[string]string) {
    ctx := r.Context()
    
    // 获取 RPC 方法名（由 grpc-gateway 在路由后设置）
    rpcMethod, ok := runtime.RPCMethod(ctx)
    
    // 提取凭证并认证
    authHeader := r.Header.Get("Authorization")
    result := authenticator.Authenticate(ctx, authHeader)
    
    // 关键逻辑：
    // 1. 如果认证成功 (result != nil) → 放行
    // 2. 如果认证失败，但无法确定 RPC 方法 (!ok) → 放行（服务层处理）
    // 3. 如果认证失败，且方法不在白名单 → 拒绝
    
    if result == nil && ok && !IsPublicMethod(rpcMethod) {
      http.Error(w, `{"code": 16, "message": "authentication required"}`, http.StatusUnauthorized)
      return
    }
    
    // 应用认证结果到上下文
    if result != nil {
      ctx = auth.ApplyToContext(ctx, result)
      r = r.WithContext(ctx)
    }
    
    next(w, r, pathParams)
  }
}
```

**关键分析**：

| 场景 | `ok` (是否能确定 RPC 方法) | 行为 |
|------|---------------------------|------|
| 标准 API 请求 (`/api/v1/attachments/123`) | `true` | 检查 `IsPublicMethod`，附件服务不在白名单 → 需认证 |
| 文件服务请求 (`/file/attachments/uid/name`) | `false` | 无法确定 RPC 方法 → **放行到服务层** |

**重要发现**：
- `/file/*` 路径虽然注册到了 gRPC-Gateway，但由于文件服务不是标准 gRPC 方法，`runtime.RPCMethod()` 无法获取方法名
- 因此**文件服务的所有请求都会被网关层放行**，权限检查完全在 `fileserver.go` 中实现

### 4.4 入口三：HTTP 文件服务（独立权限检查）

**文件位置**: `server/router/fileserver/fileserver.go:checkAttachmentPermission`

文件服务有独立的、完整的权限检查逻辑：

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, c *echo.Context, attachment *store.Attachment) error {
  // 场景 1: 未关联 Memo 的附件
  if attachment.MemoID == nil {
    user, err := s.getCurrentUser(ctx, c)
    if user == nil {
      return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
    }
    if user.ID != attachment.CreatorID && user.Role != store.RoleAdmin {
      return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
    }
    return nil
  }

  // 场景 2: 关联 Memo 的附件
  memo, _ := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
  
  // Public Memo: 任何人可访问
  if memo.Visibility == store.Public {
    return nil
  }

  // 特性：Share Token 临时访问（API 层不支持）
  if shareToken := c.QueryParam("share_token"); shareToken != "" {
    ms, err := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
    if err == nil && ms != nil && !isMemoShareExpired(ms) && ms.MemoID == memo.ID {
      return nil  // 有效 Share Token → 允许访问
    }
  }

  // 需要登录
  user, _ := s.getCurrentUser(ctx, c)
  if user == nil {
    return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
  }

  // Private Memo: 仅创建者/管理员
  if memo.Visibility == store.Private && user.ID != memo.CreatorID && user.Role != store.RoleAdmin {
    return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
  }

  return nil
}
```

### 4.5 三入口鉴权对比表

| 维度 | Connect RPC | gRPC-Gateway (API) | HTTP 文件服务 |
|------|-------------|---------------------|---------------|
| **路径** | `/memos.api.v1.*` | `/api/v1/*` | `/file/attachments/*` |
| **网关层鉴权** | ✅ 严格检查 | ✅ 严格检查 | ❌ 放行（方法名不确定） |
| **白名单依赖** | `PublicMethods` | `PublicMethods` | 不依赖 |
| **附件服务** | 不在白名单 → 需认证 | 不在白名单 → 需认证 | 网关放行 |
| **服务层鉴权** | `checkAttachmentAccess` | `checkAttachmentAccess` | `checkAttachmentPermission` |
| **Share Token** | ❌ 不支持 | ❌ 不支持 | ✅ 支持 |
| **Public Memo 附件** | 网关层拒绝（无认证） | 网关层拒绝（无认证） | 服务层允许 |

### 4.6 边界场景分析

#### 场景 1：未登录用户访问 Public Memo 的附件

**请求路径**: `GET /file/attachments/abc123/photo.jpg`

**鉴权流程**:
1. **网关层**: `runtime.RPCMethod()` 返回 `ok = false` → 放行
2. **服务层** (`checkAttachmentPermission`):
   - `attachment.MemoID != nil` → 关联了 Memo
   - `memo.Visibility == Public` → 返回 `nil`（允许）

**结果**: ✅ 访问成功

**对比**：如果通过 API 层 `GetAttachment` 访问：
- 网关层：`AttachmentService` 不在白名单 → 需要认证
- 未登录请求直接返回 `Unauthenticated`

#### 场景 2：使用 Share Token 访问 Private Memo 的附件

**请求路径**: `GET /file/attachments/abc123/photo.jpg?share_token=xyz789`

**鉴权流程**:
1. **网关层**: 放行
2. **服务层**:
   - `memo.Visibility == Private`
   - 检查 `share_token` 参数
   - Token 有效且对应正确的 Memo → 允许访问

**结果**: ✅ 访问成功

**对比**：API 层 `GetAttachment` 不支持 `share_token` 参数，无法通过此方式访问。

#### 场景 3：未登录用户访问未绑定的附件

**请求路径**: `GET /file/attachments/abc123/photo.jpg`（附件 `MemoID = nil`）

**鉴权流程**:
1. **网关层**: 放行
2. **服务层**:
   - `attachment.MemoID == nil`
   - `getCurrentUser()` 返回 `nil`（未登录）
   - 返回 `Unauthorized`

**结果**: ❌ 访问被拒绝

### 4.7 鉴权边界总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         鉴权边界关键差异                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  网关层 vs 服务层的分工:                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   网关层 (AuthInterceptor / gatewayAuthMiddleware)                   │  │
│  │   ┌──────────────────────────────────────────────────────────────┐   │  │
│  │   │  职责: 认证凭证验证（是否登录）                                │   │  │
│  │   │  依据: PublicMethods 白名单                                   │   │  │
│  │   │  特点: 粗粒度，只区分"公开/非公开"方法                        │   │  │
│  │   └──────────────────────────────────────────────────────────────┘   │  │
│  │                              ↓                                         │  │
│  │   服务层 (checkAttachmentAccess / checkAttachmentPermission)          │  │
│  │   ┌──────────────────────────────────────────────────────────────┐   │  │
│  │   │  职责: 具体业务权限检查                                        │   │  │
│  │   │  依据: 附件状态、Memo 可见性、用户身份、Share Token           │   │  │
│  │   │  特点: 细粒度，完整的权限决策逻辑                              │   │  │
│  │   └──────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  文件服务的特殊性:                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   由于 /file/* 路径无法被 grpc-gateway 识别为标准 RPC 方法，        │  │
│  │   网关层会放行所有请求，完全依赖服务层的权限检查。                    │  │
│  │                                                                       │  │
│  │   这导致了一个重要的边界差异:                                        │  │
│   │                                                                       │  │
│  │   - Public Memo 的附件: 通过文件服务可匿名访问                       │  │
│  │   - 通过 API GetAttachment: 需要先登录（网关层拦截）                 │  │
│  │                                                                       │  │
│  │   设计意图:                                                           │  │
│  │   - 文件服务用于直接嵌入到网页、邮件中的资源链接                      │  │
│  │   - 这些场景下用户可能没有登录状态                                    │  │
│  │   - 但业务权限仍由服务层严格控制                                      │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、删除权限：单个 vs 批量操作的差异

### 5.1 单个删除 (DeleteAttachment)

**文件位置**: `server/router/api/v1/attachment_service.go:336-365`

```go
func (s *APIV1Service) DeleteAttachment(ctx context.Context, request *v1pb.DeleteAttachmentRequest) (*emptypb.Empty, error) {
  attachmentUID, err := ExtractAttachmentUIDFromName(request.Name)
  // ...
  
  user, err := s.fetchCurrentUser(ctx)
  // ... 必须登录
  
  // 关键：查询时直接限制 CreatorID
  attachment, err := s.Store.GetAttachment(ctx, &store.FindAttachment{
    UID:       &attachmentUID,
    CreatorID: &user.ID,  // ← 隐式权限控制
  })
  
  if attachment == nil {
    return nil, status.Errorf(codes.NotFound, "attachment not found")
  }
  
  // 删除操作
  s.Store.DeleteAttachment(ctx, &store.DeleteAttachment{ID: attachment.ID})
  
  return &emptypb.Empty{}, nil
}
```

**核心特点**：
1. **隐式权限控制**：通过查询条件 `CreatorID = user.ID` 实现
2. **管理员也受限**：即使是管理员，查询时也被限制为只能查找自己创建的附件
3. **错误类型**：尝试删除他人附件时返回 `NotFound`，不是 `PermissionDenied`

### 5.2 批量删除 (BatchDeleteAttachments)

**文件位置**: `server/router/api/v1/attachment_service.go:367-415`

```go
func (s *APIV1Service) BatchDeleteAttachments(ctx context.Context, request *v1pb.BatchDeleteAttachmentsRequest) (*emptypb.Empty, error) {
  user, err := s.fetchCurrentUser(ctx)
  // ... 必须登录
  
  attachments := make([]*store.Attachment, 0, len(request.Names))
  
  for _, name := range request.Names {
    attachmentUID, _ := ExtractAttachmentUIDFromName(name)
    
    // 关键：查询时不限制 CreatorID
    attachment, err := s.Store.GetAttachment(ctx, &store.FindAttachment{UID: &attachmentUID})
    // ...
    
    // 显式权限检查
    if attachment.CreatorID != user.ID && !isSuperUser(user) {
      return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
    
    attachments = append(attachments, attachment)
  }
  
  // 批量删除
  s.Store.DeleteAttachments(ctx, attachments)
  
  return &emptypb.Empty{}, nil
}
```

**核心特点**：
1. **显式权限控制**：查询后单独检查 `attachment.CreatorID != user.ID && !isSuperUser(user)`
2. **管理员有权限**：管理员可以删除任意用户的附件
3. **错误类型**：权限不足时返回 `PermissionDenied`

### 5.3 两种删除方式对比

| 维度 | 单个删除 (DeleteAttachment) | 批量删除 (BatchDeleteAttachments) |
|------|----------------------------|-----------------------------------|
| **查询条件** | `UID + CreatorID = user.ID` | 仅 `UID` |
| **权限控制方式** | 隐式（查询条件过滤） | 显式（查询后检查） |
| **管理员能力** | ❌ 无法删除他人附件 | ✅ 可以删除任意附件 |
| **越权时返回** | `NotFound` (5) | `PermissionDenied` (7) |
| **原子性** | 单条操作 | 全有或全无（循环中任一失败则全部终止） |
| **适用场景** | 用户自主管理个人附件 | 管理员批量管理、用户清理自己的多个附件 |

### 5.4 行为差异流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    删除权限行为差异                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  单个删除 (DeleteAttachment):                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                      │  │
│  │   请求: DELETE /memos.api.v1.AttachmentService/DeleteAttachment    │  │
│  │   参数: name = "attachments/att_other_user"                        │  │
│  │                                                                      │  │
│  │                              ↓                                       │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │  Store.GetAttachment(ctx, &FindAttachment{                   │  │  │
│  │   │      UID:       &attachmentUID,                               │  │  │
│  │   │      CreatorID: &user.ID,  // ← 关键：只能查自己的           │  │  │
│  │   │  })                                                           │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              ↓                                       │  │
│  │   结果: attachment == nil                                           │  │
│  │   返回: NotFound ("attachment not found")                           │  │
│  │                                                                      │  │
│  │   注意: 管理员执行此 API 也会得到同样结果！                         │  │
│  │                                                                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  批量删除 (BatchDeleteAttachments):                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                      │  │
│  │   请求: POST /memos.api.v1.AttachmentService/BatchDeleteAttachments│  │
│  │   参数: names = ["attachments/att_other_user"]                     │  │
│  │                                                                      │  │
│  │                              ↓                                       │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │  Store.GetAttachment(ctx, &FindAttachment{                   │  │  │
│  │   │      UID: &attachmentUID,  // ← 不限制 CreatorID            │  │  │
│  │   │  })                                                           │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              ↓                                       │  │
│  │   结果: attachment != nil (找到了)                                  │  │
│  │                              ↓                                       │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │  显式检查:                                                    │  │  │
│  │   │  if attachment.CreatorID != user.ID && !isSuperUser(user) { │  │  │
│  │   │      return PermissionDenied                                 │  │  │
│  │   │  }                                                           │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              ↓                                       │  │
│  │   普通用户: 返回 PermissionDenied                                   │  │
│  │   管理员:   检查通过 → 执行删除                                     │  │
│  │                                                                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 管理员删除附件的正确方式

由于 `DeleteAttachment` API 限制了只能删除自己的附件，管理员需要使用 `BatchDeleteAttachments` 来删除其他用户的附件：

```go
// 管理员删除他人附件的正确姿势：

// ❌ 错误方式：DeleteAttachment 会返回 NotFound
s.DeleteAttachment(ctx, &v1pb.DeleteAttachmentRequest{
  Name: "attachments/att_other_user",
})
// 结果: NotFound (因为查询时加了 CreatorID 限制)

// ✅ 正确方式：使用 BatchDeleteAttachments
s.BatchDeleteAttachments(ctx, &v1pb.BatchDeleteAttachmentsRequest{
  Names: []string{"attachments/att_other_user"},
})
// 结果: 
// - 管理员: 删除成功
// - 普通用户: PermissionDenied
```

### 5.6 设计意图分析

**为什么单个删除和批量删除有不同的权限模型？**

| 设计考量 | 单个删除 | 批量删除 |
|---------|---------|---------|
| **使用场景** | 用户在 UI 中删除单个附件 | 管理员后台操作、用户批量清理 |
| **安全性** | 最严格的隐式控制，防止越权猜测 UID | 显式检查，兼顾管理员需求 |
| **错误信息** | `NotFound` 不暴露附件是否存在 | `PermissionDenied` 明确告知权限不足 |
| **原子性** | 天然单条操作 | 需要保证批量一致性 |

**安全设计亮点**：
1. 单个删除使用 `NotFound` 而不是 `PermissionDenied`，可以防止攻击者通过枚举 UID 来确认系统中是否存在某个附件（信息泄露防护）
2. 批量删除用于管理员场景，需要明确的权限反馈，所以使用 `PermissionDenied`

### 5.7 通过 SetMemoAttachments 间接删除

**文件位置**: `server/router/api/v1/memo_attachment_service.go:71-84`

还有一种删除附件的方式是通过 `SetMemoAttachments` 解绑时删除：

```go
// 删除不在请求中的附件（解绑即删除）
for _, attachment := range currentAttachments {
  if !requestedIDs[attachment.ID] {
    // 权限检查：不能删除其他用户的附件
    if attachment.CreatorID != user.ID && !isSuperUser(user) {
      return status.Errorf(codes.PermissionDenied, "cannot remove another user's attachment")
    }
    // 执行删除
    s.Store.DeleteAttachment(ctx, &store.DeleteAttachment{
      ID:     int32(attachment.ID),
      MemoID: &memo.ID,
    })
  }
}
```

**特点**：
- 权限模型与 `BatchDeleteAttachments` 一致：显式检查，管理员可以删除他人附件
- 这是管理员可以删除已绑定到 Memo 的附件的另一种方式

---

## 六、资源访问控制逻辑（核心总结）

### 6.1 访问入口

资源访问有两个主要入口：

1. **gRPC API 层**: 通过 `AttachmentService` 访问附件元数据
   - `GetAttachment` - 获取单个附件信息
   - `ListAttachments` - 列出用户的附件
   - `ListMemoAttachments` - 列出 Memo 的附件

2. **HTTP 文件服务器**: 直接访问文件内容
   - 路由: `/file/attachments/:uid/:filename`
   - 实现: `server/router/fileserver/fileserver.go`

### 6.2 权限检查核心逻辑

#### 6.2.1 HTTP 文件访问权限检查

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

#### 6.2.2 API 层权限检查

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

### 6.3 权限决策矩阵

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

### 6.4 用户身份认证

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

### 6.5 公开端点配置

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

### 6.6 文件服务安全防护

#### 6.6.1 XSS 防护

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

#### 6.6.2 安全响应头

所有文件响应都设置安全头：

```go
func setSecurityHeaders(c *echo.Context) {
  h := c.Response().Header()
  h.Set("X-Content-Type-Options", "nosniff")     // 防止 MIME 类型嗅探
  h.Set("X-Frame-Options", "DENY")               // 防止 Clickjacking
  h.Set("Content-Security-Policy", "default-src 'none'; style-src 'unsafe-inline';")
}
```

#### 6.6.3 缓存控制

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

## 七、文件服务实现细节

### 7.1 文件服务路由

**文件位置**: `server/router/fileserver/fileserver.go`

```go
func (s *FileServerService) RegisterRoutes(echoServer *echo.Echo) {
  fileGroup := echoServer.Group("/file")
  fileGroup.GET("/attachments/:uid/:filename", s.serveAttachmentFile)  // 附件文件
  fileGroup.GET("/users/:identifier/avatar", s.serveUserAvatar)         // 用户头像
}
```

### 7.2 附件文件服务流程

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

### 7.3 不同存储类型的文件服务

#### 7.3.1 流式传输（视频/音频）

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

#### 7.3.2 静态文件服务

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

### 7.4 缩略图生成

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

### 7.5 动态照片（Motion Photo）支持

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

## 八、附件数据模型

### 8.1 Attachment 结构体

**文件位置**: `