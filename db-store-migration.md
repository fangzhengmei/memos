# Memos 数据库层设计分析报告

## 1. 概述

Memos 采用了一个设计良好的三层数据库架构：

1. **驱动层 (Driver)**: 针对 SQLite、MySQL、PostgreSQL 的具体实现
2. **存储抽象层 (Store)**: 统一接口封装 + 缓存机制
3. **数据定义层 (Proto)**: Protocol Buffers 定义的数据结构

这种设计确保了：
- 多数据库支持的一致性
- 良好的可扩展性
- 版本升级时的数据兼容性

## 2. 迁移脚本系统

### 2.1 迁移文件组织

迁移脚本位于 `store/migration/{driver}/` 目录下，按版本号组织：

```
migration/
├── sqlite/
│   ├── 0.10/
│   │   └── 00__activity.sql
│   ├── 0.11/
│   │   ├── 00__user_avatar.sql
│   │   ├── 01__idp.sql
│   │   └── 02__storage.sql
│   ├── ...
│   ├── 0.28/
│   │   └── 00__user_identity.sql
│   └── LATEST.sql
├── mysql/
│   └── (版本目录结构相同)
└── postgres/
    └── (版本目录结构相同)
```

### 2.2 迁移文件命名规范

**格式**: `NN__description.sql`

- `NN`: 零填充的 patch 编号（从 00 开始）
- `description`: 人类可读的迁移描述
- `__`: 分隔符（`MigrateFileNameSplit` 常量）

**示例**:
- `00__user_identity.sql`
- `01__drop_activity.sql`

### 2.3 版本计算逻辑

迁移脚本的版本号通过路径提取：

```
路径: migration/sqlite/0.22/02__memo_payload.sql
     → minorVersion = "0.22"
     → patchVersion = 2 + 1 = 3
     → 完整版本 = "0.22.3"
```

**关键代码** (`store/migrator.go:321-342`):

```go
func (s *Store) getSchemaVersionOfMigrateScript(filePath string) (string, error) {
    minorVersion := elements[len(elements)-2]
    rawPatchVersion := strings.Split(elements[len(elements)-1], MigrateFileNameSplit)[0]
    patchVersion, err := strconv.Atoi(rawPatchVersion)
    return fmt.Sprintf("%s.%d", minorVersion, patchVersion+1), nil
}
```

### 2.4 迁移执行流程

**完整流程** (`store/migrator.go:97-136`):

```
Start
  ↓
preMigrate() ──┐
  ↓            │
  ├─ 检查数据库是否初始化 (driver.IsInitialized)
  │    ↓
  │    ├─ 未初始化 → 应用 LATEST.sql → 设置 schema_version
  │    └─ 已初始化 → 继续
  │
  └─ checkMinimumUpgradeVersion()
       ↓
       ├─ 版本 >= 0.22.0 → OK
       ├─ 版本 < 0.22.0 → 错误，提示先升级到 v0.25.3
       └─ 空版本 + 已初始化 → 旧版本安装，同样错误
  ↓
检查版本降级保护
  ↓
applyMigrations()
  ↓
  ├─ 收集所有迁移文件路径
  ├─ 字典序排序
  ├─ 开启事务
  ├─ 遍历文件：
  │   ├─ 提取文件版本
  │   ├─ 判断是否需要应用 (current < fileVersion <= target)
  │   └─ 执行 SQL
  ├─ 提交事务
  └─ 更新 schema_version
  ↓
Demo 模式 → seed()
  ↓
End
```

### 2.5 LATEST.sql 机制

对于全新安装，系统直接应用 `LATEST.sql` 而非逐个执行迁移脚本：

**优势**:
- 更快的初始化速度
- 避免执行数百个过时的迁移

**位置**: 每个数据库驱动目录下都有独立的 `LATEST.sql`

### 2.6 事务原子性

所有迁移操作都在单个事务中执行：

```go
tx, err := s.driver.GetDB().Begin()
if err != nil {
    return errors.Wrap(err, "failed to start transaction")
}
defer tx.Rollback()

// ... 执行所有迁移 ...

if err := tx.Commit(); err != nil {
    return errors.Wrap(err, "failed to commit migration transaction")
}
```

**保障**: 迁移要么全部成功，要么全部回滚。

### 2.7 版本管理

**Schema 版本存储**: `system_setting` 表中的 `name = 'BASIC'` 记录

**数据库表结构** (`store/migration/sqlite/LATEST.sql:1-7`):

```sql
CREATE TABLE system_setting (
  name TEXT NOT NULL,
  value TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  UNIQUE(name)
);
```

**实际存储示例**:
- `name`: `'BASIC'`
- `value`: JSON 字符串，包含 `schemaVersion` 和 `secretKey`
- `description`: 描述（可为空）

**Store 层中间结构体** (`store/instance_setting.go:12-16`):

```go
type InstanceSetting struct {
    Name        string  // 对应 system_setting.name
    Value       string  // 对应 system_setting.value (JSON 字符串)
    Description string  // 对应 system_setting.description
}
```

**Proto 层定义** (`proto/store/instance_setting.proto:40-45`):

```protobuf
message InstanceBasicSetting {
  string secret_key = 1;
  string schema_version = 2;  // 存储当前数据库的 schema 版本
}
```

**三层映射关系（以 Schema 版本为例）**:

