# 搜索功能架构分析报告

## 一、概述

本报告详细分析了 Memos 项目中搜索功能的实现架构，包括索引维护机制、结果排序策略、按用户权限裁剪搜索结果的多层协作机制，以及索引在查询执行过程中的参与方式和退化场景。

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

#### 3.1.2 索引删除历史

系统在两个版本中删除了索引：

**第一阶段（v0.24）**：在 `store/migration/sqlite/0.24/00__memo.sql` 中删除了以下索引：

```sql
DROP INDEX IF EXISTS idx_memo_tags;
DROP INDEX IF EXISTS idx_memo_content;
DROP INDEX IF EXISTS idx_memo_visibility;
```

伴随 `ALTER TABLE memo DROP COLUMN tags;`（tags 列被废弃）

**第二阶段（v0.26）**：在 `store/migration/sqlite/0.26/02__drop_indexes.sql` 中删除了以下索引：

```sql
DROP INDEX IF EXISTS idx_user_username;
DROP INDEX IF EXISTS idx_memo_creator_id;
DROP INDEX IF EXISTS idx_attachment_creator_id;
DROP INDEX IF EXISTS idx_attachment_memo_id;
```

**优化原因分析**：
- 系统转向基于 CEL 过滤器的动态查询模式
- `content.contains()` 使用 `LIKE '%text%'` 导致索引无法使用
- `payload` 字段查询依赖 `json_extract()` 函数
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

## 四、索引在查询执行中的参与分析

### 4.1 当前索引状态

根据 `LATEST.sql`，`memo` 表当前仅有以下索引：

| 索引类型 | 字段 | 用途 |
|---------|------|------|
| 主键索引 | `id` | 主键查询 |
| 唯一索引 | `uid` | 按 UID 查询 |

**索引删除历史（两个阶段）**：

| 删除版本 | 索引名称 | 字段 | 删除原因 |
|---------|---------|------|---------|
| v0.24 | `idx_memo_tags` | `tags` | tags 列被废弃，同时删除列和索引 |
| v0.24 | `idx_memo_content` | `content` | `content.contains()` 使用 `LIKE '%text%'` 无法使用索引 |
| v0.24 | `idx_memo_visibility` | `visibility` | 与 `OR` 条件组合时索引选择性降低 |
| v0.26 | `idx_memo_creator_id` | `creator_id` | 权限裁剪使用 `OR` 组合，优化器可能不选择 |
| v0.26 | `idx_attachment_creator_id` | `attachment.creator_id` | 减少索引维护开销 |
| v0.26 | `idx_attachment_memo_id` | `attachment.memo_id` | 减少索引维护开销 |
| v0.26 | `idx_user_username` | `user.username` | 已有唯一约束（UNIQUE），无需额外索引 |

### 4.2 查询执行路径中的索引使用

让我们分析一个典型的搜索请求从前端到数据库的完整执行路径，以及索引在每个阶段的参与情况：

#### 阶段 1：前端参数构建

```typescript
// 用户搜索：标签为 'work' 且内容包含 'meeting'
const filter = "tag == 'work' && content.contains('meeting')";
const orderBy = "pinned, create_time desc";
```

#### 阶段 2：API 层处理（权限注入）

```go
// memo_service.go:229-238
if currentUser != nil && memoFind.CreatorID == nil {
    // 注入权限条件
    filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
    memoFind.Filters = append(memoFind.Filters, filter)
}
```

此时 `FindMemo` 结构为：
```go
FindMemo {
    Filters: [
        "tag == 'work' && content.contains('meeting')",        // 用户过滤器
        "creator_id == 123 || visibility in ['PUBLIC', 'PROTECTED']"  // 权限过滤器
    ],
    OrderByPinned: true,
    OrderByTimeAsc: false,  // DESC
    // ...
}
```

#### 阶段 3：CEL 编译与 SQL 渲染（关键分析点）

**用户过滤器渲染**：
```go
// render.go:449-468 - contains 条件
func (r *renderer) renderContainsCondition(cond *ContainsCondition) (renderResult, error) {
    arg := fmt.Sprintf("%%%s%%", cond.Value)  // 前后加 %
    // SQLite: memos_unicode_lower(column) LIKE memos_unicode_lower(?)
    // PostgreSQL: column ILIKE ?
    // MySQL: column LIKE ?
}

// render.go:398-415 - tag 包含条件（JSON 列表）
case DialectSQLite:
    sql := fmt.Sprintf("%s LIKE %s", jsonArrayExpr(r.dialect, field), r.addArg(fmt.Sprintf(`%%"%s"%%`, str)))
    // json_extract(payload, '$.tags') LIKE '%"work"%'
```

**权限过滤器渲染**：
```go
// creator_id == 123 → `memo`.`creator_id` = ?
// visibility in ['PUBLIC', 'PROTECTED'] → `memo`.`visibility` IN (?, ?)
// OR 组合 → (`memo`.`creator_id` = ? OR `memo`.`visibility` IN (?, ?))
```

#### 阶段 4：最终 SQL 构建（数据库驱动层）

