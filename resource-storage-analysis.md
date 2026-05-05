# Memos 资源存储机制分析报告

## 1. 概述

Memos 支持三种资源存储方式：本地文件系统（LOCAL）、S3 兼容对象存储（S3）和外部链接（EXTERNAL）。本报告深入分析资源上传路径、存储后端切换机制和下载 URL 生成的实现原理。

---

## 2. 存储类型定义

### 2.1 实例级存储类型（InstanceStorageSetting）

定义在 `proto/store/instance_setting.proto`：

```protobuf
message InstanceStorageSetting {
  enum StorageType {
    STORAGE_TYPE_UNSPECIFIED = 0;
    DATABASE = 1;  // 数据库存储（默认回退）
    LOCAL = 2;     // 本地文件系统
    S3 = 3;        // S3 兼容对象存储
  }
  StorageType storage_type = 1;
  string filepath_template = 2;       // 文件路径模板
  int64 upload_size_limit_mb = 3;     // 上传大小限制（MB）
  StorageS3Config s3_config = 4;      // S3 配置
}
```

默认值（`store/instance_setting.go:241-244`）：
- `defaultInstanceStorageType = LOCAL`
- `defaultInstanceUploadSizeLimitMb = 30`
- `defaultInstanceFilepathTemplate = "assets/{timestamp}_{uuid}_{filename}"`

### 2.2 附件级存储类型（AttachmentStorageType）

定义在 `proto/store/attachment.proto`：

```protobuf
enum AttachmentStorageType {
  ATTACHMENT_STORAGE_TYPE_UNSPECIFIED = 0;
  LOCAL = 1;       // 本地文件系统
  S3 = 2;          // S3 兼容对象存储
  EXTERNAL = 3;    // 外部链接（URL 形式）
}
```

### 2.3 数据模型

Attachment 结构（`store/attachment.go:16-41`）：

```go
type Attachment struct {
  ID          int32
  UID         string
  CreatorID   int32
  CreatedTs   int64
  UpdatedTs   int64
  Filename    string
  Blob        []byte                       // DATABASE 模式存储内容
  Type        string                       // MIME 类型
  Size        int64
  StorageType storepb.AttachmentStorageType // 存储类型
  Reference   string                       // 存储引用（路径/URL/预签名链接）
  Payload     *storepb.AttachmentPayload   // 附加载荷（S3 配置等）
  MemoID      *int32
  MemoUID     *string
}
```

AttachmentPayload 结构：

```protobuf
message AttachmentPayload {
  oneof payload {
    S3Object s3_object = 1;
  }
  MotionMedia motion_media = 10;

  message S3Object {
    StorageS3Config s3_config = 1;           // S3 配置快照
    string key = 2;                           // S3 对象 Key
    google.protobuf.Timestamp last_presigned_time = 3;  // 上次预签名时间
  }
}
```

---

## 3. 资源上传路径分析

### 3.1 完整上传链路

```
前端上传服务
    ↓
uploadService.uploadFiles()
    ↓
attachmentServiceClient.createAttachment()
    ↓
CreateAttachment API (gRPC/Connect)
    ↓
SaveAttachmentBlob()  [核心存储路由]
    ↓
┌─────────────────────────────────────────────────────────┐
│              根据 InstanceStorageSetting 路由             │
├─────────────┬─────────────────┬─────────────────────────┤
│   LOCAL     │      S3         │    DATABASE（默认）     │
├─────────────┼─────────────────┼─────────────────────────┤
│ os.WriteFile │ s3.UploadObject│ 直接存储在 Blob 字段    │
│ + Reference │ + PresignGetURL │                         │
│ 设置为路径  │ + Reference 设  │                         │
│             │ 为预签名URL      │                         │
└─────────────┴─────────────────┴─────────────────────────┘
```

### 3.2 前端上传服务

`web/src/components/MemoEditor/services/uploadService.ts`：

```typescript
export const uploadService = {
  async uploadFiles(localFiles: LocalFile[]): Promise<Attachment[]> {
    const attachments: Attachment[] = [];
    for (const localFile of localFiles) {
      const { file, motionMedia } = localFile;
      const buffer = new Uint8Array(await file.arrayBuffer());
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
  },
};
```