```
┌─────────────────────────────────────────────────────────────────────┐
│  数据库层 (system_setting 表)                                        │
│  ┌──────────────┬────────────────────────────────┬──────────────┐  │
│  │ name         │ value                          │ description  │  │
│  ├──────────────┼────────────────────────────────┼──────────────┤  │
│  │ 'BASIC'      │ {"schemaVersion":"0.28.0",     │ ""           │  │
│  │              │  "secretKey":"abc123..."}      │              │  │
│  └──────────────┴────────────────────────────────┴──────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │ protojson.Marshal/Unmarshal
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Store 层 (store/instance_setting.go)                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ type InstanceSetting struct {                               │   │
│  │   Name        string  // "BASIC"                            │   │
│  │   Value       string  // "{\"schemaVersion\":\"0.28.0\",...}"│   │
│  │   Description string  // ""                                  │   │
│  │ }                                                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ convertInstanceSettingFromRaw()                              │   │
│  │   ↓ 根据 Name 选择正确的 oneof 类型                           │   │
│  │ convertInstanceSettingToRaw()                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────┘
                             │ oneof value 映射
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Proto 层 (proto/store/instance_setting.proto)                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ message InstanceSetting {                                   │   │
│  │   InstanceSettingKey key = 1;           // BASIC            │   │
│  │   oneof value {                                            │   │
│  │     InstanceBasicSetting basic_setting = 2;                 │   │
│  │     InstanceGeneralSetting general_setting = 3;             │   │
│  │     InstanceStorageSetting storage_setting = 4;             │   │
│  │     InstanceMemoRelatedSetting memo_related_setting = 5;    │   │
│  │     InstanceTagsSetting tags_setting = 6;                   │   │
│  │     InstanceNotificationSetting notification_setting = 7;    │   │
│  │     InstanceAISetting ai_setting = 8;                       │   │
│  │   }                                                         │   │
│  │ }                                                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**转换函数** (`store/instance_setting.go:275-327`):

```go
func convertInstanceSettingFromRaw(instanceSettingRaw *InstanceSetting) (*storepb.InstanceSetting, error) {
    instanceSetting := &storepb.InstanceSetting{
        Key: storepb.InstanceSettingKey(storepb.InstanceSettingKey_value[instanceSettingRaw.Name]),
    }
    switch instanceSettingRaw.Name {
    case storepb.InstanceSettingKey_BASIC.String():
        basicSetting := &storepb.InstanceBasicSetting{}
        if err := protojsonUnmarshaler.Unmarshal([]byte(instanceSettingRaw.Value), basicSetting); err != nil {
            return nil, err
        }
        instanceSetting.Value = &storepb.InstanceSetting_BasicSetting{BasicSetting: basicSetting}
    case storepb.InstanceSettingKey_GENERAL.String():
        // ... 类似处理
    // ... 其他 case
    default:
        return nil, nil
    }
    return instanceSetting, nil
}
```

**版本比较**: 使用 `golang.org/x/mod/semver` 进行语义化版本比较

```go
func IsVersionGreaterThan(version, target string) bool {
    return semver.Compare(fmt.Sprintf("v%s", version), fmt.Sprintf("v%s", target)) > 0
}
```

### 2.8 升级路径管理

**最低版本要求**: v0.22.0

对于 v0.22.0 之前的安装，必须先升级到 v0.25.3：

```go
func (s *Store) checkMinimumUpgradeVersion(ctx context.Context) error {
    if !isVersionEmpty(schemaVersion) && 
       version.IsVersionGreaterOrEqualThan(schemaVersion, "0.22.0") {
        return nil
    }
    return errors.Errorf(
        "Your Memos installation is too old to upgrade directly.\n" +
        "Upgrade path:\n" +
        "1. First upgrade to v0.25.3\n" +
        "2. Start the server and verify it works\n" +
        "3. Then upgrade to the latest version"
    )
}
```

**原因**: v0.22.0 将 schema 版本追踪从 `migration_history` 表迁移到了 `system_setting` 表。

### 2.9 迁移示例

**示例 1**: 简单列添加 (`0.22/02__memo_payload.sql`)

```sql
ALTER TABLE memo ADD COLUMN payload TEXT NOT NULL DEFAULT '{}';
```

**示例 2**: 复杂数据转换 (`0.27/02__migrate_inbox_message_payload.sql`)

```sql
UPDATE inbox
SET message = json_set(
  json_remove(message, '$.activityId'),
  '$.memoComment',
  json_object(
    'memoId',
    (SELECT json_extract(activity.payload, '$.memoComment.memoId')
     FROM activity WHERE activity.id = json_extract(inbox.message, '$.activityId')),
    ...
  )
)
WHERE json_extract(message, '$.activityId') IS NOT NULL;
```

**示例 3**: 枚举值更新 (`0.26/04__migrate_host_to_admin.sql`)

```sql
UPDATE user SET role = 'ADMIN' WHERE role = 'HOST';
```

### 2.10 Schema 版本管理完整调用链

迁移系统的核心入口是 `Migrate()` 函数，它协调整个版本管理流程。以下是完整的调用链分析：

#### 2.10.1 主入口函数

**位置**: `store/migrator.go:100-136`

```go
func (s *Store) Migrate(ctx context.Context) error {
    // 步骤 1: 预迁移（新库初始化）
    if err := s.preMigrate(ctx); err != nil {
        return errors.Wrap(err, "failed to pre-migrate")
    }

    // 步骤 2: 读取当前数据库中的 schema 版本
    instanceBasicSetting, err := s.GetInstanceBasicSetting(ctx)
    
    // 步骤 3: 获取代码期望的目标 schema 版本
    currentSchemaVersion, err := s.GetCurrentSchemaVersion()
    
    // 步骤 4: 检查是否是降级操作（禁止降级）
    if !isVersionEmpty(instanceBasicSetting.SchemaVersion) && 
       version.IsVersionGreaterThan(instanceBasicSetting.SchemaVersion, currentSchemaVersion) {
        return errors.Errorf("cannot downgrade schema version...")
    }
    
    // 步骤 5: 判断是否需要迁移
    if isVersionEmpty(instanceBasicSetting.SchemaVersion) || 
       version.IsVersionGreaterThan(currentSchemaVersion, instanceBasicSetting.SchemaVersion) {
        if err := s.applyMigrations(ctx, instanceBasicSetting.SchemaVersion, currentSchemaVersion); err != nil {
            return errors.Wrap(err, "failed to apply migrations")
        }
    }
    
    // 步骤 6: Demo 模式注入种子数据
    if s.profile.Demo {
        if err := s.seed(ctx); err != nil {
            return errors.Wrap(err, "failed to seed")
        }
    }
    return nil
}
```

#### 2.10.2 关键函数调用关系图

```
Migrate()
    │
    ├──► preMigrate()
    │       │
    │       ├──► driver.IsInitialized() ──┐
    │       │                             │ 未初始化
    │       │                             ▼
    │       │                    执行 LATEST.sql
    │       │                             │
    │       │                    ┌────────▼────────┐
    │       │                    │ GetCurrentSchemaVersion()
    │       │                    │     (从迁移文件计算目标版本)
    │       │                    └────────┬────────┘
    │       │                             │
    │       │                    ┌────────▼────────┐
    │       │                    │ updateCurrentSchemaVersion()
    │       │                    │  写入 system_setting (name='BASIC')
    │       │                    └────────┬────────┘
    │       │                             │
    │       │                    (新库初始化完成)
    │       │
    │       └──► checkMinimumUpgradeVersion() ──┐
    │                                            │ 旧版本检测
    │                                            ▼
    │                                    GetInstanceBasicSetting()
    │                                            │
    │                              ┌─────────────┴─────────────┐
    │                              ▼                           ▼
    │                        版本 >= 0.22.0              版本 < 0.22.0 或空
    │                              │                           │
    │                              OK                        报错退出
    │
    ├──► GetInstanceBasicSetting()  (读取当前数据库版本)
    │       │
    │       └──► GetInstanceSetting(name='BASIC')
    │               │
    │               ├──► 查缓存 instanceSettingCache
    │               │       ├── 命中 → 直接返回
    │               │       └── 未命中 → 继续
    │               │
    │               └──► ListInstanceSettings()
    │                       │
    │                       ├──► driver.ListInstanceSettings()
    │                       │       └──► SELECT FROM system_setting WHERE name='BASIC'
    │                       │
    │                       └──► convertInstanceSettingFromRaw()
    │                               └──► protojson.Unmarshal → storepb.InstanceBasicSetting
    │
    ├──► GetCurrentSchemaVersion()  (获取目标版本)
    │       │
    │       └──► 扫描所有迁移文件 → 提取版本号 → 返回最大版本
    │
    ├──► 降级检查 (DB版本 > 代码版本？)
    │       └──► 是 → 报错退出
    │
    ├──► applyMigrations(currentVersion, targetVersion)
    │       │
    │       ├──► 开启事务
    │       │
    │       ├──► 收集并排序所有迁移文件
    │       │
    │       ├──► 遍历文件：
    │       │       └──► shouldApplyMigration(fileVer, current, target)
    │       │               └──► 是 → 执行 SQL
    │       │
    │       ├──► 提交事务
    │       │
    │       └──► updateCurrentSchemaVersion(targetVersion)
    │               │
    │               ├──► GetInstanceBasicSetting()
    │               │
    │               ├──► 修改 SchemaVersion 字段
    │               │
    │               └──► UpsertInstanceSetting()
    │                       │
    │                       ├──► protojson.Marshal(InstanceBasicSetting)
    │                       │
    │                       ├──► driver.UpsertInstanceSetting()
    │                       │       └──► INSERT OR REPLACE INTO system_setting ...
    │                       │
    │                       └──► 更新缓存 instanceSettingCache
    │
    └──► seed()  (Demo 模式)
            └──► 注入种子数据（包含 system_setting 的 MEMO_RELATED 记录）
