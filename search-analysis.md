# 搜索功能架构分析报告

## 一、概述

本报告详细分析了 Memos 项目中搜索功能的实现架构，包括索引维护机制、结果排序策略以及按用户权限裁剪搜索结果的多层协作机制。

---

## 二、架构层次总览

搜索功能的实现跨越了以下几个主要层次：

1. **前端层**：React 18 + TypeScript，通过 React Query 管理搜索状态和缓存
2. **API 服务层**：Go Echo 框架，处理请求参数验证、权限检查和排序解析
3. **过滤器引擎层**：基于 CEL (Common Expression Language) 的表达式编译与 SQL 生成
4. **数据存储层**：SQLite / MySQL / PostgreSQL 三数据库驱动

---

## 三、索引维护机制

### 3.1 数据库索引策略

根据数据库迁移文件分析，系统经历了索引策略的演变：

#### 3.1.1 历史索引（v0.14）

在 `store/migration/sqlite/0.14/01__create_indexes.sql` 中创建了以下索引：

```sql
CREATE INDEX IF NOT EXISTS idx_user_username ON user (username);
CREATE INDEX IF NOT EXISTS idx_memo_creator_id ON memo (creator_id);
CREATE INDEX IF NOT EXISTS idx_memo_content ON memo (content);
CREATE INDEX IF NOT EXISTS idx_memo_visibility ON memo (visibility);
CREATE INDEX IF NOT EXISTS idx_resource_creator_id ON resource (creator_id);
```

#### 3.1.2 索引优化（v0.26）

在 `store/migration/sqlite/0.26/02__drop_indexes.sql` 中删除了部分索引：

```sql
DROP INDEX IF EXISTS idx_user_username;
DROP INDEX IF EXISTS idx_memo_creator_id;
DROP INDEX IF EXISTS idx_attachment_creator_id;
DROP INDEX IF EXISTS idx_attachment_memo_id;
```

**优化原因分析**：
- 系统转向基于 CEL 过滤器的动态查询模式
- 避免过多索引导致的写入性能下降
- 依赖数据库查询优化器自动选择合适的执行计划

#### 3.1.3 当前保留的索引

在 `LATEST.sql` 中保留的索引：

```sql
-- memo_share 表
CREATE INDEX idx_memo_share_memo_id ON memo_share(memo_id);

-- user_identity 表  
CREATE INDEX idx_user_identity_user_id ON user_identity(user_id);

-- memo 表
uid TEXT NOT NULL UNIQUE  -- 唯一索引
```

### 3.2 索引维护的触发时机

索引维护由数据库引擎自动处理，触发于以下数据操作：

| 操作 | 触发位置 | 代码文件 |
|------|---------|---------|
| 创建 memo | `CreateMemo` | `store/db/{driver}/memo.go:16-52` |
| 更新 memo | `UpdateMemo` | `store/db/{driver}/memo.go:195-235` |
| 删除 memo | `DeleteMemo` | `store/db/{driver}/memo.go:237-248` |

---

## 四、搜索请求处理流程

### 4.1 前端搜索状态管理

#### 4.1.1 MemoFilterContext

位于 `web/src/contexts/MemoFilterContext.tsx`，提供以下功能：

- **URL 同步**：将过滤器状态与 URL 查询参数双向同步
- **过滤因子**：支持 `tagSearch`、`visibility`、`contentSearch`、`displayTime`、`pinned`、`property.*` 等
- **快捷方式管理**：支持保存和使用预定义的过滤器组合

```typescript
// 过滤器数据结构
export interface MemoFilter {
  factor: FilterFactor;  // 过滤因子类型
  value: string;          // 过滤值
}
```

#### 4.1.2 React Query 缓存策略

位于 `web/src/hooks/useMemoQueries.ts`：

```typescript
// 查询键工厂
export const memoKeys = {
  all: ["memos"] as const,
  lists: () => [...memoKeys.all, "list"] as const,
  list: (filters: Partial<ListMemosRequest>) => [...memoKeys.lists(), filters] as const,
  // ...
};

// 无限滚动查询
export function useInfiniteMemos(request: Partial<ListMemosRequest> = {}, options?: { enabled?: boolean }) {
  return useInfiniteQuery({
    queryKey: memoKeys.list(request),
    staleTime: 1000 * 60,      // 60 秒新鲜期
    gcTime: 1000 * 60 * 5,      // 5 分钟缓存期
    // ...
  });
}
```