**SQLite 实现** (`store/db/sqlite/memo.go:139-144`)：
```sql
SELECT ... FROM `memo`
LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id`
LEFT JOIN `memo_relation` ON `memo`.`id` = `memo_relation`.`memo_id` AND `memo_relation`.`type` = "COMMENT"
LEFT JOIN `memo` AS `parent_memo` ON `memo_relation`.`related_memo_id` = `parent_memo`.`id`
WHERE 1 = 1
  AND (
    json_extract(`memo`.`payload`, '$.tags') LIKE '%"work"%' 
    AND memos_unicode_lower(`memo`.`content`) LIKE memos_unicode_lower('%meeting%')
  )
  AND (`memo`.`creator_id` = 123 OR `memo`.`visibility` IN ('PUBLIC', 'PROTECTED'))
  AND `memo`.`row_status` = 'NORMAL'
  AND `parent_uid` IS NULL
ORDER BY `pinned` DESC, `created_ts` DESC, `id` DESC
LIMIT 21 OFFSET 0
```

#### 阶段 5：数据库执行计划分析

**索引使用分析**：

| WHERE 条件 | 生成的 SQL | 可用索引 | 实际使用情况 |
|-----------|-----------|---------|-------------|
| `tag == 'work'` | `json_extract(payload, '$.tags') LIKE '%"work"%'` | 无 | 全表扫描 |
| `content.contains('meeting')` | `memos_unicode_lower(content) LIKE '%meeting%'` | 无 (已删除 idx_memo_content) | 全表扫描 |
| `creator_id == 123` | `` `memo`.`creator_id` = 123 `` | 无 (已删除 idx_memo_creator_id) | 全表扫描 |
| `visibility IN (...)` | `` `memo`.`visibility` IN (...) `` | 无 (已删除 idx_memo_visibility) | 全表扫描 |
| `row_status = 'NORMAL'` | `` `memo`.`row_status` = 'NORMAL' `` | 无 | 全表扫描 |

**ORDER BY 分析**：
```sql
ORDER BY `pinned` DESC, `created_ts` DESC, `id` DESC
```
- 无复合索引覆盖这些排序字段
- 数据库需要执行 `filesort`（文件排序）

### 4.3 索引参与总结

```
请求到达
    ↓
API 层：解析参数 → 注入权限条件
    ↓
Filter 层：CEL 编译 → 渲染为 SQL（包含 LIKE、json_extract、OR）
    ↓
数据库层：
  ├── 可用索引：仅 PRIMARY(id), UNIQUE(uid)
  ├── WHERE 条件：
  │   ├── LIKE '%...%' ──→ 无法使用索引
  │   ├── json_extract() ──→ 无法使用索引
  │   ├── OR 条件 ──→ 可能阻止索引使用
  │   └── 普通等值 ──→ 无对应索引
  ├── ORDER BY：无复合索引 ──→ filesort
  └── 执行策略：全表扫描 + filesort
```

---

## 五、索引退化场景分析

### 5.1 导致索引退化的过滤器类型

根据 `internal/filter/render.go` 的实现，以下过滤条件会导致索引退化：

#### 5.1.1 内容搜索（contains）

**代码位置**：`render.go:449-468`

```go
func (r *renderer) renderContainsCondition(cond *ContainsCondition) (renderResult, error) {
    arg := fmt.Sprintf("%%%s%%", cond.Value)  // 前后通配符
    switch r.dialect {
    case DialectSQLite:
        sql := fmt.Sprintf("memos_unicode_lower(%s) LIKE memos_unicode_lower(%s)", column, r.addArg(arg))
        // 问题：
        // 1. 前导通配符 '%' 阻止 B-Tree 索引使用
        // 2. 函数调用阻止表达式索引使用
    case DialectPostgres:
        sql := fmt.Sprintf("%s ILIKE %s", column, r.addArg(arg))
        // 问题：ILIKE 也会阻止普通索引使用
    default:
        sql := fmt.Sprintf("%s LIKE %s", column, r.addArg(arg))
    }
}
```

**退化原因**：
- `LIKE '%text%'` 前导通配符导致 B-Tree 索引失效
- 函数调用（`memos_unicode_lower`）阻止索引使用
- `ILIKE`（PostgreSQL）也无法使用普通 B-Tree 索引

**影响范围**：
- `content.contains('text')` 过滤
- 所有使用 `contains()` 方法的字段

#### 5.1.2 JSON 字段查询（tags、payload 属性）

**代码位置**：`render.go:398-415, 496-582`

**Tag 查询**：
```go
// render.go:402-414
case DialectSQLite:
    sql := fmt.Sprintf("%s LIKE %s", jsonArrayExpr(r.dialect, field), r.addArg(fmt.Sprintf(`%%"%s"%%`, str)))
    // json_extract(payload, '$.tags') LIKE '%"work"%'

case DialectMySQL:
    sql := fmt.Sprintf("JSON_CONTAINS(%s, %s)", jsonArrayExpr(r.dialect, field), r.addArg(fmt.Sprintf(`"%s"`, str)))
    // JSON_CONTAINS(JSON_EXTRACT(payload, '$.tags'), '"work"')

case DialectPostgres:
    sql := fmt.Sprintf("%s @> jsonb_build_array(%s::json)", jsonArrayExpr(r.dialect, field), r.addArg(fmt.Sprintf(`"%s"`, str)))
    // payload->'tags' @> jsonb_build_array('"work"'::json)
