# Memos 内容关系分析报告（修订版）

## 1. 整体数据结构概览

Memos 使用**统一的关系表模型**来维护内容之间的图谱关系。核心设计是将评论和引用都抽象为 `MemoRelation`，通过 `type` 字段区分关系类型。

### 1.1 核心数据表

**memo 表** (`store/migration/sqlite/LATEST.sql:32-44`):
- 存储所有内容节点（包括主笔记和评论）
- 关键字段：`id`, `uid`, `creator_id`, `content`, `visibility`, `payload`
- 每个内容条目都是独立的 memo 实体

**memo_relation 表** (`store/migration/sqlite/LATEST.sql:46-52`):
```sql
CREATE TABLE memo_relation (
  memo_id INTEGER NOT NULL,          -- 源 memo ID
  related_memo_id INTEGER NOT NULL,  -- 目标 memo ID
  type TEXT NOT NULL,                -- 关系类型
  UNIQUE(memo_id, related_memo_id, type)  -- 组合唯一约束
);
```

## 2. 关系类型定义

在 `store/memo_relation.go:7-14` 中定义了两种关系类型：

```go
const (
    MemoRelationReference MemoRelationType = "REFERENCE"  // 引用关系
    MemoRelationComment   MemoRelationType = "COMMENT"    // 评论关系
)
```

### 2.1 MemoRelation 结构体 (`store/memo_relation.go:16-20`)

```go
type MemoRelation struct {
    MemoID        int32            // 源 memo 的 ID
    RelatedMemoID int32            // 目标 memo 的 ID
    Type          MemoRelationType // 关系类型：REFERENCE 或 COMMENT
}
```

**方向语义**：
- `MemoID → RelatedMemoID` 表示"源指向目标"
- 对于 `COMMENT`：`MemoID` 是评论，`RelatedMemoID` 是被评论的原笔记
- 对于 `REFERENCE`：`MemoID` 是引用者，`RelatedMemoID` 是被引用的笔记

## 3. 评论（Comment）数据组织

### 3.1 评论的双重身份

评论在 Memos 中具有**双重身份**：
1. **作为独立内容**：评论本身就是一个 memo，存储在 `memo` 表中
2. **作为关系边**：通过 `memo_relation` 表的 `COMMENT` 类型建立与原笔记的关联

### 3.2 ParentUID 机制：完全依赖实时 JOIN

**重要修正**：`ParentUID` 不是缓存字段，而是每次查询时通过实时 JOIN 计算得出。

在所有三个数据库驱动的 `ListMemos` 实现中，查询都包含以下 JOIN 逻辑：

**SQLite** (`store/db/sqlite/memo.go:139-142`):
```go
query := "SELECT " + strings.Join(fields, ", ") + "FROM `memo` " +
    "LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id` " +
    "LEFT JOIN `memo_relation` ON `memo`.`id` = `memo_relation`.`memo_id` AND `memo_relation`.`type` = \"COMMENT\" " +
    "LEFT JOIN `memo` AS `parent_memo` ON `memo_relation`.`related_memo_id` = `parent_memo`.`id` " +
    "WHERE " + strings.Join(where, " AND ") + " " +
    "ORDER BY " + strings.Join(orderBy, ", ")
```

**MySQL** (`store/db/mysql/memo.go:147-150`):
```go
query := "SELECT " + strings.Join(fields, ", ") + " FROM `memo`" + " " +
    "LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id`" + " " +
    "LEFT JOIN `memo_relation` ON `memo`.`id` = `memo_relation`.`memo_id` AND `memo_relation`.`type` = 'COMMENT'" + " " +
    "LEFT JOIN `memo` AS `parent_memo` ON `memo_relation`.`related_memo_id` = `parent_memo`.`id`" + " " +
```

**PostgreSQL** (`store/db/postgres/memo.go:132-136`):
```go
query := `SELECT ` + strings.Join(fields, ", ") + `
    FROM memo
    LEFT JOIN "user" AS memo_creator ON memo.creator_id = memo_creator.id
    LEFT JOIN memo_relation ON memo.id = memo_relation.memo_id AND memo_relation.type = 'COMMENT'
    LEFT JOIN memo AS parent_memo ON memo_relation.related_memo_id = parent_memo.id
    WHERE ` + strings.Join(where, " AND ") + `
```