### 4.2 API 层参数处理

#### 4.2.1 ListMemos 入口

位于 `server/router/api/v1/memo_service.go:189-353`，核心流程：

1. **状态过滤**：根据 `request.State` 区分普通/归档状态
2. **排序解析**：解析 `request.OrderBy` 参数
3. **过滤器验证**：验证并附加 CEL 过滤器
4. **权限裁剪**：根据当前用户身份注入权限条件
5. **分页处理**：解析 `pageToken` 或 `pageSize`

#### 4.2.2 排序解析

位于 `server/router/api/v1/memo_service.go:1000-1056`：

```go
func (*APIV1Service) parseMemoOrderBy(orderBy string, memoFind *store.FindMemo) error {
    fields := strings.Split(orderBy, ",")  // 支持多字段排序
    
    for _, field := range fields {
        switch fieldName {
        case "pinned":
            memoFind.OrderByPinned = true  // 置顶优先
        case "create_time", "name":
            memoFind.OrderByTimeAsc = fieldDirection == "asc"
        case "update_time":
            memoFind.OrderByUpdatedTs = true
            memoFind.OrderByTimeAsc = fieldDirection == "asc"
        }
    }
    // ...
}
```

**排序优先级**：
1. 优先按 `pinned` 降序（置顶在前）
2. 然后按 `created_ts` 或 `updated_ts` 排序
3. 最后按 `id` 降序作为 tie-breaker（SQLite:122）

---

## 五、CEL 过滤器引擎

### 5.1 过滤器架构

位于 `internal/filter/` 目录，核心组件：

| 组件 | 文件 | 功能 |
|------|------|------|
| Engine | `engine.go` | CEL 环境管理、表达式编译 |
| Schema | `schema.go` | 字段定义、类型映射 |
| Renderer | `render.go` | 将条件树渲染为 SQL |
| Helpers | `helpers.go` | 便捷函数（AppendConditions） |

### 5.2 支持的过滤字段

在 `schema.go:101-266` 中定义：

| 字段名 | 类型 | 存储位置 | 说明 |
|--------|------|---------|------|
| `content` | string | memo.content | 支持 contains 匹配 |
| `creator` | string | user.username | 格式为 `users/{username}` |
| `creator_id` | int | memo.creator_id | 精确匹配 |
| `created_ts` | timestamp | memo.created_ts | 时间戳比较 |
| `updated_ts` | timestamp | memo.updated_ts | 时间戳比较 |
| `pinned` | bool | memo.pinned | 置顶状态 |
| `visibility` | string | memo.visibility | PUBLIC/PROTECTED/PRIVATE |
| `tags` | list | memo.payload.tags | JSON 列表 |
| `has_task_list` | bool | memo.payload.property | 任务列表属性 |
| `has_link` | bool | memo.payload.property | 链接属性 |
| `has_code` | bool | memo.payload.property | 代码块属性 |
| `has_incomplete_tasks` | bool | memo.payload.property | 未完成任务 |

### 5.3 编译与渲染流程

```
用户过滤器字符串
       ↓
   CEL 编译 (env.Compile)
       ↓
   AST → Condition Tree (buildCondition)
       ↓
   方言感知渲染 (Renderer)
       ↓
   SQL 片段 + 参数
```

关键代码（`engine.go:67-74`）：

```go
func (e *Engine) CompileToStatement(ctx context.Context, filter string, opts RenderOptions) (Statement, error) {
    program, err := e.Compile(ctx, filter)
    if err != nil {
        return Statement{}, err
    }
    return program.Render(opts)
}
```

---

## 六、用户权限裁剪机制

### 6.1 权限模型

系统定义了三种可见性级别（`store/memo.go:12-22`）：

```go
const (
    Public    Visibility = "PUBLIC"     // 所有人可见
    Protected Visibility = "PROTECTED"  // 登录用户可见
    Private   Visibility = "PRIVATE"    // 仅创建者可见
)
```

### 6.2 API 层权限注入