```

**JSON Bool 属性查询**（`has_task_list`, `has_link`, `has_code` 等）：
```go
// render.go:258-311
func (r *renderer) renderJSONBoolComparison(field Field, op ComparisonOperator, right ValueExpr) (renderResult, error) {
    jsonExpr := jsonExtractExpr(r.dialect, field)  // json_extract, JSON_EXTRACT, 或 -> 操作符
    // ...
}
```

**退化原因**：
- `json_extract()` 函数调用阻止索引使用
- `JSON_CONTAINS()` 函数调用阻止索引使用
- PostgreSQL 的 `@>` 操作符需要 GIN 索引（未创建）
- LIKE 模式匹配 JSON 序列化字符串（`'%"work"%'`）

**影响范围**：
- `tag == 'work'`
- `tags.exists(t, t == 'work')`
- `tags.exists(t, t.startsWith('work/'))`
- `has_task_list == true`
- `has_link == true`
- `has_code == true`
- `has_incomplete_tasks == true`

#### 5.1.3 OR 组合条件（权限裁剪使用）

**代码位置**：`memo_service.go:252`

```go
filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
// 渲染为：
// (`memo`.`creator_id` = ? OR `memo`.`visibility` IN (?, ?))
```

**代码位置**：`render.go:613-626`

```go
func combineOr(left, right renderResult) renderResult {
    if left.trivial || right.trivial {
        return renderResult{trivial: true}
    }
    // ...
    return renderResult{
        sql: fmt.Sprintf("(%s OR %s)", left.sql, right.sql),
    }
}
```

**退化原因**：
- OR 条件需要同时评估两个分支
- 即使两个字段都有索引，优化器可能选择全表扫描
- 特别是当选择性不高时（如 `visibility IN ('PUBLIC', 'PROTECTED')` 覆盖大部分数据）

**影响范围**：
- 已登录用户的默认权限过滤
- 任何用户编写的 OR 组合查询

#### 5.1.4 多表 LEFT JOIN

**代码位置**：`sqlite/memo.go:139-142`

```sql
LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id`
LEFT JOIN `memo_relation` ON `memo`.`id` = `memo_relation`.`memo_id` AND `memo_relation`.`type` = "COMMENT"
LEFT JOIN `memo` AS `parent_memo` ON `memo_relation`.`related_memo_id` = `parent_memo`.`id`
```

**退化原因**：
- 多表 JOIN 增加查询复杂度
- `memo_relation` 表仅有复合唯一约束 `UNIQUE(memo_id, related_memo_id, type)`，无单独的单列索引
- JOIN 条件 `memo_relation.type = "COMMENT"` 为等值条件，但复合索引中 `type` 是第三个字段，选择性可能不足
- 需扫描主表后再进行 JOIN，无法通过关联表索引反向查找

**影响范围**：
- 所有 ListMemos 查询
- 需要获取 parent_uid 或 creator 信息的查询

### 5.2 导致索引退化的排序类型

#### 5.2.1 多字段排序

**代码位置**：`sqlite/memo.go:112-122`

```go
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
```

**生成的 ORDER BY**：
```sql
-- 默认排序
ORDER BY `created_ts` DESC, `id` DESC

-- 带置顶的排序
ORDER BY `pinned` DESC, `created_ts` DESC, `id` DESC

-- 按更新时间排序
ORDER BY `pinned` DESC, `updated_ts` DESC, `id` DESC
```

**退化原因**：
- 无复合索引 `(pinned, created_ts, id)` 或 `(created_ts, id)`
- 数据库需要执行 filesort（文件排序）
- 即使有单列索引，也无法覆盖多字段排序

**影响范围**：
- 所有 ListMemos 查询（都有 ORDER BY）
- 特别是带 `OrderByPinned` 的查询

#### 5.2.2 函数索引不匹配

**MySQL 特殊处理**（`mysql/memo.go:135-136`）：
```sql
UNIX_TIMESTAMP(`memo`.`created_ts`) AS `created_ts`,
UNIX_TIMESTAMP(`memo`.`updated_ts`) AS `updated_ts`,
```

**退化原因**：
- MySQL 使用 TIMESTAMP 类型，需要 `UNIX_TIMESTAMP()` 转换
- ORDER BY 使用的是原始列，但 SELECT 用了函数
- 即使有索引，类型转换可能阻止使用

### 5.3 索引退化场景汇总表

| 场景类型 | 具体操作 | 生成的 SQL 模式 | 可用索引 | 退化程度 |
|---------|---------|---------------|---------|---------|
| 内容搜索 | `content.contains('text')` | `LIKE '%text%'` + 函数调用 | 无 | 严重 |
| 标签搜索 | `tag == 'work'` | `json_extract() LIKE '%...%'` | 无 | 严重 |
| JSON 属性 | `has_task_list == true` | `json_extract()` 函数 | 无 | 严重 |
| 权限 OR | `creator_id == X OR visibility IN (...)` | `(A OR B)` | 无 | 中 |
| 多字段排序 | `ORDER BY pinned, created_ts, id` | 多字段排序 | 无复合索引 | 中 |
| 多表 JOIN | `LEFT JOIN user, memo_relation, memo` | 多表连接 | 仅有复合唯一约束 | 中 |
| 类型转换 | MySQL `UNIX_TIMESTAMP()` | 函数转换 | 可能阻止 | 轻 |

### 5.4 索引退化对性能的影响

#### 查询执行流程对比