### 3.3 后端上传处理

`server/router/api/v1/attachment_service.go:68-199` 的 `CreateAttachment` 流程：

1. **用户认证**：获取当前登录用户
2. **参数验证**：
   - 文件名验证（防止路径遍历）
   - MIME 类型标准化
   - UID 生成验证
3. **EXIF 剥离**：对包含 EXIF 的图片（JPEG、TIFF、WebP、HEIC、HEIF）进行隐私保护处理
4. **存储处理**：调用 `SaveAttachmentBlob()` 保存文件内容
5. **元数据持久化**：调用 `s.Store.CreateAttachment()` 保存到数据库

### 3.4 核心存储路由函数：SaveAttachmentBlob

`server/router/api/v1/attachment_service.go:438-522`：

```go
func SaveAttachmentBlob(ctx context.Context, profile *profile.Profile, 
                         stores *store.Store, create *store.Attachment) error {
  instanceStorageSetting, err := stores.GetInstanceStorageSetting(ctx)
  
  if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_LOCAL {
    // 本地存储处理
    filepathTemplate := instanceStorageSetting.FilepathTemplate
    if filepathTemplate == "" {
      filepathTemplate = "assets/{timestamp}_{uuid}_{filename}"
    }
    
    // 路径模板替换
    internalPath = replaceFilenameWithPathTemplate(filepathTemplate, create.Filename)
    
    // 确保路径唯一
    osPath = ensureUniqueLocalAttachmentPath(osPath, create.UID)
    
    // 写入文件
    os.WriteFile(osPath, create.Blob, 0644)
    
    // 设置引用
    create.Reference = internalPath
    create.Blob = nil
    create.StorageType = storepb.AttachmentStorageType_LOCAL
    
  } else if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_S3 {
    // S3 存储处理
    s3Client, _ := s3.NewClient(ctx, s3Config)
    
    // 路径模板处理
    filepathTemplate = replaceFilenameWithPathTemplate(filepathTemplate, create.Filename)
    
    // 上传到 S3
    key, _ := s3Client.UploadObject(ctx, filepathTemplate, create.Type, 
                                     bytes.NewReader(create.Blob))
    
    // 生成预签名 URL（有效期 5 天）
    presignURL, _ := s3Client.PresignGetObject(ctx, key)
    
    // 设置引用和元数据
    create.Reference = presignURL
    create.Blob = nil
    create.StorageType = storepb.AttachmentStorageType_S3
    
    // 保存 S3 对象信息到 Payload
    payload.Payload = &storepb.AttachmentPayload_S3Object_{
      S3Object: &storepb.AttachmentPayload_S3Object{
        S3Config:          s3Config,
        Key:               key,
        LastPresignedTime: timestamppb.New(time.Now()),
      },
    }
    create.Payload = payload
  }
  
  // DATABASE 模式：不处理，Blob 直接存入数据库
  return nil
}
```

### 3.5 路径模板变量

`server/router/api/v1/attachment_service.go:577-606`：

```go
func replaceFilenameWithPathTemplate(path, filename string) string {
  path = fileKeyPattern.ReplaceAllStringFunc(path, func(s string) string {
    switch s {
    case "{filename}":  return filename
    case "{timestamp}": return fmt.Sprintf("%d", t.Unix())
    case "{year}":      return fmt.Sprintf("%d", t.Year())
    case "{month}":     return fmt.Sprintf("%02d", t.Month())
    case "{day}":       return fmt.Sprintf("%02d", t.Day())
    case "{hour}":      return fmt.Sprintf("%02d", t.Hour())
    case "{minute}":    return fmt.Sprintf("%02d", t.Minute())
    case "{second}":    return fmt.Sprintf("%02d", t.Second())
    case "{uuid}":      return util.GenUUID()
    default:            return s
    }
  })
  return path
}
```

---

## 4. 存储后端切换机制

### 4.1 配置存储

实例存储设置通过 `InstanceSetting` 表管理，Key 为 `STORAGE`：

`store/instance_setting.go:26-63` 的 `UpsertInstanceSetting`：

