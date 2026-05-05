# Memos 资源存储机制分析报告

## 1. 概述

Memos 支持四种资源存储方式的组合配置：
- **实例级存储配置**（控制新上传行为）：DATABASE、LOCAL、S3
- **附件级存储类型**（记录已上传资源位置）：LOCAL、S3、EXTERNAL、DATABASE（默认/回退）

本报告深入分析资源上传路径、存储后端切换机制、下载 URL 生成，特别补充了 **EXTERNAL 外部存储** 的完整链路分析，并修正了之前关于 DATABASE 存储类型的错误结论。

---

## 2. 存储类型定义

### 2.1 两套存储类型枚举

Memos 中有两套独立的存储类型枚举，分别用于不同层面：

#### 实例级存储类型（InstanceStorageSetting.StorageType）

定义在 `proto/store/instance_setting.proto`，用于**控制新上传的存储位置**：

```protobuf
message InstanceStorageSetting {
  enum StorageType {
    STORAGE_TYPE_UNSPECIFIED = 0;
    DATABASE = 1;  // 内容存储在数据库 BLOB 字段
    LOCAL = 2;     // 内容存储在本地文件系统
    S3 = 3;        // 内容存储在 S3 兼容对象存储
  }
  StorageType storage_type = 1;
  string filepath_template = 2;       // 文件路径模板
  int64 upload_size_limit_mb = 3;     // 上传大小限制（MB）
  StorageS3Config s3_config = 4;      // S3 配置
}
```

**重要说明**：
- `EXTERNAL` 不是实例级配置选项，无法在实例设置中选择"外部存储"
- 默认值：`defaultInstanceStorageType = LOCAL`（`store/instance_setting.go:241`）

#### 附件级存储类型（AttachmentStorageType）

定义在 `proto/store/attachment.proto`，用于**记录已上传资源的实际存储位置**：

```protobuf
enum AttachmentStorageType {
  ATTACHMENT_STORAGE_TYPE_UNSPECIFIED = 0;
  LOCAL = 1;       // 本地文件系统
  S3 = 2;          // S3 兼容对象存储
  EXTERNAL = 3;    // 外部链接（URL 形式）—— 仅历史迁移数据
}
```

**关键差异**：
- `EXTERNAL` 只存在于附件级枚举，不存在于实例级配置
- 当前 API 不支持创建 `EXTERNAL` 类型的附件

### 2.2 数据模型

Attachment 结构（`store/attachment.go:16-41`）：

```go
type Attachment struct {
  ID          int32
  UID         string
  CreatorID   int32
  CreatedTs   int64
  UpdatedTs   int64
  Filename    string
  Blob        []byte                       // DATABASE 模式：存储实际内容
  Type        string                       // MIME 类型
  Size        int64
  StorageType storepb.AttachmentStorageType // 实际存储类型
  Reference   string                       // 存储引用
  Payload     *storepb.AttachmentPayload   // 附加载荷（仅 S3 使用）
  MemoID      *int32
  MemoUID     *string
}
```

Reference 字段的含义根据 StorageType 不同而变化：

| StorageType | Reference 字段 | Blob 字段 | Payload.S3Object |
|-------------|----------------|-----------|------------------|
| LOCAL | 相对路径（如 `assets/123_file.jpg`） | nil | 无 |
| S3 | 预签名 URL（有效期 5 天） | nil | 有（Key、S3Config、LastPresignedTime） |
| EXTERNAL | 完整外部 URL | nil | 无 |
| UNSPECIFIED/DATABASE | 空或未使用 | 实际文件内容 | 无 |

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
SaveAttachmentBlob()  [核心存储路由函数]
    ↓
┌─────────────────────────────────────────────────────────────────┐
│              根据 InstanceStorageSetting.StorageType 路由        │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   LOCAL (2)     │    S3 (3)       │  DATABASE (1) / UNSPECIFIED │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ os.WriteFile    │ s3.UploadObject │ 不做任何处理                │
│ Reference = 路径 │ Reference = 预  │ Blob 保持不变              │
│ StorageType=LOCAL │ 签名URL        │ StorageType 保持默认值      │
│ Blob = nil      │ StorageType=S3  │                             │
│                 │ Blob = nil      │                             │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

### 3.2 核心存储路由函数：SaveAttachmentBlob

`server/router/api/v1/attachment_service.go:438-522`：

