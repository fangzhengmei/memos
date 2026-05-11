# Memos 内容关系分析报告

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

### 3.2 评论创建流程 (`server/router/api/v1/memo_service.go:654-757`)

```
CreateMemoComment 流程：
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

### 3.3 ParentUID 机制

Memos 还使用了一个**计算字段** `ParentUID` 来快速标识评论的父笔记，避免每次都 JOIN 查询。

在 SQLite 实现中 (`store/db/sqlite/memo.go:139-142`):
```go
query := "SELECT " + strings.Join(fields, ", ") + "FROM `memo` " +
    "LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id` " +
    "LEFT JOIN `memo_relation` ON `memo`.`id` = `memo_relation`.`memo_id` AND `memo_relation`.`type` = \"COMMENT\" " +
    "LEFT JOIN `memo` AS `parent_memo` ON `memo_relation`.`related_memo_id` = `parent_memo`.`id` " +
    "WHERE " + strings.Join(where, " AND ") + " " +
    "ORDER BY " + strings.Join(orderBy, ", ")
```

**查询优化**：
- 列表查询时通过 `ExcludeComments` 标志位过滤评论 (`store/db/sqlite/memo.go:104-106`)
- `ExcludeComments: true` 时添加条件 `parent_uid IS NULL`

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

引用关系与评论关系的区别：
- **创建方式**：引用通常通过 `SetMemoRelations` API 设置，评论通过 `CreateMemoComment` 创建
- **编辑权限**：引用关系可以通过更新 memo 来修改，评论关系在创建后独立维护
- **方向语义**：两者都是单向关系，但使用场景不同

### 4.2 引用设置流程 (`server/router/api/v1/memo_relation_service.go:53-91`)

```
setMemoRelationsInternal 流程：
1. 删除该 memo 所有现有的 REFERENCE 关系
2. 遍历请求中的关系列表
3. 忽略自引用（reflexive relations）
4. 忽略 COMMENT 类型（评论关系有单独的创建路径）
5. 验证目标 memo 存在
6. 创建新的 REFERENCE 关系
```

关键代码 (`server/router/api/v1/memo_relation_service.go:54-61`):
```go
referenceType := store.MemoRelationReference
// 先删除所有引用关系
if err := s.Store.DeleteMemoRelation(ctx, &store.DeleteMemoRelation{
    MemoID: &memo.ID,
    Type:   &referenceType,
}); err != nil {
    return status.Errorf(codes.Internal, "failed to delete memo relation")
}
```

**设计决策**：采用"全量替换"策略，每次设置引用时先删除再重建，简化了关系管理。

## 5. 内容图谱关系的维护位置

### 5.1 架构分层

Memos 的关系维护采用了清晰的分层架构：

```
┌─────────────────────────────────────────────────┐
│                 API Layer                       │
│  server/router/api/v1/                          │
│  - memo_service.go (CreateMemoComment, etc.)    │
│  - memo_relation_service.go (SetMemoRelations)  │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│                 Store Layer                     │
│  store/                                         │
│  - memo_relation.go (MemoRelation 定义)         │
│  - memo.go (DeleteMemo 中的级联清理)            │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│              Database Driver Layer              │
│  store/db/{sqlite,mysql,postgres}/              │
│  - memo_relation.go (CRUD 实现)                 │
└─────────────────────────────────────────────────┘
```

### 5.2 级联删除机制

删除 memo 时会自动清理相关关系 (`store/memo.go:140-158`):

```go
func (s *Store) DeleteMemo(ctx context.Context, delete *DeleteMemo) error {
    // 清理该 memo 作为源的所有关系
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{MemoID: &delete.ID}); err != nil {
        return err
    }
    // 清理该 memo 作为目标的所有关系
    if err := s.driver.DeleteMemoRelation(ctx, &DeleteMemoRelation{RelatedMemoID: &delete.ID}); err != nil {
        return err
    }
    // ... 清理附件
    return s.driver.DeleteMemo(ctx, delete)
}
```

删除包含评论的 memo 时 (`server/router/api/v1/memo_service.go:626-636`):
```go
// 先删除所有评论（store.DeleteMemo 会处理它们的关系和附件）
commentType := store.MemoRelationComment
relations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{RelatedMemoID: &memo.ID, Type: &commentType})
for _, relation := range relations {
    if err := s.Store.DeleteMemo(ctx, &store.DeleteMemo{ID: relation.MemoID}); err != nil {
        return nil, status.Errorf(codes.Internal, "failed to delete memo comment")
    }
}
```

### 5.3 批处理优化

为避免 N+1 查询，Memos 实现了批量关系加载 (`server/router/api/v1/memo_service_converter.go:171-288`):

```
batchConvertMemoRelations 流程：
1. 构建权限过滤器（visibility + creator_id）
2. 批量查询 outgoing 关系（memos 作为源）
3. 批量查询 incoming 关系（memos 作为目标）
4. 合并并去重关系
5. 批量解析关系中引用的 memo（ID→UID 转换）
6. 构建结果 map: memo_id → relations
```

关键优化点：
- 使用 `SourceMemoIDList` 和 `RelatedMemoIDList` 进行批量 IN 查询
- 合并 outgoing 和 incoming 关系后统一解析
- 权限过滤通过 CEL 表达式编译成 SQL 子查询

### 5.4 查询接口

`FindMemoRelation` 提供了多种查询方式 (`store/memo_relation.go:22-35`):

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

在 SQLite 实现中 (`store/db/sqlite/memo_relation.go:52-63`):
```go
if len(find.MemoIDList) > 0 {
    // memo_id IN (...) OR related_memo_id IN (...)
    where = append(where, fmt.Sprintf("(memo_id IN (%s) OR related_memo_id IN (%s))", inClause, inClause))
}
```

## 6. 前端和 API 层

### 6.1 API 定义 (`proto/api/v1/memo_service.proto:62-87`)

```protobuf
// 设置 memo 的引用关系
rpc SetMemoRelations(SetMemoRelationsRequest) returns (google.protobuf.Empty);