```go
func (s *Store) UpsertInstanceSetting(ctx context.Context, 
                                        upsert *storepb.InstanceSetting) (*storepb.InstanceSetting, error) {
  if upsert.Key == storepb.InstanceSettingKey_STORAGE {
    valueBytes, err = protojson.Marshal(upsert.GetStorageSetting())
  }
  // ... 存入数据库
  s.instanceSettingCache.Set(ctx, instanceSetting.Key.String(), instanceSetting)
}
```

### 4.2 配置读取与缓存

`store/instance_setting.go:247-273` 的 `GetInstanceStorageSetting`：

```go
func (s *Store) GetInstanceStorageSetting(ctx context.Context) (*storepb.InstanceStorageSetting, error) {
  // 首先检查缓存
  if cache, ok := s.instanceSettingCache.Get(ctx, storepb.InstanceSettingKey_STORAGE.String()); ok {
    return cache.(*storepb.InstanceSetting).GetStorageSetting(), nil
  }
  
  // 从数据库读取
  instanceSetting, err := s.GetInstanceSetting(ctx, &FindInstanceSetting{
    Name: storepb.InstanceSettingKey_STORAGE.String(),
  })
  
  instanceStorageSetting := &storepb.InstanceStorageSetting{}
  if instanceSetting != nil {
    instanceStorageSetting = instanceSetting.GetStorageSetting()
  }
  
  // 应用默认值
  if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_STORAGE_TYPE_UNSPECIFIED {
    instanceStorageSetting.StorageType = defaultInstanceStorageType  // LOCAL
  }
  if instanceStorageSetting.UploadSizeLimitMb == 0 {
    instanceStorageSetting.UploadSizeLimitMb = defaultInstanceUploadSizeLimitMb  // 30
  }
  if instanceStorageSetting.FilepathTemplate == "" {
    instanceStorageSetting.FilepathTemplate = defaultInstanceFilepathTemplate
  }
  
  // 更新缓存
  s.instanceSettingCache.Set(ctx, storepb.InstanceSettingKey_STORAGE.String(), &storepb.InstanceSetting{
    Key:   storepb.InstanceSettingKey_STORAGE,
    Value: &storepb.InstanceSetting_StorageSetting{StorageSetting: instanceStorageSetting},
  })
  
  return instanceStorageSetting, nil
}
```

### 4.3 切换机制特点

1. **运行时动态读取**：每次上传时调用 `GetInstanceStorageSetting()` 获取当前配置
2. **缓存机制**：使用 `s.instanceSettingCache` 减少数据库查询
3. **向后兼容**：已上传的附件不受后端切换影响，因为每个附件自身记录 `StorageType` 和 `Reference`
4. **新上传使用新配置**：切换存储后端后，**新上传**的资源才会使用新的存储位置

### 4.4 存储类型与引用字段的关系

| StorageType | Reference 字段内容 | Payload.S3Object |
|-------------|-------------------|------------------|
| LOCAL       | 相对路径（如 `assets/123_uuid_file.jpg`） | 无 |
| S3          | 预签名 URL（有效期 5 天） | 有（Key、S3Config、LastPresignedTime） |
| EXTERNAL    | 外部 URL | 无 |
| DATABASE    | 空或未使用 | 无（Blob 字段存内容） |

---

## 5. 下载 URL 生成机制

### 5.1 文件服务路由

`server/router/fileserver/fileserver.go:120-125`：

```go
func (s *FileServerService) RegisterRoutes(echoServer *echo.Echo) {
  fileGroup := echoServer.Group("/file")
  fileGroup.GET("/attachments/:uid/:filename", s.serveAttachmentFile)
  fileGroup.GET("/users/:identifier/avatar", s.serveUserAvatar)
}
```

### 5.2 附件下载处理流程

`server/router/fileserver/fileserver.go:132-165` 的 `serveAttachmentFile`：