位于 `memo_service.go:229-238`，这是核心的权限裁剪逻辑：

```go
if currentUser == nil {
    // 未登录用户：仅可见 PUBLIC
    memoFind.VisibilityList = []store.Visibility{store.Public}
} else {
    if memoFind.CreatorID == nil {
        // 未指定创建者：自己的 + 公开的 + 受保护的
        filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
        memoFind.Filters = append(memoFind.Filters, filter)
    } else if *memoFind.CreatorID != currentUser.ID {
        // 指定了其他创建者：仅可见 PUBLIC 和 PROTECTED
        memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
    }
}
```

**权限矩阵**：

| 用户状态 | 查询范围 | 可见的 memo |
|---------|---------|------------|
| 未登录 | 全部 | 仅 `PUBLIC` |
| 已登录 | 全部 | `creator_id == currentUser` 或 `visibility in [PUBLIC, PROTECTED]` |
| 已登录 | 指定创建者 = 自己 | 全部（无额外限制） |
| 已登录 | 指定创建者 ≠ 自己 | 仅 `PUBLIC` 和 `PROTECTED` |

### 6.3 归档状态的特殊处理

位于 `memo_service.go:199-210`：

```go
if request.State == v1pb.State_ARCHIVED {
    memoFind.RowStatus = &state
    // 归档的 memo 仅创建者可见
    if currentUser == nil {
        return &v1pb.ListMemosResponse{}, nil  // 直接返回空
    }
    memoFind.CreatorID = &currentUser.ID
}
```

### 6.4 单条 memo 的权限检查

位于 `memo_service.go:42-71`，用于 `GetMemo` 等操作：

```go
func (s *APIV1Service) checkMemoReadAccess(ctx context.Context, memo *store.Memo) error {
    // 归档状态：仅创建者可见
    if memo.RowStatus == store.Archived {
        user, err := s.fetchCurrentUser(ctx)
        if user == nil || memo.CreatorID != user.ID {
            return status.Errorf(codes.NotFound, "memo not found")
        }
    }

    // 非 PUBLIC：需要登录
    if memo.Visibility != store.Public {
        user, err := s.fetchCurrentUser(ctx)
        if user == nil {
            return status.Errorf(codes.Unauthenticated, "user not authenticated")
        }
        // PRIVATE：仅创建者可见
        if memo.Visibility == store.Private && memo.CreatorID != user.ID {
            return status.Errorf(codes.PermissionDenied, "permission denied")
        }
    }
    return nil
}
```

---

## 七、数据库层查询构建

### 7.1 SQLite 实现

位于 `store/db/sqlite/memo.go:54-193`：