```go
func SaveAttachmentBlob(ctx context.Context, profile *profile.Profile, 
                         stores *store.Store, create *store.Attachment) error {
  instanceStorageSetting, err := stores.GetInstanceStorageSetting(ctx)
  
  if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_LOCAL {
    // ===== LOCAL 存储处理 =====
    filepathTemplate := instanceStorageSetting.FilepathTemplate
    if filepathTemplate == "" {
      filepathTemplate = "assets/{timestamp}_{uuid}_{filename}"
    }
    
    // 路径模板替换（支持 {timestamp}, {year}, {month}, {day}, {hour}, {minute}, {second}, {uuid}, {filename}）
    internalPath = replaceFilenameWithPathTemplate(filepathTemplate, create.Filename)
    
    // 确保路径唯一（如果冲突，追加 UID）
    osPath = ensureUniqueLocalAttachmentPath(osPath, create.UID)
    
    // 创建目录并写入文件
    dir := filepath.Dir(osPath)
    os.MkdirAll(dir, os.ModePerm)
    os.WriteFile(osPath, create.Blob, 0644)
    
    // 设置附件元数据
    create.Reference = internalPath
    create.Blob = nil
    create.StorageType = storepb.AttachmentStorageType_LOCAL
    
  } else if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_S3 {
    // ===== S3 存储处理 =====
    s3Config := instanceStorageSetting.S3Config
    if s3Config == nil {
      return errors.Errorf("No activated external storage found")
    }
    
    s3Client, err := s3.NewClient(ctx, s3Config)
    
    // 路径模板处理
    filepathTemplate := instanceStorageSetting.FilepathTemplate
    if !strings.Contains(filepathTemplate, "{filename}") {
      filepathTemplate = filepath.Join(filepathTemplate, "{filename}")
    }
    filepathTemplate = replaceFilenameWithPathTemplate(filepathTemplate, create.Filename)
    
    // 上传到 S3
    key, err := s3Client.UploadObject(ctx, filepathTemplate, create.Type, 
                                     bytes.NewReader(create.Blob))
    
    // 生成预签名 URL（有效期 5 天）
    presignURL, err := s3Client.PresignGetObject(ctx, key)
    
    // 设置附件元数据
    create.Reference = presignURL
    create.Blob = nil
    create.StorageType = storepb.AttachmentStorageType_S3
    
    // 保存 S3 对象信息到 Payload（用于后续刷新和访问）
    payload := ensureAttachmentPayload(create.Payload)
    payload.Payload = &storepb.AttachmentPayload_S3Object_{
      S3Object: &storepb.AttachmentPayload_S3Object{
        S3Config:          s3Config,  // S3 配置快照
        Key:               key,
        LastPresignedTime: timestamppb.New(time.Now()),
      },
    }
    create.Payload = payload
  }
  
  // ===== DATABASE / UNSPECIFIED：不做任何处理 =====
  // Blob 字段保持不变，Reference 为空，StorageType 保持默认值（UNSPECIFIED）
  return nil
}
```

### 3.3 关键结论修正

**之前的错误**：认为 DATABASE 是"默认回退"模式。

**正确理解**：
1. `InstanceStorageSetting.DATABASE` 是一个**主动选择**的配置选项
2. 当 `StorageType` 既不是 LOCAL 也不是 S3 时，`SaveAttachmentBlob` **不做任何处理**
3. 此时 `create.Blob` 保持不变，内容最终存储在数据库的 `blob` 列
4. `create.StorageType` 保持默认值 `ATTACHMENT_STORAGE_TYPE_UNSPECIFIED`

---

## 4. EXTERNAL 外部存储深度分析

### 4.1 EXTERNAL 类型的来源

**重要发现**：当前版本的 `CreateAttachment` API **不支持**创建 EXTERNAL 类型的附件。

EXTERNAL 类型的附件只能来自：

#### 历史数据迁移

从迁移脚本 `store/migration/sqlite/0.22/00__resource_storage_type.sql` 可以看到：

```sql
-- 版本 0.22 之前的 resource 表有 external_link 列
ALTER TABLE resource ADD COLUMN storage_type TEXT NOT NULL DEFAULT '';
ALTER TABLE resource ADD COLUMN reference TEXT NOT NULL DEFAULT '';

-- 将旧数据迁移到新结构
UPDATE resource
SET storage_type = 'LOCAL', reference = internal_path
WHERE internal_path IS NOT NULL AND internal_path != '';

UPDATE resource
SET storage_type = 'EXTERNAL', reference = external_link
WHERE external_link IS NOT NULL AND external_link != '';

-- 删除旧列
ALTER TABLE resource DROP COLUMN internal_path;
ALTER TABLE resource DROP COLUMN external_link;
```

这说明：
- 旧版本 memos 有单独的 `external_link` 字段
- 用户可能通过某种方式添加外部链接作为附件
- 迁移时，这些数据被转换为 `StorageType = EXTERNAL`，`Reference = external_link`

#### 当前 API 限制

检查 `CreateAttachment` 的实现（`server/router/api/v1/attachment_service.go:68-199`）：

- API 定义中有 `optional string external_link = 5` 字段（`proto/api/v1/attachment_service.proto:97-98`）
- 但后端实现**完全没有读取或处理**这个字段
- `SaveAttachmentBlob` 只处理 LOCAL 和 S3，其他情况走 DATABASE 模式

**结论**：当前版本无法通过标准 API 创建 EXTERNAL 类型的附件。

### 4.2 EXTERNAL 类型的下载链路

#### 前端处理逻辑

`web/src/hooks/useAttachmentLibrary.ts:84`：

```typescript
sourceUrl: attachment.externalLink || `${window.location.origin}/file/${attachment.name}/${attachment.filename}`,
```

这是关键的分流逻辑：