```go
func (s *FileServerService) serveAttachmentFile(c *echo.Context) error {
  uid := c.Param("uid")
  wantThumbnail := c.QueryParam("thumbnail") == "true"
  wantMotion := c.QueryParam("motion") == "true"
  
  // 1. 读取附件元数据
  attachment, _ := s.Store.GetAttachment(ctx, &store.FindAttachment{
    UID:     &uid,
    GetBlob: true,
  })
  
  // 2. 权限检查
  s.checkAttachmentPermission(ctx, c, attachment)
  
  // 3. 动态内容类型处理（XSS 防护）
  contentType := s.sanitizeContentType(attachment.Type)
  
  // 4. 媒体文件流处理
  if isMediaType(attachment.Type) {  // video/* 或 audio/*
    return s.serveMediaStream(c, attachment, contentType)
  }
  
  // 5. 静态文件处理（图片、文档等）
  return s.serveStaticFile(c, attachment, contentType, wantThumbnail)
}
```

### 5.3 媒体文件流服务（视频/音频）

`server/router/fileserver/fileserver.go:204-230`：

```go
func (s *FileServerService) serveMediaStream(c *echo.Context, attachment *store.Attachment, 
                                              contentType string) error {
  switch attachment.StorageType {
  case storepb.AttachmentStorageType_LOCAL:
    filePath, _ := s.resolveLocalPath(attachment.Reference)
    http.ServeFile(c.Response(), c.Request(), filePath)  // 支持 Range 请求
    return nil
    
  case storepb.AttachmentStorageType_S3:
    // 媒体文件直接重定向到 S3 预签名 URL
    presignURL, _ := s.getS3PresignedURL(c.Request().Context(), attachment)
    return c.Redirect(http.StatusTemporaryRedirect, presignURL)
    
  default:  // DATABASE
    modTime := time.Unix(attachment.UpdatedTs, 0)
    http.ServeContent(c.Response(), c.Request(), attachment.Filename, 
                      modTime, bytes.NewReader(attachment.Blob))
    return nil
  }
}
```

### 5.4 静态文件服务

`server/router/fileserver/fileserver.go:232-273`：

```go
func (s *FileServerService) serveStaticFile(c *echo.Context, attachment *store.Attachment,
                                              contentType string, wantThumbnail bool) error {
  // 缩略图处理
  if wantThumbnail && thumbnailSupportedTypes[attachment.Type] {
    if thumbnailBlob, err := s.getOrGenerateThumbnail(ctx, attachment); err == nil {
      return c.Blob(http.StatusOK, "image/jpeg", thumbnailBlob)
    }
  }
  
  // 非媒体文件强制下载
  if !strings.HasPrefix(contentType, "image/") && contentType != "application/pdf" {
    c.Response().Header().Set(echo.HeaderContentDisposition, 
                               fmt.Sprintf("attachment; filename=%q", attachment.Filename))
  }
  
  switch attachment.StorageType {
  case storepb.AttachmentStorageType_LOCAL:
    filePath, _ := s.resolveLocalPath(attachment.Reference)
    http.ServeFile(c.Response(), c.Request(), filePath)
    return nil
    
  case storepb.AttachmentStorageType_S3:
    // 非媒体文件：通过服务器流式代理返回
    reader, _ := s.getAttachmentReader(c.Request().Context(), attachment)
    defer reader.Close()
    return c.Stream(http.StatusOK, contentType, reader)
    
  default:  // DATABASE
    return c.Blob(http.StatusOK, contentType, attachment.Blob)
  }
}
```

### 5.5 下载 URL 格式

前端使用的附件 URL 格式：
- **本地文件**：`/file/attachments/{uid}/{filename}`
- **S3 存储**：同一路由，但服务器内部处理不同
  - 媒体文件：307 重定向到 S3 预签名 URL
  - 非媒体文件：服务器代理流式返回
- **外部链接**：直接使用 `attachment.ExternalLink`（即 `Reference` 字段）

### 5.6 前端附件链接转换

`server/router/api/v1/attachment_service.go:417-435`：

```go
func convertAttachmentFromStore(attachment *store.Attachment) *v1pb.Attachment {
  attachmentMessage := &v1pb.Attachment{
    Name:        fmt.Sprintf("%s%s", AttachmentNamePrefix, attachment.UID),
    Filename:    attachment.Filename,
    Type:        attachment.Type,
    Size:        attachment.Size,
  }
  
  // S3 和 EXTERNAL 存储类型设置 ExternalLink
  if attachment.StorageType == storepb.AttachmentStorageType_EXTERNAL || 
     attachment.StorageType == storepb.AttachmentStorageType_S3 {
    attachmentMessage.ExternalLink = attachment.Reference
  }
  
  return attachmentMessage
}
```