```go
func (d *DB) ListMemos(ctx context.Context, find *store.FindMemo) ([]*store.Memo, error) {
    where, args := []string{"1 = 1"}, []any{}

    // 1. 应用 CEL 过滤器
    engine, err := filter.DefaultEngine()
    if err := filter.AppendConditions(ctx, engine, find.Filters, filter.DialectSQLite, &where, &args); err != nil {
        return nil, err
    }

    // 2. 应用结构化条件（ID、UID、CreatorID 等）
    if v := find.ID; v != nil {
        where, args = append(where, "`memo`.`id` = ?"), append(args, *v)
    }
    // ...

    // 3. 构建 ORDER BY
    orderBy := []string{}
    if find.OrderByPinned {
        orderBy = append(orderBy, "`pinned` DESC")
    }
    if find.OrderByUpdatedTs {
        orderBy = append(orderBy, "`updated_ts` "+order)
    } else {
        orderBy = append(orderBy, "`created_ts` "+order)
    }
    orderBy = append(orderBy, "`id` DESC")  // tie-breaker

    // 4. 构建查询
    query := "SELECT ... FROM `memo` " +
        "LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id` " +
        "WHERE " + strings.Join(where, " AND ") + " " +
        "ORDER BY " + strings.Join(orderBy, ", ")
}
```

### 7.2 三数据库驱动对比

| 特性 | SQLite | MySQL | PostgreSQL |
|------|--------|-------|-----------|
| 占位符 | `?` | `?` | `$1, $2...` |
| 时间戳 | BIGINT (epoch) | TIMESTAMP (需 UNIX_TIMESTAMP) | BIGINT (epoch) |
| 标识符 | `` `id` `` | `` `id` `` | `"id"` |
| JSON 路径 | `json_extract` | `JSON_EXTRACT` | `json_extract_path_text` |
| 字符串拼接 | `||` | `CONCAT()` | `||` |

数据库驱动通过 `filter.DialectName` 告知渲染器生成正确的 SQL。

---

## 八、跨层次协作全景图

### 8.1 完整搜索流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        前端层 (React + TypeScript)                       │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  MemoFilterContext                                                  │ │
│  │  - 解析 URL 中的 filter 参数                                        │ │
│  │  - 维护过滤器状态                                                   │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────────┐ │
│  │  useInfiniteMemos (React Query)                                    │ │
│  │  - 构建 ListMemosRequest                                            │ │
│  │  - 管理分页和缓存                                                   │ │
│  │  queryKey: memoKeys.list({filter, orderBy, pageSize})              │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────────────┘
                                │
                                ▼ gRPC / Connect RPC
┌─────────────────────────────────────────────────────────────────────────┐
│                        API 层 (Go + Echo)                               │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  ListMemos (memo_service.go:189)                                   │ │
│  │  1. 解析 request.State → RowStatus                                 │ │
│  │  2. parseMemoOrderBy() → OrderByPinned, OrderByUpdatedTs, etc.     │ │
│  │  3. validateFilter() → 验证 CEL 语法                               │ │
│  │  4. 权限注入 (Lines 229-238):                                      │ │
│  │     - 未登录: VisibilityList = [PUBLIC]                            │ │
│  │     - 已登录: Filters += "creator_id == X || visibility in [...]"  │ │
│  │  5. 分页参数处理                                                   │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────────┐ │
│  │  store.FindMemo                                                     │ │
│  │  {                                                                 │ │
│  │    RowStatus: *NORMAL,                                             │ │
│  │    Filters: [userFilter, "content.contains('test')"],              │ │
│  │    VisibilityList: [PUBLIC] (if unauthenticated),                  │ │
│  │    OrderByPinned: true,                                            │ │
│  │    OrderByUpdatedTs: false,                                        │ │
│  │    Limit: &10,                                                     │ │
│  │    Offset: &0                                                      │ │
│  │  }                                                                 │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        存储层 (Store + Driver)                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Store.ListMemos (store/memo.go:116)                               │ │
│  │  → s.driver.ListMemos(ctx, find)                                   │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────────┐ │
│  │  SQLite/MySQL/PostgreSQL Driver                                    │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │  filter.AppendConditions()                                   │  │ │
│  │  │  - 遍历 Filters: ["creator_id == 1 || ...", "content..."]   │  │ │
│  │  │  - 为每个 filter 创建 Engine                                  │  │ │
│  │  │  - CompileToStatement(dialect=SQLite|MySQL|Postgres)         │  │ │
│  │  │    → CEL 编译 → AST → 条件树 → SQL 片段 + args               │  │ │
│  │  │  - 追加到 where 子句                                         │  │ │
│  │  └──────────────────────────────┬───────────────────────────────┘  │ │
│  │                                 │                                   │ │
│  │  ┌──────────────────────────────▼───────────────────────────────┐  │ │
│  │  │  构建 SQL 查询                                               │  │ │
│  │  │  SELECT id, uid, creator_id, ...                             │  │ │
│  │  │  FROM memo                                                   │  │ │
│  │  │  LEFT JOIN user AS memo_creator ON ...                       │  │ │
│  │  │  WHERE 1 = 1                                                 │  │ │
│  │  │    AND (memo.creator_id = ? OR memo.visibility IN (?, ?))    │  │ │
│  │  │    AND memo.content LIKE ?                                   │  │ │
│  │  │    AND memo.row_status = ?                                   │  │ │
│  │  │  ORDER BY pinned DESC, created_ts DESC, id DESC              │  │ │
│  │  │  LIMIT ? OFFSET ?                                            │  │ │
│  │  └──────────────────────────────┬───────────────────────────────┘  │ │
│  └─────────────────────────────────┼──────────────────────────────────┘ │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │
                                     ▼
                          ┌─────────────────┐
                          │   数据库引擎     │
                          │  (索引优化)      │
                          └─────────────────┘
```