```

#### 2.10.3 核心函数详解

##### GetInstanceBasicSetting - 读取版本

**位置**: `store/instance_setting.go:108-125`

```go
func (s *Store) GetInstanceBasicSetting(ctx context.Context) (*storepb.InstanceBasicSetting, error) {
    // 1. 通过 GetInstanceSetting 读取
    instanceSetting, err := s.GetInstanceSetting(ctx, &FindInstanceSetting{
        Name: storepb.InstanceSettingKey_BASIC.String(),  // "BASIC"
    })
    
    // 2. 提取 oneof 中的 BasicSetting
    instanceBasicSetting := &storepb.InstanceBasicSetting{}
    if instanceSetting != nil {
        instanceBasicSetting = instanceSetting.GetBasicSetting()
    }
    
    // 3. 更新缓存
    s.instanceSettingCache.Set(ctx, storepb.InstanceSettingKey_BASIC.String(), ...)
    
    return instanceBasicSetting, nil
}
```

**关键点**:
- 如果 `name='BASIC'` 记录不存在，返回空的 `InstanceBasicSetting{}`（SchemaVersion = ""）
- `isVersionEmpty("")` = true，`isVersionEmpty("0.0.0")` = true

##### GetCurrentSchemaVersion - 计算目标版本

**位置**: `store/migrator.go:299-319`

```go
func (s *Store) GetCurrentSchemaVersion() (string, error) {
    filePaths, err := fs.Glob(migrationFS, fmt.Sprintf("%s*/*.sql", s.getMigrationBasePath()))
    
    currentSchemaVersion := defaultSchemaVersion  // "0.0.0"
    for _, filePath := range filePaths {
        fileSchemaVersion, err := s.getSchemaVersionOfMigrateScript(filePath)
        if version.IsVersionGreaterThan(fileSchemaVersion, currentSchemaVersion) {
            currentSchemaVersion = fileSchemaVersion
        }
    }
    return currentSchemaVersion, nil
}
```

**版本计算规则**:
- 扫描 `migration/{driver}/*/*.sql` 所有文件
- 提取路径中的版本号：`0.22/02__xxx.sql` → `0.22.3`
- 返回最大版本号

##### updateCurrentSchemaVersion - 写入版本

**位置**: `store/migrator.go:355-368`

```go
func (s *Store) updateCurrentSchemaVersion(ctx context.Context, schemaVersion string) error {
    // 1. 读取当前配置
    instanceBasicSetting, err := s.GetInstanceBasicSetting(ctx)
    
    // 2. 修改版本号
    instanceBasicSetting.SchemaVersion = schemaVersion
    
    // 3. 写回数据库
    if _, err := s.UpsertInstanceSetting(ctx, &storepb.InstanceSetting{
        Key:   storepb.InstanceSettingKey_BASIC,
        Value: &storepb.InstanceSetting_BasicSetting{BasicSetting: instanceBasicSetting},
    }); err != nil {
        return errors.Wrap(err, "failed to upsert instance setting")
    }
    return nil
}
```

**写入时的三层转换**:

```
Proto 层 (storepb.InstanceSetting):
  { Key: BASIC, Value: BasicSetting{SchemaVersion: "0.28.0", SecretKey: "..."} }
        ↓ protojson.Marshal
Store 层 (InstanceSetting):
  { Name: "BASIC", Value: "{\"schemaVersion\":\"0.28.0\",\"secretKey\":\"...\"}", Description: "" }
        ↓ driver.UpsertInstanceSetting
数据库层 (system_setting 表):
  name:  'BASIC'
  value: '{"schemaVersion":"0.28.0","secretKey":"..."}'
  description: ''
```

### 2.11 三种场景下的 system_setting 状态变化

#### 2.11.1 场景一：新库初始化

**触发条件**: `driver.IsInitialized()` = false（无 `memo` 表）

**执行路径**:

```
Migrate()
    └──► preMigrate()
            ├──► IsInitialized() = false
            │
            ├──► 读取 LATEST.sql
            │       └──► 包含 CREATE TABLE system_setting (name, value, description)
            │
            ├──► 事务执行 LATEST.sql (仅建表，无初始数据)
            │
            ├──► GetCurrentSchemaVersion()
            │       └──► 返回 "0.28.0" (最大迁移文件版本)
            │
            └──► updateCurrentSchemaVersion("0.28.0")
                    ├──► GetInstanceBasicSetting()
                    │       └──► system_setting 中无 'BASIC' 记录
                    │       └──► 返回 InstanceBasicSetting{SchemaVersion: "", SecretKey: ""}
                    │
                    ├──► 设置 SchemaVersion = "0.28.0"
                    │
                    └──► UpsertInstanceSetting()
                            └──► INSERT INTO system_setting
                                    (name='BASIC', value='{"schemaVersion":"0.28.0",...}')
```

**system_setting 状态变化**:

| 时间点 | 记录存在性 | name='BASIC' | value |
|--------|-----------|--------------|-------|
| **初始化前** | 无表 | - | - |
| **LATEST.sql 执行后** | 表存在，无记录 | 不存在 | - |
| **updateCurrentSchemaVersion 后** | 表存在，1 条记录 | `'BASIC'` | `{"schemaVersion":"0.28.0","secretKey":""}` |

**注意**: `secretKey` 是空字符串，后续由其他逻辑生成并更新。

#### 2.11.2 场景二：旧库升级

**触发条件**: 
- `IsInitialized()` = true（有数据）
- 数据库版本 < 代码版本
- 数据库版本 >= 0.22.0

**执行路径** (假设 DB=0.27.0, 代码=0.28.0):

```
Migrate()
    ├──► preMigrate()
    │       ├──► IsInitialized() = true
    │       │
    │       └──► checkMinimumUpgradeVersion()
    │               ├──► GetInstanceBasicSetting()
    │               │       └──► 返回 {SchemaVersion: "0.27.0"}
    │               │
    │               └──► 0.27.0 >= 0.22.0 → OK
    │
    ├──► GetInstanceBasicSetting()
    │       └──► SchemaVersion = "0.27.0" (数据库当前版本)
    │
    ├──► GetCurrentSchemaVersion()
    │       └──► 目标版本 = "0.28.0"
    │
    ├──► 降级检查: 0.27.0 > 0.28.0? → NO
    │
    ├──► 版本比较: 0.28.0 > 0.27.0? → YES, 需要迁移
    │
    ├──► applyMigrations("0.27.0", "0.28.0")
    │       │
    │       ├──► 收集所有迁移文件
    │       │
    │       ├──► 遍历判断:
    │       │       ├──► 0.10/* → 0.10.x > 0.27.0? NO
    │       │       ├──► ...
    │       │       ├──► 0.27/* → 0.27.x > 0.27.0? 部分 YES
    │       │       └──► 0.28/* → 0.28.x > 0.27.0? YES
    │       │
    │       ├──► 事务执行符合条件的迁移 SQL
    │       │
    │       ├──► 提交事务
    │       │
    │       └──► updateCurrentSchemaVersion("0.28.0")
    │               ├──► GetInstanceBasicSetting() → {SchemaVersion: "0.27.0"}
    │               ├──► 修改 SchemaVersion = "0.28.0"
    │               └──► UpsertInstanceSetting() (REPLACE)
    │
    └──► seed() (Demo 模式才执行)
```

**system_setting 状态变化**:

| 时间点 | name='BASIC' 的 value |
|--------|----------------------|
| **迁移前** | `{"schemaVersion":"0.27.0","secretKey":"abc"}` |
| **applyMigrations 中** | 保持 `0.27.0` (事务中) |
| **事务提交后** | 保持 `0.27.0` (还未更新 version 记录) |
| **updateCurrentSchemaVersion 后** | `{"schemaVersion":"0.28.0","secretKey":"abc"}` |

**关键点**:
- 迁移 SQL 和 version 更新分两个事务
- 如果迁移 SQL 执行失败，version 记录保持不变
- 如果 SQL 成功但 version 更新失败，下次启动会重新迁移（可能需要幂等性）

#### 2.11.3 场景三：降级拦截

**触发条件**: 
- `IsInitialized()` = true
- 数据库版本 > 代码版本

**执行路径** (假设 DB=0.29.0, 代码=0.28.0):

```
Migrate()
    ├──► preMigrate()
    │       ├──► IsInitialized() = true
    │       │
    │       └──► checkMinimumUpgradeVersion()
    │               └──► 0.29.0 >= 0.22.0 → OK
    │
    ├──► GetInstanceBasicSetting()
    │       └──► SchemaVersion = "0.29.0" (数据库版本)
    │
    ├──► GetCurrentSchemaVersion()
    │       └──► 目标版本 = "0.28.0" (代码版本)
    │
    ├──► 降级检查: 0.29.0 > 0.28.0? → YES!
    │
    ├──► 日志错误: cannot downgrade schema version
    │
    └──► 返回错误，程序退出
```

**system_setting 状态变化**:

| 时间点 | name='BASIC' 的 value | 说明 |
|--------|----------------------|------|
| **检查前** | `{"schemaVersion":"0.29.0",...}` | 高版本数据 |
| **降级检查后** | 保持不变 | 程序报错退出，无任何修改 |

**保护机制代码** (`store/migrator.go:113-120`):

```go
// Check for downgrade (but skip if schema version is empty - that means fresh/old installation)
if !isVersionEmpty(instanceBasicSetting.SchemaVersion) && 
   version.IsVersionGreaterThan(instanceBasicSetting.SchemaVersion, currentSchemaVersion) {
    slog.Error("cannot downgrade schema version",
        slog.String("databaseVersion", instanceBasicSetting.SchemaVersion),
        slog.String("currentVersion", currentSchemaVersion),
    )
    return errors.Errorf("cannot downgrade schema version from %s to %s", 
        instanceBasicSetting.SchemaVersion, currentSchemaVersion)
}
```

**例外情况**: `isVersionEmpty()` = true 时不检查降级

这允许：
- 新库初始化（version 为空）
- v0.22 之前的旧库升级（version 可能为空或 0.0.0）

### 2.12 三种场景对比总结

| 维度 | 新库初始化 | 旧库升级 | 降级拦截 |
|------|-----------|----------|----------|
| **触发条件** | 无 memo 表 | 0.22 <= DB < 代码 | DB > 代码 |
| **IsInitialized** | false | true | true |
| **LATEST.sql** | 执行 | 不执行 | 不执行 |
| **迁移脚本** | 不执行 | 执行增量 | 不执行 |
| **updateCurrentSchemaVersion** | 执行 1 次 | 执行 1 次 | 不执行 |
| **system_setting 变化** | 新增 'BASIC' 记录 | 更新 'BASIC' 的 value | 无变化 |
| **最终 schema_version** | 代码版本 | 代码版本 | 保持高版本 |
| **结果** | 成功 | 成功 | 失败退出 |
| **seed() 调用** | Demo 模式调用 | Demo 模式调用 | 不调用 |

### 2.13 version 字段的特殊值处理

```go
const defaultSchemaVersion = "0.0.0"

func getSchemaVersionOrDefault(schemaVersion string) string {
    if schemaVersion == "" {
        return defaultSchemaVersion  // "" → "0.0.0"
    }
    return schemaVersion
}

func isVersionEmpty(schemaVersion string) bool {
    return schemaVersion == "" || schemaVersion == defaultSchemaVersion
}
```

**版本状态机**:

```
空字符串 ("")
    │
    ├──► GetInstanceBasicSetting 返回空
    │       └──► isVersionEmpty = true
    │
    ├──► getSchemaVersionOrDefault 转换
    │       └──► "0.0.0" (用于版本比较)
    │
    └──► 含义:
            ├──► 新库刚创建（LATEST.sql 已执行但还没写 version）
            ├──► v0.22 之前的旧库（system_setting 为空）
            └──► 配置丢失或损坏
```

## 3. 存储抽象层设计

### 3.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      上层业务层                                    │
│        (server/services, runner, etc.)                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                      Store 层                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  业务方法封装 (CreateUser, ListMemos, etc.)                │  │
│  │  + 缓存机制 (instanceSetting, user, userSetting)          │  │
│  │  + Proto ↔ Raw 结构体转换 (convert*FromRaw/ToRaw)         │  │
│  └───────────────────────────┬───────────────────────────────┘  │
│                              │                                    │
│  ┌───────────────────────────▼───────────────────────────────┐  │
│  │              Driver 接口 (store/driver.go)                │  │
│  └───────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────▼───────┐    ┌────────▼────────┐    ┌────────▼────────┐
│   SQLite      │    │     MySQL       │    │    PostgreSQL   │
│ store/db/     │    │  store/db/      │    │  store/db/      │
│ sqlite/       │    │   mysql/        │    │  postgres/      │
└───────────────┘    └─────────────────┘    └─────────────────┘
```

### 3.2 Driver 接口定义

**位置**: `store/driver.go`

```go
type Driver interface {
    // 基础方法
    GetDB() *sql.DB
    Close() error
    IsInitialized(ctx context.Context) (bool, error)
    GetDatabaseSize(ctx context.Context) (int64, error)

    // Attachment 模型
    CreateAttachment(...)
    ListAttachments(...)
    UpdateAttachment(...)
    DeleteAttachment(...)
    DeleteAttachments(...)

    // Memo 模型
    CreateMemo(...)
    ListMemos(...)
    UpdateMemo(...)
    DeleteMemo(...)

    // ... 其他模型: MemoRelation, InstanceSetting, User,
    //     UserSetting, IdentityProvider, Inbox, Reaction,
    //     MemoShare, UserIdentity
}
```

**特点**:
- 每个模型都有 CRUD 操作
- 使用 `Find*` 结构体进行复杂查询条件封装
- 返回指针类型，支持 nil 返回

### 3.3 驱动工厂

**位置**: `store/db/db.go`

```go
func NewDBDriver(profile *profile.Profile) (store.Driver, error) {
    switch profile.Driver {
    case "sqlite":
        return sqlite.NewDB(profile)
    case "mysql":
        return mysql.NewDB(profile)
    case "postgres":
        return postgres.NewDB(profile)
    default:
        return nil, errors.New("unknown db driver")
    }
}
```

### 3.4 各数据库驱动实现

#### SQLite 驱动 (`store/db/sqlite/sqlite.go`)

```go
func NewDB(profile *profile.Profile) (store.Driver, error) {
    sqliteDB, err := sql.Open("sqlite", 
        profile.DSN + 
        "?_pragma=foreign_keys(0)" +
        "&_pragma=busy_timeout(10000)" +
        "&_pragma=journal_mode(WAL)" +
        "&_pragma=mmap_size(0)")
    // ...
}
```

**关键配置**:
- `foreign_keys(0)`: 禁用外键约束（简化迁移）
- `journal_mode(WAL)`: WAL 模式，提升并发性能
- `busy_timeout(10000)`: 10 秒锁等待超时
- `mmap_size(0)`: 禁用内存映射（避免 OOM）

**初始化检测**: 检查 `memo` 表是否存在

```go
func (d *DB) IsInitialized(ctx context.Context) (bool, error) {
    var exists bool
    err := d.db.QueryRowContext(ctx, 
        "SELECT EXISTS(SELECT 1 FROM sqlite_master WHERE type='table' AND name='memo')"
    ).Scan(&exists)
    return exists, err
}
```

#### MySQL 驱动 (`store/db/mysql/mysql.go`)

**关键配置**:
- `MultiStatements = true`: 支持一次执行多个 SQL 语句（迁移必需）

```go
func mergeDSN(baseDSN string) (string, error) {
    config, err := mysql.ParseDSN(baseDSN)
    config.MultiStatements = true
    return config.FormatDSN(), nil
}
```

#### PostgreSQL 驱动

采用类似模式，使用 PostgreSQL 特定的 SQL 语法。

### 3.5 Store 层封装

**位置**: `store/store.go`

```go
type Store struct {
    profile *profile.Profile
    driver  Driver

    userCreateMu sync.Mutex  // 用户创建互斥锁

    // 缓存配置
    cacheConfig cache.Config

    // 缓存实例
    instanceSettingCache *cache.Cache
    userCache            *cache.Cache
    userSettingCache     *cache.Cache
}
```

**Store 层职责**:

1. **参数验证**: 如 UID 格式校验 (`store/memo.go:109-113`)

```go
func (s *Store) CreateMemo(ctx context.Context, create *Memo) (*Memo, error) {
    if !base.UIDMatcher.MatchString(create.UID) {
        return nil, errors.New("invalid uid")
    }
    return s.driver.CreateMemo(ctx, create)
}
```

2. **业务逻辑封装**: 如删除 memo 时清理关联数据 (`store/memo.go:140-158`)

```go
func (s *Store) DeleteMemo(ctx context.Context, delete *DeleteMemo) error {
    // 清理 memo_relation
    if err := s.driver.DeleteMemoRelation(ctx, 
        &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    // 清理关联附件
    attachments, err := s.ListAttachments(ctx, 
        &FindAttachment{MemoID: &delete.ID})
    for _, attachment := range attachments {
        if err := s.DeleteAttachment(ctx, 
            &DeleteAttachment{ID: attachment.ID}); err != nil {
            return err
        }
    }
    return s.driver.DeleteMemo(ctx, delete)
}
```

3. **缓存管理**: 见下一节

### 3.6 缓存机制

**位置**: `store/cache/cache.go`

**缓存配置**:
- 默认 TTL: 10 分钟
- 清理间隔: 5 分钟
- 最大条目: 1000

**缓存的对象**:
- `instanceSettingCache`: 实例设置（高频读取）
- `userCache`: 用户信息
- `userSettingCache`: 用户设置

**缓存使用示例** (`store/instance_setting.go:87-106`):

```go
func (s *Store) GetInstanceSetting(ctx context.Context, find *FindInstanceSetting) (*storepb.InstanceSetting, error) {
    // 先查缓存
    if cache, ok := s.instanceSettingCache.Get(ctx, find.Name); ok {
        if instanceSetting, ok := cache.(*storepb.InstanceSetting); ok {
            return instanceSetting, nil
        }
    }
    
    // 缓存未命中，查数据库
    list, err := s.ListInstanceSettings(ctx, find)
    // ...
    return list[0], nil
}

func (s *Store) UpsertInstanceSetting(...) (*storepb.InstanceSetting, error) {
    // ... 写入数据库 ...
    s.instanceSettingCache.Set(ctx, instanceSetting.Key.String(), instanceSetting)
    return instanceSetting, nil
}
```

**缓存特性**:
- 线程安全（使用 `sync.Map` + `atomic`）
- TTL 过期自动清理
- 容量限制 + LRU-like 淘汰策略
- 支持 eviction 回调

## 4. 数据结构版本兼容

### 4.1 Proto 定义层

**位置**: `proto/store/*.proto`

Memos 使用 Protocol Buffers 定义核心数据结构，这天然提供了版本兼容性：

**MemoPayload 示例** (`proto/store/memo.proto`):

```protobuf
message MemoPayload {
  Property property = 1;
  Location location = 2;
  repeated string tags = 3;

  message Property {
    bool has_link = 1;
    bool has_task_list = 2;
    bool has_code = 3;
    bool has_incomplete_tasks = 4;
    string title = 5;  // 新增字段
  }
}
```

**Proto 的兼容性保障**:

1. **字段编号不可变**: 每个字段有唯一编号，新增字段不影响旧代码
2. **默认值机制**: 缺失字段使用类型默认值
3. **可选性**: proto3 所有字段都是可选的

### 4.2 字段预留机制

**位置**: `proto/store/instance_setting.proto:106-107`

```protobuf
message InstanceMemoRelatedSetting {
  reserved 2;
  reserved "display_with_update_time";
  // ...
}
```

**作用**: 防止已删除/重命名字段的编号被复用，避免版本冲突。

### 4.3 JSON 序列化与未知字段处理

**位置**: `store/common.go:5-10`

```go
var (
    protojsonUnmarshaler = protojson.UnmarshalOptions{
        AllowPartial:   true,   // 允许部分字段缺失
        DiscardUnknown: true,   // 忽略未知字段（向前兼容）
    }
)
```

**关键配置 `DiscardUnknown: true`**:

- **场景**: 新版本代码读取旧版本写入的数据（可能缺少新字段）
- **场景**: 旧版本代码读取新版本写入的数据（可能有额外字段）
- **效果**: 未知字段被静默忽略，不抛出错误

**使用示例** (`store/db/sqlite/memo.go:180-184`):

```go
payload := &storepb.MemoPayload{}
if err := protojsonUnmarshaler.Unmarshal(payloadBytes, payload); err != nil {
    return nil, errors.Wrap(err, "failed to unmarshal payload")
}
memo.Payload = payload
```

### 4.4 数据库存储格式

**设计**: 结构化数据以 JSON 格式存储在 TEXT 列中

**示例表结构** (`store/migration/sqlite/LATEST.sql`):

```sql
CREATE TABLE memo (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  uid TEXT NOT NULL UNIQUE,
  -- ... 标准字段 ...
  payload TEXT NOT NULL DEFAULT '{}'  -- JSON 格式的 MemoPayload
);

CREATE TABLE attachment (
  -- ...
  payload TEXT NOT NULL DEFAULT '{}'  -- JSON 格式的 AttachmentPayload
);
```

**优势**:
- 无需 ALTER TABLE 即可添加新字段
- 各数据库驱动统一处理
- Proto ↔ JSON 无缝转换

### 4.5 默认值与回退机制

**位置**: `store/instance_setting.go`

对于设置类数据，Store 层提供了默认值回退：

```go
func (s *Store) GetInstanceStorageSetting(ctx context.Context) (*storepb.InstanceStorageSetting, error) {
    instanceSetting, err := s.GetInstanceSetting(ctx, ...)
    
    instanceStorageSetting := &storepb.InstanceStorageSetting{}
    if instanceSetting != nil {
        instanceStorageSetting = instanceSetting.GetStorageSetting()
    }
    
    // 默认值回退
    if instanceStorageSetting.StorageType == storepb.InstanceStorageSetting_STORAGE_TYPE_UNSPECIFIED {
        instanceStorageSetting.StorageType = defaultInstanceStorageType  // LOCAL
    }
    if instanceStorageSetting.UploadSizeLimitMb == 0 {
        instanceStorageSetting.UploadSizeLimitMb = defaultInstanceUploadSizeLimitMb  // 30
    }
    if instanceStorageSetting.FilepathTemplate == "" {
        instanceStorageSetting.FilepathTemplate = defaultInstanceFilepathTemplate
    }
    
    return instanceStorageSetting, nil
}
```

**保障**: 旧版本数据库缺少新设置时，使用合理默认值。

### 4.6 数据迁移中的兼容性处理

**示例 1**: 角色重命名迁移 (`0.26/04__migrate_host_to_admin.sql`)

```sql
UPDATE user SET role = 'ADMIN' WHERE role = 'HOST';
```

- 旧值 `HOST` → 新值 `ADMIN`
- 在 SQL 层面完成数据转换

**示例 2**: Inbox 消息结构迁移 (`0.27/02__migrate_inbox_message_payload.sql`)

```sql
UPDATE inbox
SET message = json_set(
  json_remove(message, '$.activityId'),  -- 移除旧字段
  '$.memoComment',                       -- 添加新字段
  json_object(...)
)
WHERE ...
```

- 使用数据库原生 JSON 函数进行结构转换
- 保持数据完整性

### 4.7 版本降级保护

**位置**: `store/migrator.go:113-120`

```go
// 检查降级（但如果 schema version 为空则跳过 - 表示新安装/旧安装）
if !isVersionEmpty(instanceBasicSetting.SchemaVersion) && 
   version.IsVersionGreaterThan(instanceBasicSetting.SchemaVersion, currentSchemaVersion) {
    slog.Error("cannot downgrade schema version",
        slog.String("databaseVersion", instanceBasicSetting.SchemaVersion),
        slog.String("currentVersion", currentSchemaVersion),
    )
    return errors.Errorf("cannot downgrade schema version from %s to %s", 
        instanceBasicSetting.SchemaVersion, currentSchemaVersion)
}
```

**保护**: 防止新版本数据库被旧版本代码访问。

### 4.8 测试验证

**位置**: `store/test/migrator_test.go`

关键测试用例：

1. **TestFreshInstall**: 验证全新安装时 LATEST.sql 正确应用
2. **TestMigrationReRun**: 验证迁移可重复执行（幂等性）
3. **TestMigrationWithData**: 验证迁移不破坏现有数据
4. **TestMigrationMultipleReRuns**: 验证多次重复执行无问题
5. **TestMigrationFromStableVersion**: 验证从稳定版本升级到当前版本

```go
func TestMigrationFromStableVersion(t *testing.T) {
    // 1. 启动旧版本 Memos 容器创建数据库
    // 2. 停止容器
    // 3. 使用当前代码运行迁移
    // 4. 验证迁移成功并可写入数据
}
```

## 5. 三层数据模型映射详解

### 5.1 映射架构总览

Memos 采用三层数据模型设计，各层职责明确：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      上层业务层 (Services/Handlers)                  │
│                    使用 Proto 类型 (storepb.*)                       │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     Store 层转换        │
                    │  convert*FromRaw/ToRaw │
                    └────────────┬────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                         Store 层 (Raw 结构体)                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  InstanceSetting: Name/Value(string)/Description              │ │
│  │  UserSetting: UserID/Key/Value(string)                        │ │
│  │  Memo: ID/UID/Content/Visibility/Pinned/Payload(*Proto)       │ │
│  └───────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Driver 层 SQL 执行     │
                    │  protojson Marshal     │
                    └────────────┬────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      数据库层 (Tables)                               │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  system_setting: name/value(JSON)/description                 │ │
│  │  user_setting: user_id/key/value(JSON)                        │ │
│  │  memo: id/uid/content/visibility/pinned/payload(JSON)         │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 InstanceSetting 映射详解

| 层级 | 类型/结构 | 关键字段 |
|------|-----------|----------|
| **数据库层** | `system_setting` 表 | `name` (TEXT), `value` (TEXT JSON), `description` (TEXT) |
| **Store 层** | `store.InstanceSetting` 结构体 | `Name string`, `Value string`, `Description string` |
| **Proto 层** | `storepb.InstanceSetting` message | `key` (enum), `value` (oneof) |

**转换流程**:

```
写入 (Upsert):
Proto (InstanceSetting{Key: BASIC, Value: BasicSetting{SchemaVersion: "0.28.0"}})
  ↓ protojson.Marshal
Store (InstanceSetting{Name: "BASIC", Value: "{\"schemaVersion\":\"0.28.0\"}", Description: ""})
  ↓ SQL INSERT
DB (system_setting: name='BASIC', value='{"schemaVersion":"0.28.0"}', description='')

读取 (List):
DB (system_setting: name='BASIC', value='{"schemaVersion":"0.28.0"}', description='')
  ↓ SQL SELECT
Store (InstanceSetting{Name: "BASIC", Value: "{\"schemaVersion\":\"0.28.0\"}", Description: ""})
  ↓ convertInstanceSettingFromRaw → protojson.Unmarshal
Proto (InstanceSetting{Key: BASIC, Value: BasicSetting{SchemaVersion: "0.28.0"}})
```

### 5.3 UserSetting 映射详解

| 层级 | 类型/结构 | 关键字段 |
|------|-----------|----------|
| **数据库层** | `user_setting` 表 | `user_id` (INTEGER), `key` (TEXT), `value` (TEXT JSON) |
| **Store 层** | `store.UserSetting` 结构体 | `UserID int32`, `Key storepb.UserSetting_Key`, `Value string` |
| **Proto 层** | `storepb.UserSetting` message | `user_id`, `key` (enum), `value` (oneof) |

**转换函数** (`store/user_setting.go:419-508`):

```go
func convertUserSettingFromRaw(raw *UserSetting) (*storepb.UserSetting, error) {
    userSetting := &storepb.UserSetting{
        UserId: raw.UserID,
        Key:    raw.Key,
    }
    switch raw.Key {
    case storepb.UserSetting_REFRESH_TOKENS:
        refreshTokensUserSetting := &storepb.RefreshTokensUserSetting{}
        if err := protojsonUnmarshaler.Unmarshal([]byte(raw.Value), refreshTokensUserSetting); err != nil {
            return nil, err
        }
        userSetting.Value = &storepb.UserSetting_RefreshTokens{RefreshTokens: refreshTokensUserSetting}
    // ... 其他 case
    }
    return userSetting, nil
}
```

### 5.4 Memo 映射详解

| 层级 | 类型/结构 | 关键字段 |
|------|-----------|----------|
| **数据库层** | `memo` 表 | `id`, `uid`, `creator_id`, `content`, `visibility`, `pinned`, `payload` (TEXT JSON) |
| **Store 层** | `store.Memo` 结构体 | `ID int32`, `UID string`, `Content string`, `Visibility`, `Pinned bool`, `Payload *storepb.MemoPayload` |
| **Proto 层** | `storepb.MemoPayload` message | `property`, `location`, `tags` (嵌套结构) |

**注意**: Memo 的大部分字段直接映射，只有 `Payload` 字段需要 JSON 序列化：

```go
// 写入时 (store/db/sqlite/memo.go:20-26)
payload := "{}"
if create.Payload != nil {
    payloadBytes, err := protojson.Marshal(create.Payload)
    if err != nil {
        return nil, err
    }
    payload = string(payloadBytes)
}

// 读取时 (store/db/sqlite/memo.go:180-184)
payload := &storepb.MemoPayload{}
if err := protojsonUnmarshaler.Unmarshal(payloadBytes, payload); err != nil {
    return nil, errors.Wrap(err, "failed to unmarshal payload")
}
memo.Payload = payload
```

### 5.5 各层职责总结

| 层级 | 职责 | 设计优势 |
|------|------|----------|
| **数据库层** | 物理存储，关系型表结构 | 标准化、可查询、ACID |
| **Store 层 (Raw)** | 中间转换，统一驱动接口 | 隔离数据库差异，便于多数据库支持 |
| **Proto 层** | 强类型定义，版本兼容 | 向后/向前兼容，API 稳定 |

## 6. 关键设计总结

### 6.1 迁移系统设计要点

| 设计点 | 实现方式 | 优势 |
|--------|----------|------|
| 版本追踪 | `system_setting` 表中 `name='BASIC'` 记录 | 统一管理，易于查询 |
| 全新安装 | 直接应用 LATEST.sql | 快速初始化 |
| 增量升级 | 按版本号顺序执行脚本 | 精确控制升级过程 |
| 原子性 | 单事务包裹所有迁移 | 部分失败自动回滚 |
| 降级保护 | 版本比较检查 | 防止版本回退 |
| 旧版本支持 | 强制升级到 v0.25.3 过渡 | 平滑的升级路径 |

### 6.2 存储抽象层设计要点

| 层级 | 职责 | 关键实现 |
|------|------|----------|
| Driver | 数据库特定实现 | SQLite/MySQL/PostgreSQL 各自实现 Driver 接口 |
| Store | 业务封装 + 缓存 + 类型转换 | 参数验证、关联操作、缓存管理、Proto↔Raw 转换 |
| Proto | 数据定义 | 版本兼容的结构化数据 |

### 6.3 版本兼容性策略

1. **Proto 字段编号**: 永不复用已分配的字段编号
2. **DiscardUnknown**: 反序列化时忽略未知字段
3. **默认值回退**: 缺失配置使用合理默认值
4. **SQL 迁移**: 在数据库层面完成数据结构转换
5. **版本检查**: 启动时验证 schema 版本兼容性
6. **字段预留**: Proto 中使用 `reserved` 防止字段编号复用

### 6.4 三层映射策略

1. **直接映射字段**: 如 `memo.id`, `memo.uid`, `memo.content` — 直接读写
2. **JSON 序列化字段**: 如 `memo.payload`, `system_setting.value` — Proto ↔ JSON ↔ DB
3. **枚举映射**: 如 `InstanceSettingKey` — 枚举值 ↔ 字符串 ↔ DB
4. **Oneof 映射**: 如 `InstanceSetting.value` — 根据 key 选择正确的反序列化类型

### 6.5 目录结构参考

```
store/
├── driver.go              # Driver 接口定义
├── store.go               # Store 层封装
├── migrator.go            # 迁移系统核心
├── common.go              # 通用配置（protojson）
├── memo.go                # Memo 业务方法
├── user.go                # User 业务方法
├── user_setting.go        # UserSetting（含 convert 函数）
├── instance_setting.go    # InstanceSetting（含缓存 + convert 函数）
│
├── db/
│   ├── db.go              # 驱动工厂
│   ├── sqlite/
│   │   ├── sqlite.go      # SQLite 驱动实现
│   │   ├── memo.go        # SQLite Memo CRUD
│   │   ├── instance_setting.go  # SQLite InstanceSetting CRUD
│   │   └── ...
│   ├── mysql/
│   └── postgres/
│
├── migration/
│   ├── sqlite/
│   │   ├── 0.10/          # 版本 0.10.x 迁移
│   │   ├── ...
│   │   ├── 0.28/          # 版本 0.28.x 迁移
│   │   └── LATEST.sql     # 最新完整 schema
│   ├── mysql/
│   └── postgres/
│
├── cache/
│   └── cache.go           # 内存缓存实现
│
└── test/
    └── migrator_test.go   # 迁移测试
```

## 7. 最佳实践参考

基于 Memos 的设计，可提炼出以下数据库层设计最佳实践：

1. **驱动抽象**: 使用统一接口支持多数据库
2. **三层 Store**: Driver 层做纯 CRUD，Store 层封装业务逻辑和类型转换，Proto 层定义数据结构
3. **JSON + Proto**: 灵活的结构化数据存储 + 强类型定义
4. **版本化迁移**: 语义化版本号 + 增量脚本 + LATEST 快照
5. **DiscardUnknown**: 反序列化时忽略未知字段实现向前兼容
6. **字段预留**: Proto 中使用 `reserved` 防止字段编号复用
7. **默认值策略**: 缺失配置使用合理默认值而非报错
8. **事务迁移**: 所有迁移在单事务中执行保证原子性
9. **升级测试**: 测试从旧版本到新版本的完整升级路径
10. **缓存策略**: 高频读取数据使用内存缓存，写入时更新缓存
11. **中间层转换**: 使用 `convert*FromRaw/ToRaw` 函数隔离各层差异
12. **Oneof 模式**: 使用 Proto oneof + switch-case 处理多类型配置