---

## 6. S3 预签名 URL 自动刷新机制

### 6.1 背景

S3 预签名 URL 有效期为 5 天（`internal/storage/s3/s3.go:66`），因此需要定期刷新。

### 6.2 后台 Runner

`server/runner/s3presign/runner.go`：

```go
type Runner struct {
  Store *store.Store
}

// 每 12 小时运行一次
const runnerInterval = time.Hour * 12

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

### 6.3 刷新逻辑

`server/runner/s3presign/runner.go:46-133`：

```go
func (r *Runner) CheckAndPresign(ctx context.Context) {
  instanceStorageSetting, _ := r.Store.GetInstanceStorageSetting(ctx)
  
  s3StorageType := storepb.AttachmentStorageType_S3
  const batchSize = 100
  offset := 0
  
  for {
    // 批量获取 S3 存储的附件
    attachments, _ := r.Store.ListAttachments(ctx, &store.FindAttachment{
      GetBlob:     false,
      StorageType: &s3StorageType,
      Limit:       &batchSize,
      Offset:      &offset,
    })
    
    if len(attachments) == 0 {
      break
    }
    
    for _, attachment := range attachments {
      s3ObjectPayload := attachment.Payload.GetS3Object()
      if s3ObjectPayload == nil {
        continue
      }
      
      // 检查是否需要刷新：上次预签名时间 + 4 天 > 现在
      // 预留 1 天缓冲期
      if s3ObjectPayload.LastPresignedTime != nil {
        if time.Now().Before(s3ObjectPayload.LastPresignedTime.AsTime().Add(4 * 24 * time.Hour)) {
          continue  // 还不需要刷新
        }
      }
      
      // 获取 S3 配置（优先使用 Payload 中的快照，否则使用当前实例配置）
      s3Config := instanceStorageSetting.GetS3Config()
      if s3ObjectPayload.S3Config != nil {
        s3Config = s3ObjectPayload.S3Config
      }
      
      // 重新生成预签名 URL
      s3Client, _ := s3.NewClient(ctx, s3Config)
      presignURL, _ := s3Client.PresignGetObject(ctx, s3ObjectPayload.Key)
      
      // 更新数据库
      r.Store.UpdateAttachment(ctx, &store.UpdateAttachment{
        ID:        attachment.ID,
        Reference: &presignURL,
        Payload: &storepb.AttachmentPayload{
          Payload: &storepb.AttachmentPayload_S3Object_{
            S3Object: &storepb.AttachmentPayload_S3Object{
              S3Config:          s3Config,
              Key:               s3ObjectPayload.Key,
              LastPresignedTime: timestamppb.New(time.Now()),
            },
          },
        },
      })
    }
    
    offset += len(attachments)
  }
}
```

### 6.4 刷新策略

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 预签名 URL 有效期 | 5 天 | `s3.go:66` 设置 |
| Runner 运行间隔 | 12 小时 | `runner.go:26` |
| 刷新阈值 | 4 天 | 预留 1 天缓冲，防止链接过期 |

---

## 7. 删除处理机制

### 7.1 附件删除流程

`store/attachment.go:132-149`：

```go
func (s *Store) DeleteAttachment(ctx context.Context, delete *DeleteAttachment) error {
  // 1. 获取附件信息
  attachment, _ := s.GetAttachment(ctx, &FindAttachment{ID: &delete.ID})
  
  // 2. 删除存储内容（本地文件或 S3 对象）
  if err := s.DeleteAttachmentStorage(ctx, attachment); err != nil {
    if attachment.StorageType == storepb.AttachmentStorageType_LOCAL {
      return errors.Wrap(err, "failed to delete local file")
    }
    slog.Warn("Failed to delete attachment storage", slog.Any("err", err))
  }
  
  // 3. 删除数据库记录
  return s.driver.DeleteAttachment(ctx, delete)
}
```

### 7.2 存储内容删除

`store/attachment.go:202-260`：

```go
func (s *Store) deleteAttachmentStorageImpl(ctx context.Context, attachment *Attachment,
                                              instanceStorageSetting *storepb.InstanceStorageSetting) error {
  if attachment.StorageType == storepb.AttachmentStorageType_LOCAL {
    // 删除本地文件
    p := filepath.FromSlash(attachment.Reference)
    if !filepath.IsAbs(p) {
      p = filepath.Join(s.profile.Data, p)
    }
    os.Remove(p)
    
  } else if attachment.StorageType == storepb.AttachmentStorageType_S3 {
    // 删除 S3 对象
    s3ObjectPayload := attachment.Payload.GetS3Object()
    s3Config := s3ObjectPayload.S3Config
    
    // 如果 Payload 中没有 S3 配置，使用当前实例配置作为回退
    if s3Config == nil {
      if instanceStorageSetting == nil {
        instanceStorageSetting, _ = s.GetInstanceStorageSetting(ctx)
      }
      s3Config = instanceStorageSetting.S3Config
    }
    
    s3Client, _ := s3.NewClient(ctx, s3Config)
    s3Client.DeleteObject(ctx, s3ObjectPayload.Key)
  }
  
  // 清理派生缓存（缩略图、动态视频）
  s.deleteAttachmentDerivedCaches(attachment)
  return nil
}
```

---

## 8. 架构总结

### 8.1 核心设计原则

1. **配置与数据分离**：
   - 实例级 `InstanceStorageSetting` 控制**新上传**的存储位置
   - 附件级 `StorageType` + `Reference` 记录**已上传**资源的实际位置

2. **向后兼容**：
   - 存储后端切换不影响已有附件
   - S3 配置快照保存在 `AttachmentPayload.S3Object.S3Config`，即使实例配置变更，旧附件仍可访问

3. **统一访问入口**：
   - 所有附件通过 `/file/attachments/:uid/:filename` 路由访问
   - 服务器内部根据 `StorageType` 透明处理不同存储后端

### 8.2 存储类型对比

| 特性 | LOCAL | S3 | DATABASE | EXTERNAL |
|------|-------|-----|----------|----------|
| 内容存储位置 | 本地文件系统 | S3 兼容存储 | 数据库 BLOB | 外部服务器 |
| Reference 字段 | 文件相对路径 | 预签名 URL | 空/未使用 | 完整 URL |
| 预签名刷新 | 不需要 | 每 12 小时检查 | 不需要 | 不需要 |
| 删除时清理 | 删除文件 | 删除 S3 对象 | 数据库清理 | 不处理 |
| 适合场景 | 单机部署 | 分布式/云部署 | 小型附件 | 第三方资源 |

### 8.3 关键代码位置

| 功能 | 文件位置 |
|------|----------|
| 上传处理 | `server/router/api/v1/attachment_service.go:68-199` |
| 存储路由核心 | `server/router/api/v1/attachment_service.go:438-522` |
| 文件服务 | `server/router/fileserver/fileserver.go` |
| S3 客户端 | `internal/storage/s3/s3.go` |
| 预签名刷新 | `server/runner/s3presign/runner.go` |
| 实例存储设置 | `store/instance_setting.go:247-273` |
| 附件数据模型 | `store/attachment.go` |
| 协议定义 | `proto/store/attachment.proto`, `proto/store/instance_setting.proto` |

---

## 9. 附录：文件路径解析

### 9.1 本地存储路径解析

`server/router/fileserver/fileserver.go:326-333`：

```go
func (s *FileServerService) resolveLocalPath(reference string) (string, error) {
  filePath := filepath.FromSlash(reference)
  if !filepath.IsAbs(filePath) {
    filePath = filepath.Join(s.Profile.Data, filePath)  // 相对于数据目录
  }
  return filePath, nil
}
```

### 9.2 数据目录来源

`Profile.Data` 来自命令行参数 `--data` 或环境变量，默认值取决于启动配置。

---

*报告生成时间：2026-05-05*
*基于代码版本：memos main 分支*