// 列出 memo 的所有关系（引用 + 评论）
rpc ListMemoRelations(ListMemoRelationsRequest) returns (ListMemoRelationsResponse);

// 创建评论（特殊的关系创建路径）
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

### 7.2 双向查询支持

通过 `FindMemoRelation` 的多种查询方式，可以：
- 查询"我引用了谁"（outgoing）
- 查询"谁引用了我"（incoming）
- 查询"和我相关的所有关系"（双向）

### 7.3 权限感知

关系查询支持 `MemoFilter` 参数，确保：
- 只返回当前用户有权限访问的 memo
- 私密 memo 的关系不会泄露

### 7.4 性能优化

1. **批量加载**：`batchConvertMemoRelations` 避免 N+1
2. **组合索引**：`UNIQUE(memo_id, related_memo_id, type)` 加速查询
3. **计算字段**：`ParentUID` 简化评论识别

## 8. 文件索引

| 文件路径 | 作用 |
|---------|------|
| `store/memo_relation.go` | MemoRelation 结构体和 Store 接口 |
| `store/memo.go` | Memo 结构体和 DeleteMemo 级联清理 |
| `store/db/sqlite/memo_relation.go` | SQLite 关系表 CRUD 实现 |
| `store/db/sqlite/memo.go` | SQLite memo 查询（含 ParentUID JOIN） |
| `server/router/api/v1/memo_service.go` | CreateMemoComment, ListMemoComments |
| `server/router/api/v1/memo_relation_service.go` | SetMemoRelations, ListMemoRelations |
| `server/router/api/v1/memo_service_converter.go` | batchConvertMemoRelations 批量优化 |
| `proto/api/v1/memo_service.proto` | API 协议定义 |
| `store/migration/sqlite/LATEST.sql` | 数据库表结构 |
| `server/router/mcp/tools_relation.go` | MCP 关系工具 |