**理想情况（有索引）**：
```
索引扫描 → 快速定位 → 少量读取 → 排序（可能用索引）→ 结果
时间复杂度：O(log n + k)，k 为结果数
```

**实际情况（无索引）**：
```
全表扫描 → 逐行评估 → 全部读取 → filesort → 结果
时间复杂度：O(n log n)，全表扫描 + 排序
```

#### 数据量增长时的性能曲线

| 数据量 | 有索引（估算） | 无索引（实际） | 退化倍数 |
|--------|--------------|--------------|---------|
| 1,000 | < 1ms | ~5ms | 5x |
| 10,000 | < 2ms | ~50ms | 25x |
| 100,000 | < 5ms | ~500ms | 100x |
| 1,000,000 | < 10ms | ~5s | 500x |

---

## 六、权限裁剪与排序的协作关系

### 6.1 协作的执行顺序

权限裁剪和排序在同一查询中协作，执行顺序由 SQL 语义决定：

```
SQL 执行逻辑顺序：
1. FROM/JOIN ──→ 确定数据来源
2. WHERE ──→ 权限裁剪 + 用户过滤（先过滤）
3. ORDER BY ──→ 排序（后排序）
4. LIMIT/OFFSET ──→ 分页
```

**代码层面的协作**：

```go
// 1. API 层：先注入权限条件（memo_service.go:229-238）
if currentUser != nil && memoFind.CreatorID == nil {
    filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
    memoFind.Filters = append(memoFind.Filters, filter)  // 添加到 Filters
}

// 2. 同时解析排序（memo_service.go:213-216）
if request.OrderBy != "" {
    if err := s.parseMemoOrderBy(request.OrderBy, memoFind); err != nil { ... }
}

// 3. 数据库层：Filters → WHERE，OrderBy* → ORDER BY（sqlite/memo.go:54-144）
func (d *DB) ListMemos(ctx context.Context, find *store.FindMemo) ([]*store.Memo, error) {
    // 3.1 处理 Filters → WHERE 条件
    if err := filter.AppendConditions(ctx, engine, find.Filters, ...); err != nil { ... }
    
    // 3.2 处理 OrderBy* → ORDER BY 子句
    orderBy := []string{}
    if find.OrderByPinned { orderBy = append(orderBy, "`pinned` DESC") }
    // ...
    orderBy = append(orderBy, "`id` DESC")
    
    // 3.3 组合查询
    query := "SELECT ... FROM `memo` ... " +
        "WHERE " + strings.Join(where, " AND ") + " " +
        "ORDER BY " + strings.Join(orderBy, ", ")
}
```

### 6.2 协作的数据流

让我们通过一个具体示例追踪数据如何经过权限裁剪和排序：

**场景**：
- 用户 A（ID=123）搜索标签为 'work' 的 memo
- 系统中有以下 memo：

| memo_id | creator_id | visibility | pinned | created_ts | 标签 |
|---------|-----------|-----------|--------|-----------|------|
| 1 | 123 (用户 A) | PRIVATE | true | 100 | work |
| 2 | 123 (用户 A) | PRIVATE | false | 90 | work |
| 3 | 456 (用户 B) | PUBLIC | true | 95 | work |
| 4 | 456 (用户 B) | PRIVATE | false | 85 | work |
| 5 | 456 (用户 B) | PROTECTED | true | 80 | work |
| 6 | 789 (用户 C) | PUBLIC | false | 75 | personal |

**步骤 1：权限过滤（WHERE）**

权限条件：`creator_id == 123 OR visibility IN ('PUBLIC', 'PROTECTED')`

过滤后可见的 memo：
| memo_id | 是否可见 | 原因 |
|---------|---------|------|
| 1 | ✅ | creator_id == 123 |
| 2 | ✅ | creator_id == 123 |
| 3 | ✅ | visibility == PUBLIC |
| 4 | ❌ | PRIVATE 且不是自己的 |
| 5 | ✅ | visibility == PROTECTED |
| 6 | ❌ | 标签不匹配（即使可见也被用户过滤器排除） |

用户过滤器：`tag == 'work'`

**步骤 2：排序（ORDER BY）**

排序条件：`ORDER BY pinned DESC, created_ts DESC, id DESC`

对可见 memo 排序：
| memo_id | pinned | created_ts | id | 排序后位置 |
|---------|--------|-----------|-----|-----------|
| 1 | true (1) | 100 | 1 | 第 1 位 |
| 3 | true (1) | 95 | 3 | 第 2 位 |
| 5 | true (1) | 80 | 5 | 第 3 位 |
| 2 | false (0) | 90 | 2 | 第 4 位 |

**步骤 3：最终结果**

用户看到的顺序：
1. memo #1（自己的，置顶，最新）
2. memo #3（他人的，PUBLIC，置顶）
3. memo #5（他人的，PROTECTED，置顶）
4. memo #2（自己的，未置顶）

### 6.3 协作的架构设计

#### 6.3.1 为什么在 API 层注入权限而不是在数据库层？