**字段计算** (`store/db/sqlite/memo.go:133`):
```go
"CASE WHEN `parent_memo`.`uid` IS NOT NULL THEN `parent_memo`.`uid` ELSE NULL END AS `parent_uid`",
```

**查询优化**：
- `ExcludeComments: true` 时过滤评论
  - SQLite: `WHERE parent_uid IS NULL` (`store/db/sqlite/memo.go:104-106`)
  - MySQL: `HAVING parent_uid IS NULL` (`store/db/mysql/memo.go:112-114`)
  - PostgreSQL: `WHERE memo_relation.related_memo_id IS NULL` (`store/db/postgres/memo.go:97-99`)

### 3.3 评论创建流程 (`server/router/api/v1/memo_service.go:654-757`)

```
CreateMemoComment 流程（API层职责）：
1. 验证目标 memo 存在且用户有权限
2. 创建评论 memo（通过 CreateMemo）
3. 创建 COMMENT 关系：comment_memo_id → original_memo_id
4. 发送通知（收件箱 + Webhook）
5. 广播 SSE 事件
```

关键代码 (`server/router/api/v1/memo_service.go:706-714`):
```go
// 构建评论 memo 与原 memo 之间的关系
_, err = s.Store.UpsertMemoRelation(ctx, &store.MemoRelation{
    MemoID:        memo.ID,           // 评论 memo ID
    RelatedMemoID: relatedMemo.ID,    // 原 memo ID
    Type:          store.MemoRelationComment,
})
```

**职责边界**：评论关系的创建完全在 API 层 `CreateMemoComment` 中完成。

### 3.4 评论列表查询 (`server/router/api/v1/memo_service.go:759-911`)

```
ListMemoComments 流程：
1. 查询所有 COMMENT 关系（目标是当前 memo）
2. 从关系中提取所有评论 memo 的 ID
3. 批量查询这些 memo 的完整信息
4. 批量加载评论的 reactions, attachments, relations
5. 转换为 API 响应格式
```

## 4. 引用（Reference）数据组织

### 4.1 引用关系的特点

引用关系与评论关系的关键区别：

| 维度 | 评论关系 (COMMENT) | 引用关系 (REFERENCE) |
|-----|-------------------|---------------------|
| 创建方式 | `CreateMemoComment` API | `SetMemoRelations` API / `CreateMemo` 时附带 |
| 更新路径 | **只读**，不能通过 `SetMemoRelations` 修改 | 可通过 `UpdateMemo`（update_mask 含 "relations"）更新 |
| 维护策略 | 独立创建，不可批量替换 | **全量替换**：先删后插 |
| API层处理 | 专门的 `CreateMemoComment` 路径 | `setMemoRelationsInternal` 统一处理 |

### 4.2 引用设置流程 (`server/router/api/v1/memo_relation_service.go:53-91`)

```
setMemoRelationsInternal 流程（API层职责）：
1. 删除该 memo 所有现有的 REFERENCE 关系（全量替换策略）
2. 遍历请求中的关系列表
3. 忽略自引用（reflexive relations）
4. 忽略 COMMENT 类型（评论关系有单独的创建路径，只读）
5. 验证目标 memo 存在
6. 创建新的 REFERENCE 关系
```

关键代码 (`server/router/api/v1/memo_relation_service.go:54-72`):
```go
referenceType := store.MemoRelationReference
// 先删除所有引用关系（全量替换策略）
if err := s.Store.DeleteMemoRelation(ctx, &store.DeleteMemoRelation{
    MemoID: &memo.ID,
    Type:   &referenceType,
}); err != nil {
    return status.Errorf(codes.Internal, "failed to delete memo relation")
}

for _, relation := range relations {
    // 忽略自引用
    if buildMemoName(memo.UID) == relation.RelatedMemo.Name {
        continue
    }
    // 忽略 COMMENT 类型（评论关系有单独的创建路径，这里只读）
    if relation.Type == v1pb.MemoRelation_COMMENT {
        continue
    }
    // ... 创建新的 REFERENCE 关系
}
```