```
┌─────────────────────────────────────────────────────────────┐
│                    前端获取附件 URL                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   attachment.externalLink 是否存在且非空？                   │
│              │                                              │
│         ┌────┴────┐                                         │
│         │         │                                         │
│        是        否                                         │
│         │         │                                         │
│         ▼         ▼                                         │
│   ┌─────────┐  ┌─────────────────────────────┐            │
│   │ 直接使用 │  │ 使用本地文件服务路由         │            │
│   │ external │  │ /file/attachments/{uid}/{fn}│            │
│   │ Link    │  │                              │            │
│   └────┬────┘  └──────────────┬──────────────┘            │
│        │                       │                            │
│        ▼                       ▼                            │
│   ┌─────────────────┐  ┌───────────────────────┐          │
│   │ 不经过服务器    │  │ 经过 fileserver       │          │
│   │ 直接访问外部 URL│  │ 有权限校验           │          │
│   │ 无下载时权限控制│  │                      │          │
│   └─────────────────┘  └───────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

#### externalLink 字段的设置

`server/router/api/v1/attachment_service.go:430-432`：

```go
// S3 和 EXTERNAL 存储类型都会设置 ExternalLink
if attachment.StorageType == storepb.AttachmentStorageType_EXTERNAL || 
   attachment.StorageType == storepb.AttachmentStorageType_S3 {
  attachmentMessage.ExternalLink = attachment.Reference
}
```

**重要**：S3 类型的附件也会设置 `externalLink`，其值为预签名 URL。

### 4.3 EXTERNAL 与 S3 的下载差异

| 特性 | EXTERNAL | S3 |
|------|----------|-----|
| 前端 URL 来源 | `attachment.externalLink` | `attachment.externalLink` |
| Reference 内容 | 外部服务的完整 URL | S3 预签名 URL |
| 预签名有效期 | 无（外部 URL 自行管理） | 5 天，需要刷新 |
| 服务器参与下载 | 完全不参与 | 媒体文件：307 重定向<br>非媒体：代理流式返回 |
| 下载时权限校验 | 无（直接访问外部） | 元数据获取时有，下载时无（S3 直连） |

### 4.4 EXTERNAL 类型的边界条件

#### 边界条件 1：文件服务器不处理 EXTERNAL

检查 `server/router/fileserver/fileserver.go` 中的所有 `switch attachment.StorageType`：

```go
// serveMediaStream 中
switch attachment.StorageType {
case storepb.AttachmentStorageType_LOCAL:
  // ... 本地文件处理
case storepb.AttachmentStorageType_S3:
  // ... S3 处理
default:
  // DATABASE 模式：使用 attachment.Blob
}

// serveStaticFile 中
switch attachment.StorageType {
case storepb.AttachmentStorageType_LOCAL:
  // ...
case storepb.AttachmentStorageType_S3:
  // ...
default:
  // DATABASE 模式
}
```

**EXTERNAL 类型没有被显式处理**，会走到 `default` 分支（DATABASE 模式）。

但实际上，因为前端优先使用 `externalLink`，所以：
- **正常情况**：EXTERNAL 类型的附件不会请求 `/file/...` 路由
- **异常情况**：如果有人手动构造 `/file/attachments/{uid}/{filename}` 请求 EXTERNAL 类型的附件：
  - 会走到 `default` 分支
  - 尝试使用 `attachment.Blob`（但 EXTERNAL 类型的 Blob 应该是 nil）
  - 可能导致空指针或错误

#### 边界条件 2：删除 EXTERNAL 附件

检查 `store/attachment.go:202-260` 的 `deleteAttachmentStorageImpl`：

```go
func (s *Store) deleteAttachmentStorageImpl(ctx context.Context, attachment *Attachment, ...) error {
  if attachment.StorageType == storepb.AttachmentStorageType_LOCAL {
    // 删除本地文件
  } else if attachment.StorageType == storepb.AttachmentStorageType_S3 {
    // 删除 S3 对象
  }
  
  // 清理派生缓存（缩略图、动态视频）
  s.deleteAttachmentDerivedCaches(attachment)
  return nil
}
```

**EXTERNAL 类型的删除行为**：
- 不会删除任何外部资源（服务器无法控制外部服务器）
- 只会清理本地派生缓存（如果有的话）
- 数据库记录会被删除

#### 边界条件 3：RSS 订阅中的处理

`server/router/rss/rss.go:280-284`：

```go
if attachment.StorageType == storepb.AttachmentStorageType_EXTERNAL || 
   attachment.StorageType == storepb.AttachmentStorageType_S3 {
  enclosure.Url = attachment.Reference  // 直接使用外部/预签名 URL
} else {
  enclosure.Url = fmt.Sprintf("%s/file/attachments/%s/%s", baseURL, attachment.UID, attachment.Filename)
}
```

RSS 订阅中，EXTERNAL 和 S3 类型直接使用 `Reference` 作为 enclosure URL，不经过本地文件服务。

---

## 5. 存储后端切换机制

### 5.1 配置存储与读取

实例存储设置通过 `InstanceSetting` 表管理，Key 为 `STORAGE`。

#### 配置读取流程（`store/instance_setting.go:247-273`）：

```
GetInstanceStorageSetting()
    │
    ▼
┌──────────────────────┐
│ 检查 instanceSetting │
│ Cache 中是否有缓存   │
└──────────┬───────────┘
           │
      ┌────┴────┐
      │         │
     有        无
      │         │
      ▼         ▼
  ┌────────┐  ┌─────────────────────────┐
  │ 返回缓存│  │ 从数据库读取           │
  │        │  │ InstanceSetting 表      │
  └────────┘  └──────────┬──────────────┘
                          │
                          ▼
                   ┌──────────────────┐
                   │ 应用默认值：      │
                   │ - StorageType=LOCAL│
                   │ - UploadLimit=30MB│
                   │ - PathTemplate=   │
                   │   assets/...      │
                   └─────────┬─────────┘
                             │
                             ▼
                      ┌────────────┐
                      │ 更新缓存   │
                      └────────────┘