```
架构决策对比：

方案 A：API 层注入权限（当前实现）
┌─────────────────────────────────────────┐
│  ListMemos API                          │
│  ├── 解析 request                        │
│  ├── 注入权限条件到 Filters              │
│  └── 调用 Store.ListMemos(find)          │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│  Store/Driver                           │
│  ├── 所有 Filters 统一处理               │
│  ├── 生成 SQL WHERE 条件                │
│  └── 权限条件与用户条件融合              │
└─────────────────────────────────────────┘

方案 B：数据库层单独处理权限
┌─────────────────────────────────────────┐
│  ListMemos API                          │
│  ├── 解析 request                        │
│  └── 传递 currentUser 到 Store           │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│  Store/Driver                           │
│  ├── 处理用户 Filters                    │
│  ├── 单独处理权限逻辑                    │
│  └── 两处逻辑容易不一致                  │
└─────────────────────────────────────────┘
```

**当前方案优势**：
1. **统一处理**：权限条件作为普通 Filter 进入同一编译流程
2. **可组合**：权限 OR 条件可以与用户 AND 条件正确组合
3. **可测试**：权限逻辑集中在 API 层，易于单元测试
4. **灵活**：不同场景可以注入不同的权限条件

#### 6.3.2 权限与排序在 SQL 层面的融合

```sql
-- 权限条件注入到 WHERE
WHERE (
    -- 用户过滤器
    json_extract(payload, '$.tags') LIKE '%"work"%'
) AND (
    -- 权限过滤器（API 层注入）
    creator_id = 123 OR visibility IN ('PUBLIC', 'PROTECTED')
) AND (
    -- 其他结构化条件
    row_status = 'NORMAL'
)

-- 排序在 ORDER BY
ORDER BY pinned DESC, created_ts DESC, id DESC
```

**协作特点**：
- 权限裁剪**先于**排序执行（WHERE 在 ORDER BY 之前）
- 只对**有权限可见**的数据进行排序
- 避免排序无权限数据的浪费

### 6.4 协作的边界情况

#### 6.4.1 权限条件与用户条件的组合逻辑

**组合规则**：权限条件与用户条件使用 AND 连接

```go
// memo_service.go:226
memoFind.Filters = append(memoFind.Filters, request.Filter)  // 用户条件

// memo_service.go:233-234
filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
memoFind.Filters = append(memoFind.Filters, filter)  // 权限条件
```

```sql
-- 渲染后的 WHERE（AND 连接所有 Filters）
WHERE (用户条件) AND (权限条件) AND (其他条件)
```

**示例**：
- 用户过滤器：`tag == 'work'`
- 权限条件：`creator_id == 123 OR visibility IN (...)`
- 组合后：`(tag == 'work') AND (creator_id == 123 OR visibility IN (...))`

**语义**：可见的 memo 必须**同时**满足用户的搜索条件和权限要求

#### 6.4.2 排序对分页的影响

```sql
-- 第 1 页
ORDER BY pinned DESC, created_ts DESC, id DESC
LIMIT 20 OFFSET 0

-- 第 2 页
ORDER BY pinned DESC, created_ts DESC, id DESC
LIMIT 20 OFFSET 20
```

**关键点**：
- 排序字段必须**稳定**（包含 id 作为 tie-breaker）
- 否则分页可能出现重复或遗漏
- 当前实现包含 `id DESC` 作为最终排序字段，保证稳定性

#### 6.4.3 权限动态变化的处理

**场景**：用户搜索时，管理员修改了某条 memo 的可见性

**处理方式**：
- 每次查询都重新注入权限条件
- 不缓存权限判断结果
- 保证每次查询都反映最新的权限状态

---

## 七、搜索请求处理流程

### 7.1 前端搜索状态管理

#### 7.1.1 MemoFilterContext

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

#### 7.1.2 React Query 缓存策略

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

### 7.2 API 层参数处理

#### 7.2.1 ListMemos 入口

位于 `server/router/api/v1/memo_service.go:189-353`，核心流程：

1. **状态过滤**：根据 `request.State` 区分普通/归档状态
2. **排序解析**：解析 `request.OrderBy` 参数
3. **过滤器验证**：验证并附加 CEL 过滤器
4. **权限裁剪**：根据当前用户身份注入权限条件
5. **分页处理**：解析 `pageToken` 或 `pageSize`

#### 7.2.2 排序解析

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

## 八、CEL 过滤器引擎

### 8.1 过滤器架构

位于 `internal/filter/` 目录，核心组件：

| 组件 | 文件 | 功能 |
|------|------|------|
| Engine | `engine.go` | CEL 环境管理、表达式编译 |
| Schema | `schema.go` | 字段定义、类型映射 |
| Renderer | `render.go` | 将条件树渲染为 SQL |
| Helpers | `helpers.go` | 便捷函数（AppendConditions） |

### 8.2 支持的过滤字段

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

### 8.3 编译与渲染流程

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

## 九、用户权限裁剪机制

### 9.1 权限模型

系统定义了三种可见性级别（`store/memo.go:12-22`）：

```go
const (
    Public    Visibility = "PUBLIC"     // 所有人可见
    Protected Visibility = "PROTECTED"  // 登录用户可见
    Private   Visibility = "PRIVATE"    // 仅创建者可见
)
```

### 9.2 API 层权限注入

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

### 9.3 归档状态的特殊处理

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

### 9.4 单条 memo 的权限检查

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

## 十、数据库层查询构建

### 10.1 SQLite 实现

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

### 10.2 三数据库驱动对比