**设计决策**：采用"全量替换"策略，每次设置引用时先删除再重建，简化了关系管理。

## 5. 关系维护链路的职责划分

### 5.1 架构分层与职责边界

```
┌─────────────────────────────────────────────────────────────┐
│                    API Layer (server/router/api/v1/)        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  memo_service.go                                     │   │
│  │  - CreateMemo: 处理创建时的 relations 设置           │   │
│  │  - UpdateMemo: 处理更新时的 relations 修改           │   │
│  │  - CreateMemoComment: 创建评论 + COMMENT 关系       │   │
│  │  - DeleteMemo: 先级联删除评论 memo（业务逻辑）        │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  memo_relation_service.go                            │   │
│  │  - SetMemoRelations: 全量替换 REFERENCE 关系         │   │
│  │  - setMemoRelationsInternal: 核心实现，跳过 COMMENT │   │
│  │  - ListMemoRelations: 查询双向关系                   │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  memo_service_converter.go                           │   │
│  │  - batchConvertMemoRelations: 批量转换（性能优化）   │   │
│  │  - loadMemoRelations: 单 memo 关系加载               │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  Store Layer (store/)                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  memo.go                                             │   │
│  │  - DeleteMemo: 清理双向关系（不分类型）+ 附件         │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  memo_relation.go                                    │   │
│  │  - MemoRelation 结构体定义                           │   │
│  │  - UpsertMemoRelation / ListMemoRelations           │   │
│  │  - DeleteMemoRelation 接口                          │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│           Database Driver Layer (store/db/{sqlite,mysql,postgres}/) │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  memo.go                                             │   │
│  │  - ListMemos: 实时 JOIN 计算 ParentUID              │   │
│  │  - ExcludeComments: 通过 parent_uid 过滤             │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  memo_relation.go                                    │   │
│  │  - UpsertMemoRelation: SQL 层实现                    │   │
│  │  - ListMemoRelations: 多种查询条件支持               │   │
│  │  - DeleteMemoRelation: SQL 层实现                    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 各环节职责详细对比

#### 5.2.1 创建环节

**创建普通 memo 并附带引用关系** (`server/router/api/v1/memo_service.go:149-157`):
```go
if len(request.Memo.Relations) > 0 {
    _, err := s.SetMemoRelations(ctx, &v1pb.SetMemoRelationsRequest{
        Name:      fmt.Sprintf("%s%s", MemoNamePrefix, memo.UID),
        Relations: request.Memo.Relations,
    })
    // ...
}
```

**创建评论** (`server/router/api/v1/memo_service.go:654-757`):
```go
// 1. 先创建评论 memo（复用 CreateMemo）
memoComment, err := s.CreateMemo(withSuppressMentionNotifications(withSuppressSSE(ctx)), ...)

// 2. 再创建 COMMENT 关系
_, err = s.Store.UpsertMemoRelation(ctx, &store.MemoRelation{
    MemoID:        memo.ID,
    RelatedMemoID: relatedMemo.ID,
    Type:          store.MemoRelationComment,
})
```

| 关系类型 | 入口 | 处理位置 | 策略 |
|---------|------|---------|------|
| REFERENCE | `CreateMemo` → `SetMemoRelations` | API层 `memo_relation_service.go` | 全量替换 |
| COMMENT | `CreateMemoComment` | API层 `memo_service.go` | 独立创建 |

#### 5.2.2 设置/更新环节

**更新 memo 的引用关系** (`server/router/api/v1/memo_service.go:548-551`):
```go
} else if path == "relations" {
    if err := s.setMemoRelationsInternal(ctx, memo, request.Memo.Relations); err != nil {
        return nil, errors.Wrap(err, "failed to set memo relations")
    }
}
```

**注意**：
- 只有 `REFERENCE` 类型关系可以通过 `UpdateMemo`（update_mask 含 "relations"）更新
- `COMMENT` 类型关系在 `setMemoRelationsInternal` 中被**显式跳过**（只读）

#### 5.2.3 删除环节

**API 层 DeleteMemo** (`server/router/api/v1/memo_service.go:626-641`):
```go
// 第一步：业务级清理 - 先级联删除评论 memo（因为评论本身也是 memo）
commentType := store.MemoRelationComment
relations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{RelatedMemoID: &memo.ID, Type: &commentType})
for _, relation := range relations {
    if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
        // ...
    }
}