```

### 5.2 切换机制的核心特性

#### 特性 1：运行时动态读取

每次上传时都会调用 `GetInstanceStorageSetting()` 获取当前配置：

```go
// SaveAttachmentBlob 中
instanceStorageSetting, err := stores.GetInstanceStorageSetting(ctx)
```

这意味着：
- 管理员更改存储配置后，**立即生效**
- 不需要重启服务器

#### 特性 2：向后兼容（关键！）

**已上传的附件不受后端切换影响**。

原因：
1. 每个附件自身记录 `StorageType` 和 `Reference`
2. 下载时根据**附件的** `StorageType` 处理，不是根据**当前实例**配置

示例场景：
```
时间线：
T0: 配置 = LOCAL
    上传附件 A → StorageType=LOCAL, Reference=assets/a.jpg

T1: 管理员切换配置到 S3
    上传附件 B → StorageType=S3, Reference=https://s3.../b.jpg

T2: 访问附件 A
    根据 A.StorageType=LOCAL → 从本地文件系统读取
    （不受当前 S3 配置影响）

T3: 访问附件 B
    根据 B.StorageType=S3 → 从 S3 读取
```

#### 特性 3：S3 配置快照

S3 类型的附件在 `Payload.S3Object.S3Config` 中保存了**创建时的 S3 配置快照**。

这意味着：
- 即使实例的 S3 配置 later 变更（如切换到另一个 bucket）
- 旧附件仍然使用**创建时的配置**访问 S3
- 避免了"配置变更导致旧附件无法访问"的问题

`server/runner/s3presign/runner.go:91-94`：

```go
// 获取 S3 配置时的优先级
s3Config := instanceStorageSetting.GetS3Config()  // 当前实例配置（备选）
if s3ObjectPayload.S3Config != nil {
  s3Config = s3ObjectPayload.S3Config  // 优先使用快照配置
}
```

### 5.3 切换存储类型的影响

| 操作 | 已上传附件 | 新上传附件 |
|------|-----------|-----------|
| LOCAL → S3 | 不受影响，仍从本地读取 | 上传到 S3 |
| S3 → LOCAL | 不受影响，仍从 S3 读取（需要配置快照） | 上传到本地 |
| 任何 → DATABASE | 不受影响 | 内容存储在数据库 |

---

## 6. 下载 URL 生成与权限校验

### 6.1 四种存储类型的下载链路对比

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           附件下载流程总览                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  前端需要访问附件                                                         │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 有 externalLink 吗？（S3 或 EXTERNAL 类型）                      │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                              │                                           │
│                    ┌─────────┴─────────┐                                │
│                    │                   │                                │
│                   是                  否                                │
│                    │                   │                                │
│                    ▼                   ▼                                │
│           ┌──────────────┐    ┌─────────────────────────┐            │
│           │ 直接使用      │    │ 请求 /file/attachments/ │            │
│           │ externalLink │    │ {uid}/{filename} 路由   │            │
│           └──────┬───────┘    └───────────┬─────────────┘            │
│                  │                         │                            │
│                  ▼                         ▼                            │
│           ┌──────────────────┐    ┌────────────────────────┐          │
│           │ S3 预签名 URL 或 │    │ 经过 fileserver 处理  │          │
│           │ 外部服务 URL     │    │                        │          │
│           │                  │    │ - 权限校验            │          │
│           │ 无服务器参与下载 │    │ - 根据 StorageType 路由│          │
│           └──────────────────┘    └────────────────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 文件服务路由（LOCAL / DATABASE 类型）

#### 路由注册

`server/router/fileserver/fileserver.go:120-125`：

```go
func (s *FileServerService) RegisterRoutes(echoServer *echo.Echo) {
  fileGroup := echoServer.Group("/file")
  fileGroup.GET("/attachments/:uid/:filename", s.serveAttachmentFile)
  fileGroup.GET("/users/:identifier/avatar", s.serveUserAvatar)
}
```

#### 完整处理流程

`server/router/fileserver/fileserver.go:132-165`：

```go
func (s *FileServerService) serveAttachmentFile(c *echo.Context) error {
  uid := c.Param("uid")
  wantThumbnail := c.QueryParam("thumbnail") == "true"
  wantMotion := c.QueryParam("motion") == "true"
  
  // ===== 步骤 1：读取附件元数据 =====
  attachment, err := s.Store.GetAttachment(ctx, &store.FindAttachment{
    UID:     &uid,
    GetBlob: true,  // DATABASE 模式需要读取 Blob
  })
  
  // ===== 步骤 2：权限校验 =====
  if err := s.checkAttachmentPermission(ctx, c, attachment); err != nil {
    return err
  }
  
  // ===== 步骤 3：内容类型处理（XSS 防护） =====
  contentType := s.sanitizeContentType(attachment.Type)
  
  // ===== 步骤 4：根据媒体类型分流 =====
  if isMediaType(attachment.Type) {  // video/* 或 audio/*
    return s.serveMediaStream(c, attachment, contentType)
  }
  
  return s.serveStaticFile(c, attachment, contentType, wantThumbnail)
}
```

### 6.3 权限校验机制

`server/router/fileserver/fileserver.go:636-688`：

```go
func (s *FileServerService) checkAttachmentPermission(ctx context.Context, 
                                                         c *echo.Context, 
                                                         attachment *store.Attachment) error {
  // ===== 场景 1：未关联任何备忘录的附件 =====
  if attachment.MemoID == nil {
    // 只有创建者或管理员可以访问
    user, _ := s.getCurrentUser(ctx, c)
    if user == nil {
      return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
    }
    if user.ID != attachment.CreatorID && user.Role != store.RoleAdmin {
      return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
    }
    return nil
  }
  
  // ===== 场景 2：已关联备忘录的附件 =====
  memo, _ := s.Store.GetMemo(ctx, &store.FindMemo{ID: attachment.MemoID})
  
  // 子场景 2a：公开备忘录 → 任何人可访问
  if memo.Visibility == store.Public {
    return nil
  }
  
  // 子场景 2b：有分享令牌 → 允许访问
  if shareToken := c.QueryParam("share_token"); shareToken != "" {
    ms, _ := s.Store.GetMemoShare(ctx, &store.FindMemoShare{UID: &shareToken})
    if ms != nil && !isMemoShareExpired(ms) && ms.MemoID == memo.ID {
      return nil
    }
  }
  
  // 子场景 2c：需要认证
  user, _ := s.getCurrentUser(ctx, c)
  if user == nil {
    return echo.NewHTTPError(http.StatusUnauthorized, "unauthorized access")
  }
  
  // 子场景 2d：私有备忘录 → 只有创建者或管理员可访问
  if memo.Visibility == store.Private && 
     user.ID != memo.CreatorID && 
     user.Role != store.RoleAdmin {
    return echo.NewHTTPError(http.StatusForbidden, "forbidden access")
  }
  
  return nil
}
```

### 6.4 不同存储类型的下载处理

#### LOCAL 类型

```go
// 媒体文件和静态文件的处理相同
filePath, _ := s.resolveLocalPath(attachment.Reference)
// resolveLocalPath 将相对路径转为绝对路径：
//   如果 Reference 是相对路径 → filepath.Join(Profile.Data, Reference)
//   如果是绝对路径 → 直接使用