| 特性 | SQLite | MySQL | PostgreSQL |
|------|--------|-------|-----------|
| 占位符 | `?` | `?` | `$1, $2...` |
| 时间戳 | BIGINT (epoch) | TIMESTAMP (需 UNIX_TIMESTAMP) | BIGINT (epoch) |
| 标识符 | `` `id` `` | `` `id` `` | `"id"` |
| JSON 路径 | `json_extract` | `JSON_EXTRACT` | `json_extract_path_text` |
| 字符串拼接 | `||` | `CONCAT()` | `||` |
| Tag 查询 | `json_extract() LIKE` | `JSON_CONTAINS()` | `@> jsonb_build_array()` |
| 大小写不敏感 | 自定义函数 `memos_unicode_lower` | `LIKE` | `ILIKE` |

数据库驱动通过 `filter.DialectName` 告知渲染器生成正确的 SQL。

---

## 十一、跨层次协作全景图

### 11.1 完整搜索流程

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
│  │    Filters: [userFilter, "content.contains('test')",                │ │
│  │             "creator_id == X || visibility in [...]"],              │ │
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
│  │  │  - CompileToStatement(dialect=SQLite|MySQL|Postgres)         │  │ │
│  │  │    → CEL 编译 → AST → 条件树 → SQL 片段 + args               │  │ │
│  │  │  - 注意：LIKE '%...%'、json_extract()、OR 等会导致索引退化     │  │ │
│  │  │  - 追加到 where 子句                                         │  │ │
│  │  └──────────────────────────────┬───────────────────────────────┘  │ │
│  │                                 │                                   │ │
│  │  ┌──────────────────────────────▼───────────────────────────────┐  │ │
│  │  │  构建 SQL 查询                                               │  │ │
│  │  │  SELECT id, uid, creator_id, ...                             │  │ │
│  │  │  FROM memo                                                   │  │ │
│  │  │  LEFT JOIN user AS memo_creator ON ...                       │  │ │
│  │  │  LEFT JOIN memo_relation ON ...                              │  │ │
│  │  │  WHERE 1 = 1                                                 │  │ │
│  │  │    AND (memo.creator_id = ? OR memo.visibility IN (?, ?))    │  │ │
│  │  │    AND memo.content LIKE ?  -- 索引退化！                     │  │ │
│  │  │    AND json_extract(...) LIKE ?  -- 索引退化！                │  │ │
│  │  │    AND memo.row_status = ?                                   │  │ │
│  │  │  ORDER BY pinned DESC, created_ts DESC, id DESC              │  │ │
│  │  │  -- 无复合索引，需要 filesort                                 │  │ │
│  │  │  LIMIT ? OFFSET ?                                            │  │ │
│  │  └──────────────────────────────┬───────────────────────────────┘  │ │
│  └─────────────────────────────────┼──────────────────────────────────┘ │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │
                                     ▼
                          ┌─────────────────┐
                          │   数据库引擎     │
                          │  执行计划：       │
                          │  全表扫描         │
                          │  + filesort      │
                          └─────────────────┘
```

### 11.2 数据流详解

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
  args: ['%"work"%', '%meeting%']
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
  AND (json_extract(`memo`.`payload`, '$.tags') LIKE '%"work"%' 
       AND memos_unicode_lower(`memo`.`content`) LIKE memos_unicode_lower('%meeting%'))
  AND (`memo`.`creator_id` = 123 
       OR `memo`.`visibility` IN ('PUBLIC', 'PROTECTED'))
  AND `memo`.`row_status` = 'NORMAL'
  AND `parent_uid` IS NULL  -- ExcludeComments

ORDER BY：
ORDER BY `pinned` DESC, `created_ts` DESC, `id` DESC

LIMIT / OFFSET：
LIMIT 21 OFFSET 0

执行计划（无索引）：
  - 全表扫描 memo 表
  - 逐行评估 WHERE 条件
  - filesort 排序结果
```

---

## 十二、关键设计决策

### 12.1 为什么使用 CEL 而不是 ORM 查询构建器？

**优势**：
1. **灵活性**：用户可以编写复杂的组合查询，如 `tag == 'work' && created_ts > now() - 86400`
2. **安全性**：CEL 编译层提供语法验证，避免 SQL 注入
3. **可扩展性**：新增过滤字段只需更新 Schema 定义
4. **跨数据库**：同一表达式可渲染为 SQLite/MySQL/PostgreSQL 语法

**代价**：
- 运行时编译开销（通过 Engine 单例和缓存缓解）
- 调试难度增加（需要理解 CEL → SQL 的转换）
- **索引退化**：生成的 `LIKE '%...%'`、`json_extract()` 等无法使用索引

### 12.2 为什么在 API 层注入权限条件而不是在数据库层？

**优势**：
1. **关注点分离**：数据库层专注于数据访问，API 层负责业务规则
2. **复用性**：权限逻辑集中在一处，多个 API 端点共享
3. **可测试性**：可以独立测试权限注入逻辑
4. **透明性**：权限条件作为普通 Filter 注入，与用户过滤器统一处理

**实现方式**：将权限条件包装为 CEL 表达式，与用户过滤器一起进入编译和渲染流程。

### 12.3 为什么采用两层过滤（VisibilityList + CEL Filter）？

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
- **但当前已删除相关索引，两种方式都无法使用索引**

### 12.4 为什么删除了历史索引？