// 第二步：调用 Store 层 DeleteMemo 清理关系表和附件
if err = s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: memo.ID}); err != nil {
    // ...
}
```

**Store 层 DeleteMemo** (`store/memo.go:140-158`):
```go
func (s *Store) DeleteMemo(ctx context.Context, delete *DeleteMemo) error {
    // 清理该 memo 作为源的所有关系（不分类型：REFERENCE + COMMENT）
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    // 清理该 memo 作为目标的所有关系（不分类型：REFERENCE + COMMENT）
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{RelatedMemoID: &delete.ID}); err != nil {
        return err
    }
    // ... 清理附件
    return s.driver.DeleteMemo(ctx, delete)
}
```

**删除职责总结**：

| 层级 | 职责 | 清理内容 |
|-----|------|---------|
| API层 `DeleteMemo` | 业务逻辑 | 先级联删除评论 memo（因为评论是独立 memo） |
| Store层 `DeleteMemo` | 通用清理 | 清理该 memo 作为源或目标的**所有关系**（不分类型）+ 附件 |
| DB Driver层 | SQL 执行 | 执行具体的 DELETE 语句 |

#### 5.2.4 查询环节

**查询 memo 时的 ParentUID 计算**：
- 完全在 **DB Driver 层**通过实时 JOIN 完成
- 每次 `ListMemos` / `GetMemo` 都会 JOIN `memo_relation` 和 `parent_memo`
- 不是缓存字段，无额外维护成本，但每次查询有 JOIN 开销

**查询关系** (`store/memo_relation.go:22-35`):
```go
type FindMemoRelation struct {
    MemoID        *int32              // 精确匹配源 memo
    RelatedMemoID *int32              // 精确匹配目标 memo
    Type          *MemoRelationType   // 精确匹配类型
    MemoFilter    *string             // CEL 过滤器（用于权限检查）
    MemoIDList    []int32             // 源或目标在列表中（双向匹配）
    SourceMemoIDList    []int32       // 仅源在列表中
    RelatedMemoIDList  []int32        // 仅目标在列表中
    Limit, Offset *int                // 分页
}
```

#### 5.2.5 批量转换环节

**batchConvertMemoRelations** (`server/router/api/v1/memo_service_converter.go:171-288`):

这是**API 层**的性能优化函数，统一处理所有关系类型的批量转换：

```
batchConvertMemoRelations 流程：
1. 构建权限过滤器（visibility + creator_id）
2. 批量查询 outgoing 关系（memos 作为源，包含 REFERENCE + COMMENT）
3. 批量查询 incoming 关系（memos 作为目标，包含 REFERENCE + COMMENT）
4. 合并并去重关系
5. 批量解析关系中引用的 memo（ID→UID 转换）
6. 构建结果 map: memo_id → relations
```

关键代码 (`server/router/api/v1/memo_service_converter.go:196-209`):
```go
// 查询 outgoing 关系（不分类型）
outgoingRelations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{
    SourceMemoIDList: memoIDs,
    MemoFilter:       &memoFilter,
})
// 查询 incoming 关系（不分类型）
incomingRelations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{
    RelatedMemoIDList: memoIDs,
    MemoFilter:        &memoFilter,
})
```

**设计特点**：
- 统一处理 `REFERENCE` 和 `COMMENT` 两种关系类型
- 同时查询 outgoing 和 incoming 关系（双向）
- 权限感知：通过 `MemoFilter` 过滤不可访问的关系

## 6. 前端和 API 层

### 6.1 API 定义 (`proto/api/v1/memo_service.proto:62-87`)

```protobuf
// 设置 memo 的引用关系（全量替换，仅影响 REFERENCE 类型）
rpc SetMemoRelations(SetMemoRelationsRequest) returns (google.protobuf.Empty);