http.ServeFile(c.Response(), c.Request(), filePath)
// http.ServeFile 特性：
// - 支持 Range 请求（视频分段加载）
// - 自动处理 Last-Modified / ETag
// - 支持 304 Not Modified
```

#### S3 类型

**媒体文件（video/audio）**：

```go
// serveMediaStream 中
case storepb.AttachmentStorageType_S3:
  // 直接重定向到 S3 预签名 URL
  presignURL, _ := s.getS3PresignedURL(c.Request().Context(), attachment)
  return c.Redirect(http.StatusTemporaryRedirect, presignURL)
```

**非媒体文件**：

```go
// serveStaticFile 中
case storepb.AttachmentStorageType_S3:
  // 通过服务器代理流式返回
  reader, _ := s.getAttachmentReader(c.Request().Context(), attachment)
  defer reader.Close()
  return c.Stream(http.StatusOK, contentType, reader)
```

**为什么差异处理？**
- 媒体文件通常较大，且播放器需要 Range 请求支持
- 重定向到 S3 可以利用 S3 的 Range 请求能力
- 非媒体文件通过服务器代理，可以保持统一的访问控制（虽然下载时实际没有权限校验）

#### DATABASE 类型

```go
// 媒体文件
modTime := time.Unix(attachment.UpdatedTs, 0)
http.ServeContent(c.Response(), c.Request(), attachment.Filename, 
                  modTime, bytes.NewReader(attachment.Blob))

// 非媒体文件
return c.Blob(http.StatusOK, contentType, attachment.Blob)
```

DATABASE 模式下，文件内容存储在 `attachment.Blob` 字段中，直接从内存返回。

### 6.5 权限校验的边界

| 存储类型 | 元数据获取时权限 | 实际下载时权限 | 说明 |
|---------|-----------------|---------------|------|
| LOCAL | 有 | 有 | 下载请求经过 fileserver，有 `checkAttachmentPermission` |
| DATABASE | 有 | 有 | 同上 |
| S3 | 有 | 无 | 元数据获取时有；下载时要么重定向到 S3，要么代理但代理前已校验 |
| EXTERNAL | 有 | 无 | 元数据获取时有；下载时直接访问外部 URL，服务器完全不参与 |

**重要说明**：
- S3 的"下载时无权限"是相对的：
  - 媒体文件：307 重定向到预签名 URL（预签名本身就是一种授权）
  - 非媒体文件：代理流式返回，但代理之前已经过 `checkAttachmentPermission`
- EXTERNAL 的"下载时无权限"是真正的无控制：
  - 只要用户拿到 `externalLink`，就可以直接访问
  - 服务器无法撤销或限制外部 URL 的访问

---

## 7. S3 预签名 URL 自动刷新机制

### 7.1 问题背景

S3 预签名 URL 有有效期限制：
- `internal/storage/s3/s3.go:66` 设置有效期为 **5 天**
- 如果不刷新，超过 5 天后 `Reference` 中的 URL 会失效

### 7.2 后台 Runner

`server/runner/s3presign/runner.go`：

```go
type Runner struct {
  Store *store.Store
}

const runnerInterval = time.Hour * 12  // 每 12 小时运行一次

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

### 7.3 刷新逻辑