### 8.2 数据流详解

#### 阶段 1：前端参数构建

```
用户操作（点击标签、输入搜索词）
    ↓
MemoFilterContext 更新 filters 数组
    ↓
useInfiniteMemos 检测到 queryKey 变化
    ↓
构建 ListMemosRequest {
  Filter: "tag == 'work' && content.contains('meeting')",
  OrderBy: "pinned, create_time desc",
  PageSize: 20
}
```

#### 阶段 2：API 层处理

```
ListMemos 接收请求
    ↓
parseMemoOrderBy("pinned, create_time desc")
    → OrderByPinned = true, OrderByTimeAsc = false
    ↓
validateFilter("tag == 'work' && content.contains('meeting')")
    → CEL 编译检查通过
    ↓
权限判断（假设用户 ID=123）：
    → Filters += "creator_id == 123 || visibility in ['PUBLIC', 'PROTECTED']"
    ↓
FindMemo {
  Filters: [
    "tag == 'work' && content.contains('meeting')",
    "creator_id == 123 || visibility in ['PUBLIC', 'PROTECTED']"
  ],
  OrderByPinned: true,
  OrderByTimeAsc: false,
  Limit: &21,  // limit+1 用于判断是否有下一页
  Offset: &0
}
```

#### 阶段 3：过滤器编译

```
filter.AppendConditions(ctx, engine, filters, DialectSQLite, &where, &args)
    ↓
filter1: "tag == 'work' && content.contains('meeting')"
    ↓
CEL 编译 → AST
    ↓
条件树：
  AND
  ├── (memo.payload.tags 包含 'work')
  └── (memo.content LIKE '%meeting%')
    ↓
SQL 渲染（SQLite）：
  "json_extract(`memo`.`payload`, '$.tags') LIKE ? AND `memo`.`content` LIKE ?"
  args: ['%work%', '%meeting%']
    ↓
filter2: "creator_id == 123 || visibility in ['PUBLIC', 'PROTECTED']"
    ↓
SQL 渲染：
  "(`memo`.`creator_id` = ? OR `memo`.`visibility` IN (?, ?))"
  args: [123, 'PUBLIC', 'PROTECTED']
```

#### 阶段 4：数据库查询

```
最终 WHERE 子句：
WHERE 1 = 1
  AND (json_extract(`memo`.`payload`, '$.tags') LIKE '%work%' 
       AND `memo`.`content` LIKE '%meeting%')
  AND (`memo`.`creator_id` = 123 
       OR `memo`.`visibility` IN ('PUBLIC', 'PROTECTED'))
  AND `memo`.`row_status` = 'NORMAL'
  AND `parent_uid` IS NULL  -- ExcludeComments

ORDER BY：
ORDER BY `pinned` DESC, `created_ts` DESC, `id` DESC

LIMIT / OFFSET：
LIMIT 21 OFFSET 0
```

---

## 九、关键设计决策

### 9.1 为什么使用 CEL 而不是 ORM 查询构建器？

**优势**：
1. **灵活性**：用户可以编写复杂的组合查询，如 `tag == 'work' && created_ts > now() - 86400`
2. **安全性**：CEL 编译层提供语法验证，避免 SQL 注入
3. **可扩展性**：新增过滤字段只需更新 Schema 定义
4. **跨数据库**：同一表达式可渲染为 SQLite/MySQL/PostgreSQL 语法

**代价**：
- 运行时编译开销（通过 Engine 单例和缓存缓解）
- 调试难度增加（需要理解 CEL → SQL 的转换）

### 9.2 为什么在 API 层注入权限条件而不是在数据库层？

**优势**：
1. **关注点分离**：数据库层专注于数据访问，API 层负责业务规则
2. **复用性**：权限逻辑集中在一处，多个 API 端点共享
3. **可测试性**：可以独立测试权限注入逻辑
4. **透明性**：权限条件作为普通 Filter 注入，与用户过滤器统一处理

**实现方式**：将权限条件包装为 CEL 表达式，与用户过滤器一起进入编译和渲染流程。

### 9.3 为什么采用两层过滤（VisibilityList + CEL Filter）？

查看代码 `memo_service.go:229-238`：