// 列出 memo 的所有关系（引用 + 评论，双向）
rpc ListMemoRelations(ListMemoRelationsRequest) returns (ListMemoRelationsResponse);

// 创建评论（特殊的关系创建路径，创建 COMMENT 类型关系）
rpc CreateMemoComment(CreateMemoCommentRequest) returns (Memo);

// 列出评论
rpc ListMemoComments(ListMemoCommentsRequest) returns (ListMemoCommentsResponse);
```

### 6.2 关系的 API 表示

MemoRelation 在 API 层包含摘要信息 (`server/router/api/v1/memo_relation_service.go:149-177`):

```go
return &v1pb.MemoRelation{
    Memo: &v1pb.MemoRelation_Memo{
        Name:    fmt.Sprintf("%s%s", MemoNamePrefix, memo.UID),
        Snippet: memoSnippet,  // 内容摘要（前64字符）
    },
    RelatedMemo: &v1pb.MemoRelation_Memo{
        Name:    fmt.Sprintf("%s%s", MemoNamePrefix, relatedMemo.UID),
        Snippet: relatedMemoSnippet,
    },
    Type: convertMemoRelationTypeFromStore(memoRelation.Type),
}
```

### 6.3 MCP 工具支持

Memos 还通过 MCP 接口暴露了关系操作 (`server/router/mcp/tools_relation.go`):
- `list_memo_relations`: 列出 memo 的关系
- `create_memo_relation`: 创建引用关系
- `delete_memo_relation`: 删除引用关系

## 7. 关键设计特点总结

### 7.1 统一关系表的优势

1. **简单性**：评论和引用使用相同的存储模型
2. **灵活性**：易于扩展新的关系类型
3. **查询一致性**：关系查询逻辑复用
4. **图遍历友好**：天然支持图结构的遍历

### 7.2 ParentUID 的实时计算

- **不是缓存字段**：每次查询通过 JOIN 实时计算
- **一致性保证**：无需额外维护，关系表变更立即可见
- **权衡**：每次查询有 JOIN 开销

### 7.3 关系类型的差异化维护

| 特性 | REFERENCE | COMMENT |
|-----|-----------|---------|
| 创建入口 | `CreateMemo` / `SetMemoRelations` | `CreateMemoComment` |
| 更新方式 | 全量替换（先删后插） | 不支持更新（只读） |
| 删除方式 | Store层自动清理 | Store层自动清理 |
| 级联删除 | 无 | 有（删除原 memo 时先删评论 memo） |
| 父关系标识 | 无 | ParentUID（实时 JOIN 计算） |

### 7.4 分层职责清晰

- **API 层**：业务逻辑（评论级联删除、权限检查、全量替换策略）
- **Store 层**：通用清理（关系表双向清理、附件清理）
- **DB Driver 层**：SQL 实现（ParentUID JOIN、查询条件构建）

### 7.5 双向查询支持

通过 `FindMemoRelation` 的多种查询方式，可以：
- 查询"我引用了谁"（outgoing）：`SourceMemoIDList`
- 查询"谁引用了我"（incoming）：`RelatedMemoIDList`
- 查询"和我相关的所有关系"（双向）：`MemoIDList`

### 7.6 权限感知

关系查询支持 `MemoFilter` 参数，确保：
- 只返回当前用户有权限访问的 memo
- 私密 memo 的关系不会泄露

### 7.7 性能优化

1. **批量加载**：`batchConvertMemoRelations` 避免 N+1
2. **组合索引**：`UNIQUE(memo_id, related_memo_id, type)` 加速查询
3. **计算字段**：`ParentUID` 实时计算，无缓存一致性问题

## 8. 关系维护流程图

### 8.1 创建评论

```
CreateMemoComment (API层)
    │
    ├──► 验证权限
    │
    ├──► CreateMemo (创建评论 memo)
    │       │
    │       └──► DB: INSERT INTO memo
    │
    ├──► UpsertMemoRelation (创建 COMMENT 关系)
    │       │
    │       └──► DB: INSERT INTO memo_relation (type='COMMENT')
    │
    ├──► 发送通知 (inbox + webhook)
    │
    └──► 广播 SSE 事件