```go
func (r *Runner) CheckAndPresign(ctx context.Context) {
  instanceStorageSetting, _ := r.Store.GetInstanceStorageSetting(ctx)
  
  s3StorageType := storepb.AttachmentStorageType_S3
  const batchSize = 100
  offset := 0
  
  for {
    // ===== 批量获取 S3 存储的附件 =====
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
      
      // ===== 判断是否需要刷新 =====
      // 策略：上次预签名时间 + 4 天 > 现在 ？
      // 预留 1 天缓冲期，防止链接过期
      if s3ObjectPayload.LastPresignedTime != nil {
        if time.Now().Before(s3ObjectPayload.LastPresignedTime.AsTime().Add(4 * 24 * time.Hour)) {
          continue  // 还不需要刷新
        }
      }
      
      // ===== 获取 S3 配置 =====
      // 优先级：Payload 中的快照 > 当前实例配置
      s3Config := instanceStorageSetting.GetS3Config()
      if s3ObjectPayload.S3Config != nil {
        s3Config = s3ObjectPayload.S3Config
      }
      
      // ===== 重新生成预签名 URL =====
      s3Client, _ := s3.NewClient(ctx, s3Config)
      presignURL, _ := s3Client.PresignGetObject(ctx, s3Object.Key)
      
      // ===== 更新数据库 =====
      r.Store.UpdateAttachment(ctx, &store.UpdateAttachment{
        ID:        attachment.ID,
        Reference: &presignURL,  // 更新预签名 URL
        Payload: &storepb.AttachmentPayload{
          Payload: &storepb.AttachmentPayload_S3Object_{
            S3Object: &storepb.AttachmentPayload_S3Object{
              S3Config:          s3Config,           // 更新配置快照
              Key:               s3ObjectPayload.Key,
              LastPresignedTime: timestamppb.New(time.Now()),  // 更新时间
            },
          },
        },
      })
    }
    
    offset += len(attachments)
  }
}
```

### 7.4 刷新策略参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 预签名 URL 有效期 | 5 天 | `s3.go:66` 中 `PresignOptions.Expires` |
| Runner 运行间隔 | 12 小时 | `runner.go:26` |
| 刷新阈值 | 4 天 | 预留 1 天缓冲，`LastPresignedTime + 4天 < 现在` 时刷新 |
| 批量大小 | 100 | 每批处理 100 个附件 |

### 7.5 潜在问题

1. **服务器重启期间**：
   - 如果服务器关闭超过 1 天，可能有部分 URL 过期
   - 重启后会立即检查（如果 Runner 启动时执行一次）

2. **实例 S3 配置变更**：
   - 如果旧附件的 `Payload.S3Object.S3Config` 为 nil（历史数据可能没有）
   - 会使用当前实例配置刷新
   - 如果当前配置指向不同的 bucket，可能导致刷新失败

---

## 8. 删除处理机制

### 8.1 附件删除流程

`store/attachment.go:132-149`：

```go
func (s *Store) DeleteAttachment(ctx context.Context, delete *DeleteAttachment) error {
  // ===== 步骤 1：获取附件信息 =====
  attachment, _ := s.GetAttachment(ctx, &FindAttachment{ID: &delete.ID})
  
  // ===== 步骤 2：删除存储内容 =====
  if err := s.DeleteAttachmentStorage(ctx, attachment); err != nil {
    if attachment.StorageType == storepb.AttachmentStorageType_LOCAL {
      return errors.Wrap(err, "failed to delete local file")
    }
    // S3 删除失败只记录警告，不阻塞数据库删除
    slog.Warn("Failed to delete attachment storage", slog.Any("err", err))
  }
  
  // ===== 步骤 3：删除数据库记录 =====
  return s.driver.DeleteAttachment(ctx, delete)
}
```

### 8.2 不同存储类型的删除行为

`store/attachment.go:202-260`：

```go
func (s *Store) deleteAttachmentStorageImpl(ctx context.Context, 
                                              attachment *Attachment,
                                              instanceStorageSetting *storepb.InstanceStorageSetting) error {
  // ===== LOCAL：删除本地文件 =====
  if attachment.StorageType == storepb.AttachmentStorageType_LOCAL {
    p := filepath.FromSlash(attachment.Reference)
    if !filepath.IsAbs(p) {
      p = filepath.Join(s.profile.Data, p)
    }
    err := os.Remove(p)
    if err != nil && !os.IsNotExist(err) {
      return errors.Wrap(err, "failed to delete local file")
    }
  }
  
  // ===== S3：删除 S3 对象 =====
  else if attachment.StorageType == storepb.AttachmentStorageType_S3 {
    s3ObjectPayload := attachment.Payload.GetS3Object()
    if s3ObjectPayload == nil {
      return errors.Errorf("No s3 object found")
    }
    
    // 获取 S3 配置
    s3Config := s3ObjectPayload.S3Config
    if s3Config == nil {
      // 如果 Payload 中没有，使用当前实例配置作为回退
      if instanceStorageSetting == nil {
        instanceStorageSetting, _ = s.GetInstanceStorageSetting(ctx)
      }
      s3Config = instanceStorageSetting.S3Config
    }
    
    s3Client, _ := s3.NewClient(ctx, s3Config)
    if err := s3Client.DeleteObject(ctx, s3ObjectPayload.Key); err != nil {
      return errors.Wrap(err, "Failed to delete s3 object")
    }
  }
  
  // ===== EXTERNAL / DATABASE：不删除任何外部资源 =====
  // EXTERNAL：服务器无法控制外部服务器的资源
  // DATABASE：内容随数据库记录一起删除
  
  // ===== 清理派生缓存（缩略图、动态视频）=====
  s.deleteAttachmentDerivedCaches(attachment)
  
  return nil
}
```