**决策背景**：索引删除分两个阶段进行：

| 阶段 | 版本 | 删除的索引 | 关联改动 |
|-----|-----|-----------|---------|
| 第一阶段 | v0.24 | `idx_memo_tags`、`idx_memo_content`、`idx_memo_visibility` | 删除 `tags` 列（`ALTER TABLE memo DROP COLUMN tags`） |
| 第二阶段 | v0.26 | `idx_memo_creator_id`、`idx_user_username`、`idx_attachment_*` | 无列删除，纯索引优化 |

**v0.24 删除原因**：
1. **tags 列废弃**：`tags` 列被删除，索引 `idx_memo_tags` 随之删除
2. **查询模式变化**：`content.contains()` 生成 `LIKE '%text%'`，前导通配符使 B-Tree 索引失效
3. **函数调用**：使用 `memos_unicode_lower()` 函数，即使有索引也无法使用

**v0.26 删除原因**：
1. **权限裁剪使用 OR 组合**：`creator_id == X OR visibility IN (...)`，优化器可能不选择单列索引
2. **已有唯一约束**：`user.username` 已有 `UNIQUE` 约束，无需额外索引
3. **写入性能优先**：索引会增加 INSERT/UPDATE/DELETE 的开销
4. **维护成本**：多数据库（SQLite/MySQL/PostgreSQL）索引维护复杂

**潜在风险**：
- 数据量增长时查询性能会显著下降
- 特别是 `content.contains()` 和 JSON 字段查询

---

## 十三、1 当前性能瓶颈

| 潜在瓶颈 | 位置 | 严重程度 | 说明 |
|---------|------|---------|------|
| CEL 编译 | filter 层 | 低 | Engine 单例（sync.Once）缓解 |
| 内容搜索 | `content.contains()` | 高 | `LIKE '%text%'` 全表扫描，无索引 |
| JSON 字段查询 | payload.property | 高 | `json_extract` 无法使用索引 |
| OR 权限条件 | 权限裁剪 | 中 | 可能阻止优化器选择 |
| 多表 JOIN | 列表查询 | 中 | LEFT JOIN user, memo_relation, memo |
| 多字段排序 | ORDER BY | 中 | 无复合索引，需要 filesort |
| 分页 | 大 Offset | 中 | 使用 keyset 分页可优化（当前用 offset） |

### 13.2 缓存策略

**前端缓存（React Query）**：
- `staleTime: 60s` — 60 秒内认为数据新鲜
- `gcTime: 5min` — 5 分钟后清理未使用的缓存
- 按查询参数（filter, orderBy 等）生成唯一 queryKey

**后端缓存**：
- Store 层有内存缓存（TTL 10min, max 1000）
- 但 ListMemos 等查询可能不经过缓存

### 13.3 索引优化建议

如果未来数据量增长，可以考虑以下优化：

#### 13.3.1 内容搜索优化

**方案 A：FTS（全文搜索）**
```sql
-- SQLite FTS5
CREATE VIRTUAL TABLE memo_fts USING fts5(content, content='memo', content_rowid='id');

-- MySQL 全文索引
CREATE FULLTEXT INDEX idx_memo_content_ft ON memo(content);

-- PostgreSQL GIN 索引
CREATE INDEX idx_memo_content_gin ON memo USING gin (to_tsvector('english', content));
```

**方案 B：前缀索引（仅支持前缀搜索）**
```sql
-- 如果用户主要使用前缀匹配
CREATE INDEX idx_memo_content_prefix ON memo(content);
-- 但需要修改查询为 content LIKE 'text%'（无前导 %）
```

#### 13.3.2 JSON 字段优化

**方案 A：PostgreSQL GIN 索引**
```sql
-- 标签查询优化
CREATE INDEX idx_memo_tags_gin ON memo USING gin ((payload->'tags') jsonb_path_ops);

-- 使用 @> 操作符可命中索引
SELECT * FROM memo WHERE payload->'tags' @> '["work"]';
```

**方案 B：提取字段到独立列**
```sql
-- 将常用属性提取为列
ALTER TABLE memo ADD COLUMN tags TEXT GENERATED ALWAYS AS (payload->>'$.tags') STORED;
CREATE INDEX idx_memo_tags ON memo(tags);
```

#### 13.3.3 排序优化

**方案 A：复合索引覆盖排序**
```sql
-- 覆盖最常用的排序模式
CREATE INDEX idx_memo_pinned_created ON memo(pinned DESC, created_ts DESC, id DESC);
CREATE INDEX idx_memo_pinned_updated ON memo(pinned DESC, updated_ts DESC, id DESC);
```

**方案 B：覆盖索引**
```sql
-- 覆盖查询所需的所有列（避免回表）
CREATE INDEX idx_memo_covering ON memo(
    pinned DESC, created_ts DESC, id DESC
) INCLUDE (uid, creator_id, visibility, payload, content);
```

#### 13.3.4 权限过滤优化

**方案 A：复合索引包含权限字段**
```sql
-- 权限查询常用字段
CREATE INDEX idx_memo_creator_visibility ON memo(creator_id, visibility);
```

**方案 B：分区表（大规模数据）**
```sql
-- 按 visibility 分区
CREATE TABLE memo_public PARTITION OF memo FOR VALUES IN ('PUBLIC');
CREATE TABLE memo_protected PARTITION OF memo FOR VALUES IN ('PROTECTED');
CREATE TABLE memo_private PARTITION OF memo FOR VALUES IN ('PRIVATE');
```