```go
if currentUser == nil {
    memoFind.VisibilityList = []store.Visibility{store.Public}
} else {
    if memoFind.CreatorID == nil {
        filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
        memoFind.Filters = append(memoFind.Filters, filter)
    } else if *memoFind.CreatorID != currentUser.ID {
        memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
    }
}
```

**设计原因**：
- `VisibilityList` 使用 SQL `IN` 子句，数据库可利用索引（如果存在）
- CEL Filter 支持更复杂的逻辑（OR 组合）
- 两种方式结合，兼顾性能和灵活性

---

## 十、性能考虑

### 10.1 查询性能

| 潜在瓶颈 | 位置 | 缓解措施 |
|---------|------|---------|
| CEL 编译 | filter 层 | Engine 单例（sync.Once） |
| 内容搜索 | `content.contains()` | 使用 `LIKE '%text%'`（全表扫描） |
| JSON 字段查询 | payload.property | `json_extract` 无法使用索引 |
| 多表 JOIN | 列表查询 | LEFT JOIN user, memo_relation, memo |
| 分页 | 大 Offset | 使用 keyset 分页可优化（当前用 offset） |

### 10.2 缓存策略

**前端缓存（React Query）**：
- `staleTime: 60s` — 60 秒内认为数据新鲜
- `gcTime: 5min` — 5 分钟后清理未使用的缓存
- 按查询参数（filter, orderBy 等）生成唯一 queryKey

**后端缓存**：
- Store 层有内存缓存（TTL 10min, max 1000）
- 但 ListMemos 等查询可能不经过缓存

---

## 十一、代码位置索引

| 功能 | 文件路径 | 关键行号 |
|------|---------|---------|
| 前端过滤器上下文 | `web/src/contexts/MemoFilterContext.tsx` | 全部 |
| 前端查询 hooks | `web/src/hooks/useMemoQueries.ts` | 111-139 |
| ListMemos API | `server/router/api/v1/memo_service.go` | 189-353 |
| 权限注入逻辑 | `server/router/api/v1/memo_service.go` | 229-238 |
| 单条 memo 权限检查 | `server/router/api/v1/memo_service.go` | 42-71 |
| 排序解析 | `server/router/api/v1/memo_service.go` | 1000-1056 |
| 过滤器验证 | `server/router/api/v1/shortcut_service.go` | 336-360 |
| CEL 引擎核心 | `internal/filter/engine.go` | 全部 |
| 过滤字段 Schema | `internal/filter/schema.go` | 100-266 |
| 过滤器辅助函数 | `internal/filter/helpers.go` | 8-25 |
| Store 层接口 | `store/memo.go` | 109-159 |
| SQLite 驱动实现 | `store/db/sqlite/memo.go` | 54-193 |
| MySQL 驱动实现 | `store/db/mysql/memo.go` | 62-150 |
| PostgreSQL 驱动实现 | `store/db/postgres/memo.go` | 51-150 |
| 可见性定义 | `store/memo.go` | 12-22 |
| 数据库迁移 | `store/migration/sqlite/LATEST.sql` | 全部 |

---

## 十二、总结

Memos 的搜索功能采用了**分层协作**的架构设计：

1. **前端层**：通过 MemoFilterContext 管理搜索状态，React Query 处理缓存和分页
2. **API 层**：解析参数、验证过滤器、**注入权限条件**、解析排序规则
3. **过滤器层**：CEL 引擎将表达式编译为条件树，再根据数据库方言渲染为 SQL
4. **存储层**：将所有条件（用户过滤器 + 权限过滤器 + 结构化条件）组合为最终查询

**权限裁剪是整个架构的核心亮点**：它不是在查询后过滤结果，而是在查询构建阶段就将权限条件注入到 SQL 中，保证了性能和安全性。权限条件通过 CEL 表达式的形式与用户过滤器无缝融合，由同一套编译和渲染流程处理。

**排序策略**采用了多优先级排序：置顶优先 → 时间排序 → ID 作为 tie-breaker，确保结果的一致性和可预测性。

**索引维护**由数据库自动处理，系统经历了从"创建大量索引"到"精简索引"的优化过程，依赖查询优化器自动选择最优执行计划。