### 8.3 派生缓存清理

`store/attachment.go:284-293`：

```go
func (s *Store) deleteAttachmentDerivedCaches(attachment *Attachment) {
  for _, cachePath := range []string{
    // 缩略图缓存
    filepath.Join(s.profile.Data, thumbnailCacheFolder, attachment.UID+".jpeg"),
    // 动态照片视频缓存
    filepath.Join(s.profile.Data, motionCacheFolder, attachment.UID+".mp4"),
  } {
    if err := os.Remove(cachePath); err != nil && !os.IsNotExist(err) {
      slog.Warn("Failed to delete derived attachment cache", ...)
    }
  }
}
```

### 8.4 删除行为汇总

| 存储类型 | 删除本地文件 | 删除 S3 对象 | 删除外部资源 | 清理派生缓存 | 删除数据库记录 |
|---------|------------|-------------|-------------|-------------|---------------|
| LOCAL | ✓ | - | - | ✓ | ✓ |
| S3 | - | ✓ | - | ✓ | ✓ |
| EXTERNAL | - | - | ✗（无法控制） | ✓ | ✓ |
| DATABASE | - | - | - | ✓ | ✓（Blob 随记录删除） |

---

## 9. 四种存储类型完整对比

### 9.1 核心特性对比

| 特性 | LOCAL | S3 | DATABASE | EXTERNAL |
|------|-------|-----|----------|----------|
| **存储位置** | 本地文件系统 | S3 兼容存储 | 数据库 BLOB | 外部服务器 |
| **实例级配置** | ✓ | ✓ | ✓ | ✗（无此选项） |
| **创建方式** | 标准上传 API | 标准上传 API | 标准上传 API | 仅历史迁移 |
| **Reference 字段** | 相对路径 | 预签名 URL | 空 | 完整外部 URL |
| **Blob 字段** | nil | nil | 实际内容 | nil |
| **需要 Payload** | - | ✓（S3 配置、Key、时间） | - | - |

### 9.2 上传链路对比

| 特性 | LOCAL | S3 | DATABASE | EXTERNAL |
|------|-------|-----|----------|----------|
| 经过 SaveAttachmentBlob | ✓ | ✓ | ✓（但不处理） | N/A |
| 处理文件内容 | 写入本地文件 | 上传到 S3 | 保持在 Blob 中 | N/A |
| 设置 StorageType | LOCAL | S3 | 保持默认 | N/A |
| 清空 Blob | ✓ | ✓ | ✗（保留） | N/A |

### 9.3 下载链路对比

| 特性 | LOCAL | S3 | DATABASE | EXTERNAL |
|------|-------|-----|----------|----------|
| 前端使用 externalLink | ✗ | ✓ | ✗ | ✓ |
| 经过 /file/... 路由 | ✓ | 部分¹ | ✓ | ✗ |
| 权限校验（元数据） | ✓ | ✓ | ✓ | ✓ |
| 权限校验（下载时） | ✓ | 部分² | ✓ | ✗ |
| 服务器参与下载 | ✓ | 部分³ | ✓ | ✗ |
| 支持 Range 请求 | ✓ | ✓⁴ | ✓⁵ | 取决于外部 |

**注解**：
1. S3 类型的前端优先使用 `externalLink`，不会请求 `/file/...` 路由
2. S3 的下载权限：
   - 元数据获取时有权限校验
   - 下载时：媒体文件重定向到预签名 URL（预签名即授权）；非媒体文件代理返回（代理前已校验）
3. S3 的服务器参与：
   - 媒体文件：仅重定向，不参与数据传输
   - 非媒体文件：代理流式返回
4. S3 的 Range 请求：由 S3 服务端支持
5. DATABASE 的 Range 请求：由 `http.ServeContent` 支持

### 9.4 边界条件与特殊情况

#### 情况 1：存储类型切换

```
场景：LOCAL → S3 → LOCAL

结果：
- 所有已上传的附件保持原有 StorageType
- 只有新上传的附件使用当前配置
- S3 附件依赖 Payload 中的配置快照，不依赖当前实例配置
```

#### 情况 2：EXTERNAL 类型手动访问文件路由

```
场景：用户手动构造 /file/attachments/{uid}/{filename} 访问 EXTERNAL 附件

结果：
- fileserver 中没有 EXTERNAL 的 case，会走到 default（DATABASE 模式）
- 尝试使用 attachment.Blob，但 EXTERNAL 的 Blob 是 nil
- 可能导致错误或空响应

实际情况：
- 前端优先使用 externalLink，所以这种情况几乎不会发生
- 只有知道 API 细节的恶意用户才会尝试
```

#### 情况 3：S3 预签名 URL 过期

```
场景：服务器关闭超过 5 天，期间没有刷新预签名 URL

结果：
- attachment.Reference 中的 URL 已过期
- 但：
  1. 前端使用 externalLink 时，直接访问过期 URL 会失败
  2. 如果请求 /file/... 路由，fileserver 会实时生成新的预签名 URL
  
缓解：
- fileserver.serveMediaStream 和 serveStaticFile 中：
  - 不是直接使用 Reference，而是调用 getS3PresignedURL 实时生成
  - 所以即使 Reference 过期，通过文件路由访问仍然有效
```