```

### 8.2 设置引用关系

```
SetMemoRelations (API层)
    │
    └──► setMemoRelationsInternal
            │
            ├──► DeleteMemoRelation (删除所有 REFERENCE 类型关系)
            │       │
            │       └──► DB: DELETE FROM memo_relation WHERE type='REFERENCE'
            │
            └──► 遍历请求中的关系
                    │
                    ├──► 跳过 COMMENT 类型（只读）
                    │
                    └──► UpsertMemoRelation (创建新的 REFERENCE 关系)
                            │
                            └──► DB: INSERT INTO memo_relation (type='REFERENCE')
```

### 8.3 删除 Memo

```
DeleteMemo (API层)
    │
    ├──► 加载关系数据（用于 webhook）
    │
    ├──► 级联删除评论 memo（业务逻辑）
    │       │
    │       ├──► ListMemoRelations (type='COMMENT', target=当前 memo)
    │       │
    │       └──► 循环调用 Store.DeleteMemo 删除每个评论 memo
    │               │
    │               └──► (进入 Store.DeleteMemo 流程)
    │
    └──► Store.DeleteMemo (通用清理)
            │
            ├──► DeleteMemoRelation (memo_id=当前 memo)
            │       │
            │       └──► DB: DELETE FROM memo_relation WHERE memo_id=?
            │
            ├──► DeleteMemoRelation (related_memo_id=当前 memo)
            │       │
            │       └──► DB: DELETE FROM memo_relation WHERE related_memo_id=?
            │
            ├──► 删除附件
            │
            └──► DB: DELETE FROM memo WHERE id=?
```

## 9. 文件索引

| 文件路径 | 作用 | 关键内容 |
|---------|------|---------|
| `store/memo_relation.go` | MemoRelation 结构体和 Store 接口 | 关系类型定义、查询条件结构体 |
| `store/memo.go` | Memo 结构体和 DeleteMemo 级联清理 | Store层 DeleteMemo 的通用关系清理 |
| `store/db/sqlite/memo_relation.go` | SQLite 关系表 CRUD 实现 | Upsert/List/Delete 的 SQL 实现 |
| `store/db/sqlite/memo.go` | SQLite memo 查询 | ParentUID 实时 JOIN 计算、ExcludeComments 过滤 |
| `store/db/mysql/memo.go` | MySQL memo 查询 | ParentUID 实时 JOIN 计算（HAVING 过滤） |
| `store/db/postgres/memo.go` | PostgreSQL memo 查询 | ParentUID 实时 JOIN 计算 |
| `server/router/api/v1/memo_service.go` | CreateMemoComment, DeleteMemo, UpdateMemo | 评论创建、级联删除、关系更新入口 |
| `server/router/api/v1/memo_relation_service.go` | SetMemoRelations, ListMemoRelations | 引用关系全量替换、双向关系查询 |
| `server/router/api/v1/memo_service_converter.go` | batchConvertMemoRelations | 批量关系转换（性能优化） |
| `proto/api/v1/memo_service.proto` | API 协议定义 | gRPC 接口定义 |
| `store/migration/sqlite/LATEST.sql` | 数据库表结构 | memo_relation 表定义 |
| `server/router/mcp/tools_relation.go` | MCP 关系工具 | AI 助手可调用的关系操作 |

## 10. 修订记录

- **v2 (2026-05-11)**：重新审视关系维护链路
  - 修正 ParentUID 机制：明确为**实时 JOIN 计算**，非缓存字段
  - 补充各数据库驱动（SQLite/MySQL/PostgreSQL）的 ParentUID 实现对比
  - 详细梳理各环节（创建/设置/删除/查询/批量转换）的职责划分
  - 新增关系类型差异化维护对比表
  - 新增架构分层与职责边界图
  - 新增关系维护流程图
  - 修正 Store 层 DeleteMemo 的职责描述：清理**所有类型**的双向关系