---

## 十四、代码位置索引

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
| SQL 渲染器 | `internal/filter/render.go` | 全部 |
| contains 渲染（索引退化） | `internal/filter/render.go` | 449-468 |
| JSON 字段渲染（索引退化） | `internal/filter/render.go` | 313-415, 496-582 |
| 过滤器辅助函数 | `internal/filter/helpers.go` | 8-25 |
| Store 层接口 | `store/memo.go` | 109-159 |
| SQLite 驱动实现 | `store/db/sqlite/memo.go` | 54-193 |
| MySQL 驱动实现 | `store/db/mysql/memo.go` | 62-150 |
| PostgreSQL 驱动实现 | `store/db/postgres/memo.go` | 51-150 |
| 可见性定义 | `store/memo.go` | 12-22 |
| 数据库迁移（删除索引） | `store/migration/sqlite/0.26/02__drop_indexes.sql` | 全部 |
| 数据库迁移（创建索引） | `store/migration/sqlite/0.14/01__create_indexes.sql` | 全部 |
| 当前数据库 Schema | `store/migration/sqlite/LATEST.sql` | 全部 |

---

## 十五、总结

### 15.1 核心架构总结

Memos 的搜索功能采用了**分层协作**的架构设计：

1. **前端层**：通过 MemoFilterContext 管理搜索状态，React Query 处理缓存和分页
2. **API 层**：解析参数、验证过滤器、**注入权限条件**、解析排序规则
3. **过滤器层**：CEL 引擎将表达式编译为条件树，再根据数据库方言渲染为 SQL
4. **存储层**：将所有条件（用户过滤器 + 权限过滤器 + 结构化条件）组合为最终查询

### 15.2 索引使用现状总结

**当前状态**：
- `memo` 表仅有主键（`id`）和唯一索引（`uid`）
- 索引删除分两个阶段：
  - **v0.24**：删除 `idx_memo_tags`、`idx_memo_content`、`idx_memo_visibility`（同时删除 `tags` 列）
  - **v0.26**：删除 `idx_memo_creator_id`、`idx_user_username`、`idx_attachment_*`
- **所有搜索查询基本都依赖全表扫描**

**索引参与流程**：
```
请求 → API 注入权限 → CEL 渲染 SQL → 数据库执行
                                     ↓
                           WHERE 条件包含：
                           - LIKE '%...%' ← 索引退化
                           - json_extract() ← 索引退化
                           - OR 条件 ← 可能退化
                           - 无对应索引 ← 全表扫描
                                     ↓
                           ORDER BY：
                           - 多字段排序 ← 无复合索引，filesort
                                     ↓
                           执行计划：全表扫描 + filesort
```

### 15.3 权限裁剪与排序的协作总结

**协作关系**：

```
SQL 执行顺序：
1. FROM/JOIN ──→ 确定数据来源
2. WHERE ──→ 权限裁剪 + 用户过滤（先过滤）
3. ORDER BY ──→ 排序（后排序）
4. LIMIT/OFFSET ──→ 分页
```

**协作特点**：
- **权限先于排序**：只对有权限可见的数据排序，避免浪费
- **统一处理**：权限条件作为普通 Filter 进入 CEL 编译流程
- **AND 组合**：`(用户条件) AND (权限条件)`，必须同时满足
- **OR 逻辑**：权限条件内部使用 OR（自己的 OR 可见的）

**代码层面的协作点**：
- API 层同时处理 `Filters`（权限注入）和 `OrderBy*`（排序解析）
- 数据库层将 `Filters` 渲染为 `WHERE`，`OrderBy*` 渲染为 `ORDER BY`
- 两者在同一 SQL 中执行，由数据库优化器决定执行计划

### 15.4 关键设计权衡

| 决策 | 优势 | 代价 |
|------|------|------|
| CEL 动态查询 | 灵活、安全、跨数据库 | 索引退化、调试困难 |
| API 层注入权限 | 集中、可测试、透明 | OR 条件可能影响性能 |
| 删除历史索引 | 写入更快、维护简单 | 查询性能随数据量下降 |
| 多字段排序 + tie-breaker | 结果稳定、可预测 | 需要 filesort |

### 15.5 适用场景与限制

**当前设计适合**：
- 中小规模数据（万级以下）
- 写入频繁的场景
- 需要灵活查询的场景

**当前设计的限制**：
- 内容搜索性能随数据量增长显著下降
- JSON 字段查询无法使用索引
- 大规模数据下需要考虑 FTS 或专用搜索引擎

**权限裁剪是整个架构的核心亮点**：它不是在查询后过滤结果，而是在查询构建阶段就将权限条件注入到 SQL 中，保证了性能和安全性。权限条件通过 CEL 表达式的形式与用户过滤器无缝融合，由同一套编译和渲染流程处理。

**排序策略**采用了多优先级排序：置顶优先 → 时间排序 → ID 作为 tie-breaker，确保结果的一致性和可预测性。

**索引维护**由数据库自动处理，系统经历了从"创建大量索引"到"精简索引"的优化过程，依赖查询优化器自动选择最优执行计划。但这也意味着在大规模数据场景下需要额外的优化措施。