**重要发现**：S3 类型的附件通过 `/file/...` 路由访问时，**不依赖** `attachment.Reference` 中的过期 URL，而是实时生成新的预签名 URL。

让我验证这个发现...

查看 `server/router/fileserver/fileserver.go:396-399`：

```go
case storepb.AttachmentStorageType_S3:
  presignURL, err := s.getS3PresignedURL(c.Request().Context(), attachment)
  return c.Redirect(http.StatusTemporaryRedirect, presignURL)
```

和 `getS3PresignedURL`（`fileserver.go:396-407`）：

```go
func (s *FileServerService) getS3PresignedURL(ctx context.Context, attachment *store.Attachment) (string, error) {
  client, s3Object, err := s.createS3Client(attachment)
  url, err := client.PresignGetObject(ctx, s3Object.Key)
  return url, nil
}
```

**确认**：通过文件路由访问 S3 附件时，会**实时生成新的预签名 URL**，不依赖 `attachment.Reference`。

这意味着：
- `attachment.Reference` 中的预签名 URL 主要用于 `externalLink` 字段
- 如果前端使用 `externalLink`，URL 过期会导致访问失败
- 如果前端使用 `/file/...` 路由，总是能获取有效的 URL

但从前端代码看，S3 类型的附件有 `externalLink`，所以优先使用预签名 URL。这是一个潜在问题：
- 如果预签名 URL 过期，且用户从缓存或历史记录中访问
- 前端可能直接使用过期的 `externalLink`
- 解决方案：Runner 每 12 小时刷新一次，确保 URL 不会过期太久

---

## 10. 关键代码位置索引

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 实例存储类型定义 | `proto/store/instance_setting.proto` | 15-27 |
| 附件存储类型定义 | `proto/store/attachment.proto` | 10-18 |
| Attachment 数据模型 | `store/attachment.go` | 16-41 |
| CreateAttachment API | `server/router/api/v1/attachment_service.go` | 68-199 |
| 存储路由核心 SaveAttachmentBlob | `server/router/api/v1/attachment_service.go` | 438-522 |
| 路径模板变量替换 | `server/router/api/v1/attachment_service.go` | 577-606 |
| 文件服务路由注册 | `server/router/fileserver/fileserver.go` | 120-125 |
| 附件下载处理入口 | `server/router/fileserver/fileserver.go` | 132-165 |
| 权限校验 checkAttachmentPermission | `server/router/fileserver/fileserver.go` | 636-688 |
| 媒体文件流 serveMediaStream | `server/router/fileserver/fileserver.go` | 204-230 |
| 静态文件服务 serveStaticFile | `server/router/fileserver/fileserver.go` | 232-273 |
| 实时生成 S3 预签名 URL | `server/router/fileserver/fileserver.go` | 396-407 |
| S3 客户端实现 | `internal/storage/s3/s3.go` | 全部 |
| S3 预签名刷新 Runner | `server/runner/s3presign/runner.go` | 全部 |
| 实例存储设置读取 | `store/instance_setting.go` | 247-273 |
| 附件删除流程 | `store/attachment.go` | 132-149 |
| 存储内容删除实现 | `store/attachment.go` | 202-260 |
| 前端 URL 选择逻辑 | `web/src/hooks/useAttachmentLibrary.ts` | 84 |
| 历史迁移脚本（EXTERNAL 来源） | `store/migration/sqlite/0.22/00__resource_storage_type.sql` | 全部 |

---

## 11. 修正之前报告的错误结论

### 错误 1：DATABASE 是"默认回退"

**修正**：
- `InstanceStorageSetting.DATABASE` 是一个**主动选择**的配置选项
- 当 `StorageType` 既不是 LOCAL 也不是 S3 时，`SaveAttachmentBlob` 不做任何处理
- 此时内容保留在 `Blob` 字段中，最终存储在数据库

### 错误 2：EXTERNAL 是可通过 API 创建的存储类型

**修正**：
- `EXTERNAL` 只存在于 `AttachmentStorageType` 枚举，不存在于 `InstanceStorageSetting`
- 当前 `CreateAttachment` API 不支持创建 EXTERNAL 类型的附件
- EXTERNAL 类型的附件只能来自**历史数据迁移**（版本 0.22 之前的 `external_link` 字段）

### 错误 3：S3 类型的下载依赖 Reference 中的预签名 URL

**部分修正**：
- 前端使用 `externalLink` 时，确实依赖 `Reference` 中的预签名 URL
- 但通过 `/file/...` 路由访问时，会**实时生成新的预签名 URL**，不依赖 `Reference`
- Runner 的刷新机制确保 `Reference` 中的 URL 不会过期太久

### 错误 4：所有存储类型都经过文件服务

**修正**：
- S3 和 EXTERNAL 类型的前端优先使用 `externalLink`
- 这意味着：
  - 不经过 `/file/...` 路由
  - 下载时服务器不参与（S3 重定向或 EXTERNAL 直连）
  - 只有元数据获取时有 API 层面的权限校验

---

*报告更新时间：2026-05-05*
*基于代码版本：memos main 分支*
*修正内容：补充 EXTERNAL 存储类型完整分析，修正 DATABASE 类型结论*
