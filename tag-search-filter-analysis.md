# Memos 标签、全文检索和多条件过滤协作机制分析

## 概述

本文档深入分析 memos 项目中标签管理、全文检索和多条件过滤在前端列表状态与后端查询层之间的协作机制。通过追踪数据从用户交互到数据库查询的完整流转路径，揭示前后端如何通过精心设计的状态管理、查询语言和渲染层实现无缝协作。

---

## 一、整体架构概览

### 1.1 核心数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端层 (React + TypeScript)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────────────────────┐ │
│  │ TagsSection  │    │  SearchBar   │    │      其他过滤操作 (属性选择等)    │ │
│  │  (标签点击)   │    │ (关键词搜索)  │    │                                 │ │
│  └──────┬───────┘    └──────┬───────┘    └───────────────┬─────────────────┘ │
│         │                    │                              │                   │
│         └────────────────────┼──────────────────────────────┘                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │      MemoFilterContext           │                             │
│              │  (过滤器状态管理 + URL同步)        │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │        useMemoFilters             │                             │
│              │  (MemoFilter → CEL 表达式转换)     │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │      useInfiniteMemos             │                             │
│              │  (React Query + gRPC API 调用)    │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
└──────────────────────────────┼────────────────────────────────────────────────────┘
                               │
                               ▼ (gRPC: ListMemosRequest.filter)
┌─────────────────────────────────────────────────────────────────────────────┐
│                              后端层 (Go + Echo + gRPC)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                              │                                                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │      memo_service.go              │                             │
│              │   ListMemos + validateFilter      │                             │
│              │  (CEL 语法验证 + 权限过滤)         │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │         store/memo.go             │                             │
│              │      FindMemo.Filters []string    │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │   store/db/sqlite/memo.go (等)    │                             │
│              │      ListMemos SQL 构建            │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
│                              ▼                                                   │
│              ┌───────────────────────────────────┐                             │
│              │      internal/filter/             │                             │
│              │  CEL 引擎 → SQL 条件编译           │                             │
│              │  (engine + parser + render)       │                             │
│              └───────────────┬───────────────────┘                             │
│                              │                                                   │
└──────────────────────────────┼────────────────────────────────────────────────────┘
                               │
                               ▼ (SQL 查询)
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据库层 (SQLite/MySQL/PostgreSQL)                │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  SELECT ... FROM memo WHERE                                               │ │
│  │    (content LIKE '%keyword%') AND                                        │ │
│  │    (JSON_EXTRACT(payload, '$.tags') LIKE '%"tag"%' OR ...) AND         │ │
│  │    (JSON_EXTRACT(payload, '$.property.hasLink') IS TRUE) AND ...        │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| 前端状态管理 | React Context + useSearchParams | 过滤器状态与 URL 双向同步 |
| 前端数据查询 | React Query (useInfiniteQuery) | 缓存管理、无限滚动、乐观更新 |
| API 协议 | gRPC + Connect (Protocol Buffers) | 类型安全的前后端通信 |
| 查询语言 | CEL (Common Expression Language) | 跨数据库的统一查询表达式 |
| 后端存储 | SQLite / MySQL / PostgreSQL | 多数据库驱动支持 |

---

## 二、标签管理环节

### 2.1 前端标签组件

#### 2.1.1 TagsSection 组件

**文件**: `web/src/components/MemoExplorer/TagsSection.tsx`

TagsSection 是标签选择的核心 UI 组件，负责：

1. **标签展示**：从 `tagCount` props 接收标签及其计数，按使用频率排序
2. **标签点击交互**：点击标签时切换过滤状态
3. **视图模式切换**：支持平面视图和树形视图

```typescript
// 核心交互逻辑
const handleTagClick = (tag: string) => {
  const isActive = getFiltersByFactor("tagSearch").some(
    (filter: MemoFilter) => filter.value === tag
  );
  if (isActive) {
    // 移除标签过滤
    removeFilter((f: MemoFilter) => f.factor === "tagSearch" && f.value === tag);
  } else {
    // 先移除所有现有标签过滤，再添加新的（单选模式）
    removeFilter((f: MemoFilter) => f.factor === "tagSearch");
    addFilter({
      factor: "tagSearch",
      value: tag,
    });
  }
};
```

**设计亮点**：
- 标签选择采用**单选模式**：每次点击新标签会移除旧的标签过滤
- 支持**层级标签**：通过 TagTree 组件实现树形标签视图
- 标签状态与 URL 同步：刷新页面后标签选择状态保留

#### 2.1.2 标签数据来源

标签计数数据通过 `MemoExplorer` 组件的 `tagCount` props 传入，通常来自用户统计接口。标签存储在 `memo.payload.tags` JSON 数组中。

### 2.2 标签过滤状态管理

#### 2.2.1 FilterFactor 类型定义

**文件**: `web/src/contexts/MemoFilterContext.tsx`

```typescript
export type FilterFactor =
  | "tagSearch"       // 标签过滤
  | "visibility"      // 可见性过滤
  | "contentSearch"   // 内容搜索
  | "displayTime"     // 显示时间过滤
  | "pinned"          // 置顶过滤
  | "property.hasLink"      // 包含链接
  | "property.hasTaskList"  // 包含任务列表
  | "property.hasCode";     // 包含代码块
```

#### 2.2.2 URL 同步机制

MemoFilterContext 使用 `useSearchParams` 实现过滤器状态与 URL 的双向同步：

```typescript
// 从 URL 解析过滤器
export const parseFilterQuery = (query: string | null): MemoFilter[] => {
  if (!query) return [];
  try {
    return query.split(",").map((filterStr) => {
      const [factor, value] = filterStr.split(":");
      return {
        factor: factor as FilterFactor,
        value: decodeURIComponent(value || ""),
      };
    });
  } catch {
    return [];
  }
};

// 将过滤器序列化为 URL 参数
export const stringifyFilters = (filters: MemoFilter[]): string => {
  return filters
    .map((filter) => `${filter.factor}:${encodeURIComponent(filter.value)}`)
    .join(",");
};
```

**URL 示例**：
- `?filter=tagSearch:work,contentSearch:meeting`
- 表示：过滤标签为 "work" 且内容包含 "meeting" 的笔记

### 2.3 标签层级支持

标签系统支持**层级标签**（如 `work/project`、`personal/learning`），后端在 SQL 层面实现了前缀匹配：

**文件**: `internal/filter/render.go` (renderTagInList 函数)

```go
// SQLite 方言的层级标签匹配
exactMatch := fmt.Sprintf(
  "%s LIKE %s", 
  jsonArrayExpr(r.dialect, field), 
  r.addArg(fmt.Sprintf(`%%"%s"%%`, str))
)
prefixMatch := fmt.Sprintf(
  "%s LIKE %s", 
  jsonArrayExpr(r.dialect, field), 
  r.addArg(fmt.Sprintf(`%%"%s/%%`, str))
)
expr := fmt.Sprintf("(%s OR %s)", exactMatch, prefixMatch)
```

**匹配规则**：
- 搜索标签 `work` 时，会匹配：
  - 精确匹配：`["work"]`
  - 前缀匹配：`["work/project"]`、`["work/project/task"]`

---

## 三、搜索参数传递环节

### 3.1 前端搜索组件

#### 3.1.1 SearchBar 组件

**文件**: `web/src/components/SearchBar.tsx`

SearchBar 提供全文检索输入框，支持多关键词搜索：

```typescript
const onKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === "Enter") {
    e.preventDefault();
    const trimmedText = queryText.trim();
    if (trimmedText !== "") {
      // 按空格分割为多个关键词
      const words = trimmedText.split(/\s+/);
      words.forEach((word) => {
        addFilter({
          factor: "contentSearch",
          value: word,
        });
      });
      setQueryText("");
    }
  }
};
```

**行为特点**：
- 按**回车键**触发搜索
- 空格分隔的**多个关键词**会分别添加为独立的 `contentSearch` 过滤器
- 多个关键词之间是**逻辑 AND** 关系

### 3.2 过滤器到 CEL 表达式的转换

#### 3.2.1 useMemoFilters 钩子

**文件**: `web/src/hooks/useMemoFilters.ts`

这是前端过滤器到后端查询语言的核心转换层，将 `MemoFilter` 转换为 **CEL (Common Expression Language)** 表达式：

```typescript
export const useMemoFilters = (options: UseMemoFiltersOptions = {}): string | undefined => {
  // ...
  return useMemo(() => {
    const conditions: string[] = [];

    // 添加创建者过滤
    if (creatorName) {
      const creatorFilter = buildMemoCreatorFilter(creatorName);
      if (creatorFilter) {
        conditions.push(creatorFilter);
      }
    }

    // 添加快捷方式过滤
    if (includeShortcuts && selectedShortcut?.filter) {
      conditions.push(selectedShortcut.filter);
    }

    // 转换活跃的过滤器
    for (const filter of filters) {
      if (filter.factor === "contentSearch") {
        conditions.push(`content.contains(${escapeFilterValue(filter.value)})`);
      } else if (filter.factor === "tagSearch") {
        conditions.push(`tag in [${escapeFilterValue(filter.value)}]`);
      } else if (filter.factor === "pinned") {
        if (includePinned) {
          conditions.push(`pinned`);
        }
      } else if (filter.factor === "property.hasLink") {
        conditions.push(`has_link`);
      } else if (filter.factor === "property.hasTaskList") {
        conditions.push(`has_task_list`);
      } else if (filter.factor === "property.hasCode") {
        conditions.push(`has_code`);
      } else if (filter.factor === "displayTime") {
        // 构建日期范围条件
        const filterDate = new Date(filter.value);
        const filterUtcTimestamp = filterDate.getTime() + filterDate.getTimezoneOffset() * 60 * 1000;
        const timestampAfter = filterUtcTimestamp / 1000;
        conditions.push(
          `created_ts >= ${timestampAfter} && created_ts < ${timestampAfter + 60 * 60 * 24}`
        );
      }
    }

    // 添加可见性过滤
    if (visibilities && visibilities.length > 0) {
      const visibilityValues = visibilities.map((v) => `"${getVisibilityName(v)}"`).join(", ");
      conditions.push(`visibility in [${visibilityValues}]`);
    }

    return conditions.length > 0 ? conditions.join(" && ") : undefined;
  }, [creatorName, includeShortcuts, includePinned, visibilities, selectedShortcut, filters]);
};
```

#### 3.2.2 转换规则对照表

| 前端 FilterFactor | 后端 CEL 表达式 | 说明 |
|-------------------|-----------------|------|
| `contentSearch: "keyword"` | `content.contains("keyword")` | 内容包含关键词 |
| `tagSearch: "work"` | `tag in ["work"]` | 标签匹配（支持层级） |
| `pinned: "true"` | `pinned` | 置顶笔记 |
| `property.hasLink: "true"` | `has_link` | 包含链接 |
| `property.hasTaskList: "true"` | `has_task_list` | 包含任务列表 |
| `property.hasCode: "true"` | `has_code` | 包含代码块 |
| `displayTime: "2024-01-15"` | `created_ts >= ... && created_ts < ...` | 日期范围 |
| `visibility: "PUBLIC"` | `visibility in ["PUBLIC"]` | 可见性 |

### 3.3 API 调用层

#### 3.3.1 useInfiniteMemos 钩子

**文件**: `web/src/hooks/useMemoQueries.ts`

使用 React Query 的 `useInfiniteQuery` 实现无限滚动列表：

```typescript
export function useInfiniteMemos(
  request: Partial<ListMemosRequest> = {}, 
  options?: { enabled?: boolean }
) {
  return useInfiniteQuery({
    queryKey: memoKeys.list(request),
    queryFn: async ({ pageParam }) => {
      const response = await memoServiceClient.listMemos(
        create(ListMemosRequestSchema, {
          ...request,
          pageToken: pageParam || "",
        } as Record<string, unknown>),
      );
      return response;
    },
    initialPageParam: "",
    getNextPageParam: (lastPage) => lastPage.nextPageToken || undefined,
    staleTime: 1000 * 60,      // 1 分钟缓存
    gcTime: 1000 * 60 * 5,      // 5 分钟垃圾回收
    enabled: options?.enabled ?? true,
  });
}
```

**关键设计**：
- **Query Key 包含过滤器**：`memoKeys.list(request)` 确保不同过滤条件有独立缓存
- **分页 Token**：使用 `pageToken` 实现游标分页，而非传统 offset
- **缓存策略**：1 分钟新鲜期，5 分钟后垃圾回收

#### 3.3.2 ListMemosRequest 协议定义

通过 Protocol Buffers 定义，`filter` 字段是 CEL 表达式字符串：

```protobuf
message ListMemosRequest {
  string parent = 1;
  int32 page_size = 2;
  string page_token = 3;
  State state = 4;
  string order_by = 5;
  string filter = 6;  // CEL 表达式
}
```

---

## 四、列表渲染环节

### 4.1 列表组件结构

#### 4.1.1 PagedMemoList 组件

**文件**: `web/src/components/PagedMemoList/PagedMemoList.tsx`

这是笔记列表的核心渲染组件，负责：

1. **数据获取**：使用 `useInfiniteMemos` 获取分页数据
2. **无限滚动**：监听滚动事件，自动加载下一页
3. **渲染控制**：通过 `renderer` props 自定义每项渲染
4. **过滤器显示**：显示当前激活的过滤器标签

```typescript
const PagedMemoList = (props: Props) => {
  // 使用过滤器构建 CEL 表达式
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isLoading } = 
    useInfiniteMemos(
      {
        state: props.state || State.NORMAL,
        orderBy: props.orderBy || "create_time desc",
        filter: props.filter,  // 传入 CEL 表达式
        pageSize: props.pageSize || DEFAULT_LIST_MEMOS_PAGE_SIZE,
      },
      { enabled: props.enabled ?? true },
    );

  // 扁平化分页数据
  const memos = useMemo(
    () => data?.pages.flatMap((page) => page.memos) || [], 
    [data]
  );

  // 应用自定义排序
  const sortedMemoList = useMemo(
    () => (props.listSort ? props.listSort(memos) : memos), 
    [memos, props.listSort]
  );

  // 无限滚动：滚动到底部时加载更多
  useEffect(() => {
    if (!hasNextPage) return;
    const handleScroll = () => {
      const nearBottom = window.innerHeight + window.scrollY >= document.body.offsetHeight - 300;
      if (nearBottom && !isFetchingNextPage) {
        fetchNextPage();
      }
    };
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <MentionResolutionProvider contents={sortedMemoList.map((memo) => memo.content)}>
      <div className="flex flex-col justify-start w-full max-w-2xl mx-auto">
        {/* 编辑器（可选） */}
        {showMemoEditor ? <MemoEditor ... /> : null}
        
        {/* 当前激活的过滤器标签 */}
        <MemoFilters />
        
        {/* 笔记列表 */}
        {sortedMemoList.map((memo) => props.renderer(memo))}
        
        {/* 加载指示器 */}
        {isFetchingNextPage && <Skeleton showCreator={props.showCreator} count={2} />}
        
        {/* 空状态或返回顶部 */}
        {!isFetchingNextPage && (
          !hasNextPage && sortedMemoList.length === 0 ? 
            <Empty /> : 
            <BackToTop />
        )}
      </div>
    </MentionResolutionProvider>
  );
};
```

#### 4.1.2 MemoFilters 组件

**文件**: `web/src/components/MemoFilters.tsx`

显示当前激活的过滤器，支持逐个移除：

```typescript
const MemoFilters = () => {
  const t = useTranslate();
  const { filters, removeFilter } = useMemoFilterContext();

  // 过滤器配置：图标和标签
  const FILTER_CONFIGS: Record<FilterFactor, FilterConfig> = {
    tagSearch: {
      icon: HashIcon,
      getLabel: (value) => value,
    },
    contentSearch: {
      icon: SearchIcon,
      getLabel: (value) => value,
    },
    visibility: {
      icon: EyeIcon,
      getLabel: (value) => value,
    },
    // ... 其他过滤器
  };

  if (filters.length === 0) {
    return null;
  }

  return (
    <div className="w-full mb-2 flex flex-row justify-start items-center flex-wrap gap-2">
      {filters.map((filter) => {
        const config = FILTER_CONFIGS[filter.factor];
        const Icon = config?.icon;
        return (
          <div key={getMemoFilterKey(filter)} className="...">
            {Icon && <Icon className="w-3.5 h-3.5 text-muted-foreground shrink-0" />}
            <span className="text-foreground/80 font-medium max-w-32 truncate">
              {getFilterDisplayText(filter)}
            </span>
            <button
              onClick={() => handleRemoveFilter(filter)}
              className="ml-0.5 -mr-1 p-0.5 text-muted-foreground/60 hover:text-destructive hover:bg-destructive/10 rounded-full"
            >
              <XIcon className="w-3 h-3" />
            </button>
          </div>
        );
      })}
    </div>
  );
};
```

### 4.2 页面组装示例

**文件**: `web/src/pages/Home.tsx`

首页展示了完整的过滤-查询-渲染流程：

```typescript
const Home = () => {
  const user = useCurrentUser();
  const { isInitialized } = useInstance();

  // 1. 构建 CEL 过滤表达式
  const memoFilter = useMemoFilters({
    creatorName: user?.name,        // 只显示当前用户的笔记
    includeShortcuts: true,          // 包含快捷方式过滤
    includePinned: true,              // 包含置顶过滤
  });

  // 2. 构建排序参数
  const { listSort, orderBy } = useMemoSorting({
    pinnedFirst: true,                // 置顶优先
    state: State.NORMAL,
  });

  // 3. 渲染列表
  return (
    <div className="w-full min-h-full bg-background text-foreground">
      <PagedMemoList
        renderer={(memo: Memo) => (
          <MemoView 
            key={`${memo.name}-${memo.updateTime}`} 
            memo={memo} 
            showVisibility 
            showPinned 
            compact 
          />
        )}
        listSort={listSort}
        orderBy={orderBy}
        filter={memoFilter}        // 传入 CEL 表达式
        enabled={isInitialized}
        showMemoEditor
      />
    </div>
  );
};
```

---

## 五、后端查询层处理

### 5.1 API 层处理

#### 5.1.1 ListMemos 服务方法

**文件**: `server/router/api/v1/memo_service.go`

```go
func (s *APIV1Service) ListMemos(ctx context.Context, request *v1pb.ListMemosRequest) (*v1pb.ListMemosResponse, error) {
  memoFind := &store.FindMemo{
    ExcludeComments: true,  // 默认排除评论
  }
  currentUser, err := s.fetchCurrentUser(ctx)
  // ... 错误处理

  // 状态过滤（NORMAL / ARCHIVED）
  if request.State == v1pb.State_ARCHIVED {
    state := store.Archived
    memoFind.RowStatus = &state
    // 归档笔记只对创建者可见
    if currentUser == nil {
      return &v1pb.ListMemosResponse{}, nil
    }
    memoFind.CreatorID = &currentUser.ID
  } else {
    state := store.Normal
    memoFind.RowStatus = &state
  }

  // 解析排序参数
  if request.OrderBy != "" {
    if err := s.parseMemoOrderBy(request.OrderBy, memoFind); err != nil {
      return nil, status.Errorf(codes.InvalidArgument, "invalid order_by: %v", err)
    }
  } else {
    memoFind.OrderByTimeAsc = false  // 默认按创建时间倒序
  }

  // 关键：验证并添加过滤器
  if request.Filter != "" {
    if err := s.validateFilter(ctx, request.Filter); err != nil {
      return nil, status.Errorf(codes.InvalidArgument, "invalid filter: %v", err)
    }
    memoFind.Filters = append(memoFind.Filters, request.Filter)
  }

  // 权限过滤：基于用户身份
  if currentUser == nil {
    // 未登录用户只能看公开笔记
    memoFind.VisibilityList = []store.Visibility{store.Public}
  } else {
    if memoFind.CreatorID == nil {
      // 登录用户可以看自己的 + 公开/保护的
      filter := fmt.Sprintf(
        `creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, 
        currentUser.ID
      )
      memoFind.Filters = append(memoFind.Filters, filter)
    } else if *memoFind.CreatorID != currentUser.ID {
      // 查看他人笔记时，只能看公开/保护的
      memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
    }
  }

  // 分页处理
  var limit, offset int
  if request.PageToken != "" {
    var pageToken v1pb.PageToken
    if err := unmarshalPageToken(request.PageToken, &pageToken); err != nil {
      return nil, status.Errorf(codes.InvalidArgument, "invalid page token: %v", err)
    }
    limit = normalizePageSize(pageToken.Limit)
    offset = int(pageToken.Offset)
  } else {
    limit = normalizePageSize(request.PageSize)
  }
  limit = min(limit, MaxPageSize)
  limitPlusOne := limit + 1  // 多取一条判断是否有下一页
  memoFind.Limit = &limitPlusOne
  memoFind.Offset = &offset

  // 执行查询
  memos, err := s.Store.ListMemos(ctx, memoFind)
  // ... 后续处理（加载关联数据、转换格式等）
}
```

#### 5.1.2 过滤器验证

**文件**: `server/router/api/v1/shortcut_service.go` (validateFilter 函数)

```go
func (s *APIV1Service) validateFilter(ctx context.Context, filterStr string) error {
  if filterStr == "" {
    return errors.New("filter cannot be empty")
  }

  // 获取默认 CEL 引擎
  engine, err := filter.DefaultEngine()
  if err != nil {
    return err
  }

  // 根据数据库驱动选择方言
  var dialect filter.DialectName
  switch s.Profile.Driver {
  case "mysql":
    dialect = filter.DialectMySQL
  case "postgres":
    dialect = filter.DialectPostgres
  default:
    dialect = filter.DialectSQLite
  }

  // 尝试编译为 SQL，验证语法正确性
  if _, err := engine.CompileToStatement(
    ctx, 
    filterStr, 
    filter.RenderOptions{Dialect: dialect}
  ); err != nil {
    return errors.Wrap(err, "failed to compile filter")
  }
  return nil
}
```

**安全考虑**：
- 在 API 层验证过滤器语法，防止恶意输入
- 编译失败直接返回错误，不传递到存储层
- 方言选择基于实际数据库驱动，确保兼容性

### 5.2 CEL 过滤器引擎

#### 5.2.1 引擎架构

**文件**: `internal/filter/engine.go`

使用 Google CEL (Common Expression Language) 作为统一查询语言：

```go
// Engine 解析 CEL 过滤器为与方言无关的条件树
type Engine struct {
  schema Schema
  env    *cel.Env
}

// NewEngine 为指定 schema 构建引擎
func NewEngine(schema Schema) (*Engine, error) {
  env, err := cel.NewEnv(schema.EnvOptions...)
  if err != nil {
    return nil, errors.Wrap(err, "failed to create CEL environment")
  }
  return &Engine{
    schema: schema,
    env:    env,
  }, nil
}

// CompileToStatement 一步完成编译和渲染
func (e *Engine) CompileToStatement(
  ctx context.Context, 
  filter string, 
  opts RenderOptions
) (Statement, error) {
  program, err := e.Compile(ctx, filter)
  if err != nil {
    return Statement{}, err
  }
  return program.Render(opts)
}
```

#### 5.2.2 Schema 定义

**文件**: `internal/filter/schema.go`

定义了 CEL 环境中可用的字段和类型：

```go
func NewSchema() Schema {
  fields := map[string]Field{
    // 基础标量字段
    "content": {
      Name:             "content",
      Kind:             FieldKindScalar,
      Type:             FieldTypeString,
      Column:           Column{Table: "memo", Name: "content"},
      SupportsContains: true,  // 支持 contains() 方法
    },
    "creator_id": {
      Name:        "creator_id",
      Kind:        FieldKindScalar,
      Type:        FieldTypeInt,
      Column:      Column{Table: "memo", Name: "creator_id"},
    },
    "created_ts": {
      Name:   "created_ts",
      Kind:   FieldKindScalar,
      Type:   FieldTypeTimestamp,
      Column: Column{Table: "memo", Name: "created_ts"},
    },
    "pinned": {
      Name:        "pinned",
      Kind:        FieldKindBoolColumn,
      Type:        FieldTypeBool,
      Column:      Column{Table: "memo", Name: "pinned"},
    },
    "visibility": {
      Name:        "visibility",
      Kind:        FieldKindScalar,
      Type:        FieldTypeString,
      Column:      Column{Table: "memo", Name: "visibility"},
    },
    
    // JSON 列表字段（标签）
    "tags": {
      Name:     "tags",
      Kind:     FieldKindJSONList,
      Type:     FieldTypeString,
      Column:   Column{Table: "memo", Name: "payload"},
      JSONPath: []string{"tags"},  // payload.tags
    },
    "tag": {
      Name:     "tag",
      Kind:     FieldKindVirtualAlias,  // 虚拟别名
      Type:     FieldTypeString,
      AliasFor: "tags",
    },
    
    // JSON 布尔字段（属性）
    "has_task_list": {
      Name:     "has_task_list",
      Kind:     FieldKindJSONBool,
      Type:     FieldTypeBool,
      Column:   Column{Table: "memo", Name: "payload"},
      JSONPath: []string{"property", "hasTaskList"},  // payload.property.hasTaskList
    },
    "has_link": {
      Name:     "has_link",
      Kind:     FieldKindJSONBool,
      Type:     FieldTypeBool,
      Column:   Column{Table: "memo", Name: "payload"},
      JSONPath: []string{"property", "hasLink"},
    },
    "has_code": {
      Name:     "has_code",
      Kind:     FieldKindJSONBool,
      Type:     FieldTypeBool,
      Column:   Column{Table: "memo", Name: "payload"},
      JSONPath: []string{"property", "hasCode"},
    },
    "has_incomplete_tasks": {
      Name:     "has_incomplete_tasks",
      Kind:     FieldKindJSONBool,
      Type:     FieldTypeBool,
      Column:   Column{Table: "memo", Name: "payload"},
      JSONPath: []string{"property", "hasIncompleteTasks"},
    },
  }

  // CEL 环境选项
  envOptions := []cel.EnvOption{
    cel.Variable("content", cel.StringType),
    cel.Variable("creator", cel.StringType),
    cel.Variable("creator_id", cel.IntType),
    cel.Variable("created_ts", cel.IntType),
    cel.Variable("updated_ts", cel.IntType),
    cel.Variable("pinned", cel.BoolType),
    cel.Variable("tag", cel.StringType),
    cel.Variable("tags", cel.ListType(cel.StringType)),
    cel.Variable("visibility", cel.StringType),
    cel.Variable("has_task_list", cel.BoolType),
    cel.Variable("has_link", cel.BoolType),
    cel.Variable("has_code", cel.BoolType),
    cel.Variable("has_incomplete_tasks", cel.BoolType),
    nowFunction,  // now() 函数返回当前时间戳
  }

  return Schema{
    Name:       "memo",
    Fields:     fields,
    EnvOptions: envOptions,
  }
}
```

#### 5.2.3 字段类型对照表

| 字段名 | CEL 类型 | 存储类型 | SQL 列 | JSON Path |
|--------|----------|----------|--------|-----------|
| `content` | string | Scalar | `memo.content` | - |
| `creator_id` | int | Scalar | `memo.creator_id` | - |
| `created_ts` | int | Scalar | `memo.created_ts` | - |
| `pinned` | bool | BoolColumn | `memo.pinned` | - |
| `visibility` | string | Scalar | `memo.visibility` | - |
| `tag` | string | VirtualAlias | - | 别名指向 `tags` |
| `tags` | list(string) | JSONList | `memo.payload` | `$.tags` |
| `has_task_list` | bool | JSONBool | `memo.payload` | `$.property.hasTaskList` |
| `has_link` | bool | JSONBool | `memo.payload` | `$.property.hasLink` |
| `has_code` | bool | JSONBool | `memo.payload` | `$.property.hasCode` |

### 5.3 SQL 渲染层

#### 5.3.1 多数据库方言支持

**文件**: `internal/filter/render.go`

根据不同数据库生成适配的 SQL：

```go
// 渲染 contains 条件（全文检索）
func (r *renderer) renderContainsCondition(cond *ContainsCondition) (renderResult, error) {
  field, ok := r.schema.Field(cond.Field)
  if !ok {
    return renderResult{}, errors.Errorf("unknown field %q", cond.Field)
  }
  column := field.columnExpr(r.dialect)
  arg := fmt.Sprintf("%%%s%%", cond.Value)
  
  switch r.dialect {
  case DialectSQLite:
    // 使用自定义 Unicode 感知的大小写折叠函数
    sql := fmt.Sprintf(
      "memos_unicode_lower(%s) LIKE memos_unicode_lower(%s)", 
      column, 
      r.addArg(arg)
    )
    return renderResult{sql: sql}, nil
  case DialectPostgres:
    // PostgreSQL 内置 ILIKE 支持大小写不敏感
    sql := fmt.Sprintf("%s ILIKE %s", column, r.addArg(arg))
    return renderResult{sql: sql}, nil
  default:
    // MySQL 默认 LIKE
    sql := fmt.Sprintf("%s LIKE %s", column, r.addArg(arg))
    return renderResult{sql: sql}, nil
  }
}
```

#### 5.3.2 标签查询渲染（含层级支持）

```go
func (r *renderer) renderTagInList(values []ValueExpr) (renderResult, error) {
  field, ok := r.schema.ResolveAlias("tag")
  // ...
  
  conditions := make([]string, 0, len(values))
  for _, v := range values {
    str, ok := lit.(string)
    // ...
    
    switch r.dialect {
    case DialectSQLite:
      // 精确匹配 + 层级前缀匹配
      exactMatch := fmt.Sprintf(
        "%s LIKE %s", 
        jsonArrayExpr(r.dialect, field), 
        r.addArg(fmt.Sprintf(`%%"%s"%%`, str))
      )
      prefixMatch := fmt.Sprintf(
        "%s LIKE %s", 
        jsonArrayExpr(r.dialect, field), 
        r.addArg(fmt.Sprintf(`%%"%s/%%`, str))
      )
      expr := fmt.Sprintf("(%s OR %s)", exactMatch, prefixMatch)
      conditions = append(conditions, expr)
      
    case DialectMySQL:
      exactMatch := fmt.Sprintf(
        "JSON_CONTAINS(%s, %s)", 
        jsonArrayExpr(r.dialect, field), 
        r.addArg(fmt.Sprintf(`"%s"`, str))
      )
      prefixMatch := fmt.Sprintf(
        "%s LIKE %s", 
        jsonArrayExpr(r.dialect, field), 
        r.addArg(fmt.Sprintf(`%%"%s/%%`, str))
      )
      expr := fmt.Sprintf("(%s OR %s)", exactMatch, prefixMatch)
      conditions = append(conditions, expr)
      
    case DialectPostgres:
      exactMatch := fmt.Sprintf(
        "%s @> jsonb_build_array(%s::json)", 
        jsonArrayExpr(r.dialect, field), 
        r.addArg(fmt.Sprintf(`"%s"`, str))
      )
      prefixMatch := fmt.Sprintf(
        "(%s)::text LIKE %s", 
        jsonArrayExpr(r.dialect, field), 
        r.addArg(fmt.Sprintf(`%%"%s/%%`, str))
      )
      expr := fmt.Sprintf("(%s OR %s)", exactMatch, prefixMatch)
      conditions = append(conditions, expr)
    }
  }
  // ...
}
```

### 5.4 存储层集成

#### 5.4.1 SQLite 驱动的 ListMemos

**文件**: `store/db/sqlite/memo.go`

```go
func (d *DB) ListMemos(ctx context.Context, find *store.FindMemo) ([]*store.Memo, error) {
  where, args := []string{"1 = 1"}, []any{}

  // 关键：编译 CEL 过滤器为 SQL 条件
  engine, err := filter.DefaultEngine()
  if err != nil {
    return nil, err
  }
  if err := filter.AppendConditions(
    ctx, 
    engine, 
    find.Filters, 
    filter.DialectSQLite, 
    &where, 
    &args
  ); err != nil {
    return nil, err
  }

  // 其他条件（ID、UID、CreatorID 等）
  if v := find.ID; v != nil {
    where, args = append(where, "`memo`.`id` = ?"), append(args, *v)
  }
  if len(find.IDList) > 0 {
    // ... IN 条件
  }
  if v := find.UID; v != nil {
    where, args = append(where, "`memo`.`uid` = ?"), append(args, *v)
  }
  // ... 更多条件

  // 排序
  order := "DESC"
  if find.OrderByTimeAsc {
    order = "ASC"
  }
  orderBy := []string{}
  if find.OrderByPinned {
    orderBy = append(orderBy, "`pinned` DESC")
  }
  if find.OrderByUpdatedTs {
    orderBy = append(orderBy, "`updated_ts` "+order)
  } else {
    orderBy = append(orderBy, "`created_ts` "+order)
  }
  orderBy = append(orderBy, "`id` DESC")  // 最终排序键

  // 构建查询字段
  fields := []string{
    "`memo`.`id` AS `id`",
    "`memo`.`uid` AS `uid`",
    "`memo`.`creator_id` AS `creator_id`",
    "`memo`.`created_ts` AS `created_ts`",
    "`memo`.`updated_ts` AS `updated_ts`",
    "`memo`.`row_status` AS `row_status`",
    "`memo`.`visibility` AS `visibility`",
    "`memo`.`pinned` AS `pinned`",
    "`memo`.`payload` AS `payload`",
    "CASE WHEN `parent_memo`.`uid` IS NOT NULL THEN `parent_memo`.`uid` ELSE NULL END AS `parent_uid`",
  }
  if !find.ExcludeContent {
    fields = append(fields, "`memo`.`content` AS `content`")
  }

  // 最终 SQL
  query := "SELECT " + strings.Join(fields, ", ") + "FROM `memo` " +
    "LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id` " +
    "LEFT JOIN `memo_relation` ON `memo`.`id` = `memo_relation`.`memo_id` AND `memo_relation`.`type` = \"COMMENT\" " +
    "LEFT JOIN `memo` AS `parent_memo` ON `memo_relation`.`related_memo_id` = `parent_memo`.`id` " +
    "WHERE " + strings.Join(where, " AND ") + " " +
    "ORDER BY " + strings.Join(orderBy, ", ")
  
  // 分页
  if find.Limit != nil {
    query = fmt.Sprintf("%s LIMIT %d", query, *find.Limit)
    if find.Offset != nil {
      query = fmt.Sprintf("%s OFFSET %d", query, *find.Offset)
    }
  }

  // 执行查询
  rows, err := d.db.QueryContext(ctx, query, args...)
  // ... 扫描结果
}
```

#### 5.4.2 AppendConditions 辅助函数

**文件**: `internal/filter/helpers.go`

```go
func AppendConditions(
  ctx context.Context, 
  engine *Engine, 
  filters []string, 
  dialect DialectName, 
  where *[]string, 
  args *[]any
) error {
  for _, filterStr := range filters {
    stmt, err := engine.CompileToStatement(ctx, filterStr, RenderOptions{
      Dialect:           dialect,
      PlaceholderOffset: len(*args),  // 正确处理参数索引
    })
    if err != nil {
      return err
    }
    if stmt.SQL == "" {
      continue  // 空条件跳过
    }
    *where = append(*where, fmt.Sprintf("(%s)", stmt.SQL))
    *args = append(*args, stmt.Args...)
  }
  return nil
}
```

---

## 六、完整数据流转示例

### 6.1 场景：用户搜索标签为 "work" 且内容包含 "meeting" 的笔记

#### 步骤 1：用户交互

1. 用户在 **TagsSection** 点击标签 "work"
2. 用户在 **SearchBar** 输入 "meeting" 并按回车

#### 步骤 2：前端状态更新

**MemoFilterContext** 中的 `filters` 变为：
```typescript
[
  { factor: "tagSearch", value: "work" },
  { factor: "contentSearch", value: "meeting" }
]
```

URL 同步为：
```
?filter=tagSearch:work,contentSearch:meeting
```

#### 步骤 3：CEL 表达式构建

**useMemoFilters** 转换为：
```
tag in ["work"] && content.contains("meeting")
```

#### 步骤 4：API 调用

**ListMemosRequest**：
```protobuf
{
  filter: "tag in [\"work\"] && content.contains(\"meeting\")",
  page_size: 20,
  order_by: "create_time desc"
}
```

#### 步骤 5：后端验证与编译

**validateFilter** 使用 CEL 引擎验证语法。

**SQL 渲染（SQLite 方言）**：
```sql
-- tag in ["work"] 的渲染（含层级支持）
(
  JSON_EXTRACT(`memo`.`payload`, '$.tags') LIKE '%"work"%' 
  OR 
  JSON_EXTRACT(`memo`.`payload`, '$.tags') LIKE '%"work/%'
)
AND
-- content.contains("meeting") 的渲染
memos_unicode_lower(`memo`.`content`) LIKE memos_unicode_lower('%meeting%')
```

#### 步骤 6：最终 SQL 查询

```sql
SELECT 
  `memo`.`id` AS `id`,
  `memo`.`uid` AS `uid`,
  `memo`.`creator_id` AS `creator_id`,
  `memo`.`created_ts` AS `created_ts`,
  `memo`.`updated_ts` AS `updated_ts`,
  `memo`.`row_status` AS `row_status`,
  `memo`.`visibility` AS `visibility`,
  `memo`.`pinned` AS `pinned`,
  `memo`.`payload` AS `payload`,
  `memo`.`content` AS `content`
FROM `memo` 
LEFT JOIN `user` AS `memo_creator` ON `memo`.`creator_id` = `memo_creator`.`id` 
-- ... 其他 JOIN
WHERE 
  1 = 1
  AND (
    (JSON_EXTRACT(`memo`.`payload`, '$.tags') LIKE '%"work"%' 
     OR JSON_EXTRACT(`memo`.`payload`, '$.tags') LIKE '%"work/%')
    AND memos_unicode_lower(`memo`.`content`) LIKE memos_unicode_lower('%meeting%')
  )
  AND (creator_id == 123 || visibility in ["PUBLIC", "PROTECTED"])
ORDER BY `created_ts` DESC, `id` DESC
LIMIT 21
```

#### 步骤 7：前端渲染

**PagedMemoList** 收到数据后：
1. 扁平化分页数据
2. 应用自定义排序
3. 渲染 **MemoView** 组件列表
4. 显示 **MemoFilters** 标签（"work" 和 "meeting"）

---

## 七、技术亮点与设计考量

### 7.1 前后端协作的核心设计

#### 1. 状态与 URL 双向同步

```typescript
// MemoFilterContext 使用 useSearchParams 实现
useEffect(() => {
  // URL 变化 → 状态同步
  const filterParam = searchParams.get("filter") || "";
  if (filterParam !== lastSyncedUrlRef.current) {
    const newFilters = parseFilterQuery(filterParam);
    setFiltersState(newFilters);
  }
}, [searchParams]);

useEffect(() => {
  // 状态变化 → URL 同步
  const storeString = stringifyFilters(filters);
  if (storeString !== lastSyncedStoreRef.current) {
    const newParams = new URLSearchParams(searchParams);
    if (filters.length > 0) {
      newParams.set("filter", storeString);
    } else {
      newParams.delete("filter");
    }
    setSearchParams(newParams, { replace: true });
  }
}, [filters, searchParams, setSearchParams]);
```

**优势**：
- 刷新页面后状态保留
- 可分享链接
- 浏览器前进/后退正常工作

#### 2. CEL 作为中间查询语言

| 优势 | 说明 |
|------|------|
| **跨数据库** | 一套表达式，SQLite/MySQL/PostgreSQL 各自渲染 |
| **类型安全** | CEL 编译时检查类型错误 |
| **防止注入** | 参数化查询，SQL 注入风险低 |
| **表达能力强** | 支持逻辑运算、比较、函数调用 |
| **前端友好** | 语法类似 JavaScript/TypeScript |

#### 3. 分层职责清晰

```
┌─────────────────────────────────────────────────────────────┐
│  前端层                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐│
│  │ UI 组件      │→│ Context 状态 │→│ useMemoFilters 转换  ││
│  │ (点击/输入)  │  │ (FilterFactor)│  │ (→ CEL 表达式)       ││
│  └─────────────┘  └─────────────┘  └─────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  API 层                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ListMemos (gRPC)                                    │   │
│  │  - validateFilter: CEL 语法验证                      │   │
│  │  - 权限过滤: 基于用户身份添加额外条件                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  存储层                                                      │
│  ┌─────────────┐  ┌─────────────────────────────────────┐  │
│  │  Driver     │→│  internal/filter                    │  │
│  │  (SQLite等) │  │  CEL → SQL 编译 + 参数化查询        │  │
│  └─────────────┘  └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 标签系统的特殊设计

#### 1. 层级标签支持

```go
// 搜索 "work" 时匹配:
// - 精确: ["work"]
// - 层级: ["work/project"], ["work/project/task"]

// SQLite 实现
exactMatch := fmt.Sprintf("%s LIKE %s", jsonExpr, `%"work"%`)
prefixMatch := fmt.Sprintf("%s LIKE %s", jsonExpr, `%"work/%`)
expr := fmt.Sprintf("(%s OR %s)", exactMatch, prefixMatch)
```

#### 2. JSON 字段的高效查询

标签和属性存储在 `memo.payload` JSON 列中：

| 字段 | JSON Path | 查询方式 |
|------|-----------|----------|
| 标签 | `$.tags` | JSON 数组匹配 + 前缀 LIKE |
| hasLink | `$.property.hasLink` | JSON 布尔提取 |
| hasTaskList | `$.property.hasTaskList` | JSON 布尔提取 |

### 7.3 性能考量

#### 1. 无限滚动 + 游标分页

```typescript
// useInfiniteMemos 使用游标而非 offset
getNextPageParam: (lastPage) => lastPage.nextPageToken || undefined
```

**优势**：
- 避免 offset 越深越慢的问题
- 数据变化时分页稳定

#### 2. React Query 缓存策略

```typescript
staleTime: 1000 * 60,      // 1 分钟内视为新鲜
gcTime: 1000 * 60 * 5,      // 5 分钟后回收
```

**优势**：
- 减少重复请求
- 用户体验流畅

#### 3. 数据库层面

- **参数化查询**：防止 SQL 注入，提升计划缓存命中率
- **多列排序**：`created_ts DESC, id DESC` 确保稳定性
- **预加载关联**：批量加载 reactions、attachments、relations 避免 N+1

---

## 八、关键文件索引

### 前端文件

| 文件路径 | 职责 |
|----------|------|
| `web/src/contexts/MemoFilterContext.tsx` | 过滤器状态管理 + URL 同步 |
| `web/src/hooks/useMemoFilters.ts` | FilterFactor → CEL 转换 |
| `web/src/hooks/useMemoQueries.ts` | React Query 封装 + API 调用 |
| `web/src/components/MemoExplorer/TagsSection.tsx` | 标签选择 UI |
| `web/src/components/SearchBar.tsx` | 搜索输入 UI |
| `web/src/components/MemoFilters.tsx` | 激活过滤器显示 |
| `web/src/components/PagedMemoList/PagedMemoList.tsx` | 列表渲染 + 无限滚动 |
| `web/src/pages/Home.tsx` | 页面组装示例 |

### 后端文件

| 文件路径 | 职责 |
|----------|------|
| `server/router/api/v1/memo_service.go` | ListMemos API 实现 |
| `server/router/api/v1/shortcut_service.go` | validateFilter 实现 |
| `store/memo.go` | Store 层接口定义 |
| `store/db/sqlite/memo.go` | SQLite 驱动实现 |
| `internal/filter/engine.go` | CEL 引擎封装 |
| `internal/filter/parser.go` | CEL AST → 条件树解析 |
| `internal/filter/schema.go` | 字段定义 + CEL 环境 |
| `internal/filter/render.go` | 条件树 → SQL 渲染 |
| `internal/filter/helpers.go` | AppendConditions 工具函数 |

---

## 九、总结

Memos 的标签、全文检索和多条件过滤系统展现了一个**精心设计的前后端协作架构**：

1. **前端状态管理**：使用 React Context + useSearchParams 实现过滤器状态与 URL 的双向同步，确保用户体验和可分享性。

2. **查询语言抽象**：引入 CEL (Common Expression Language) 作为中间查询语言，实现了：
   - 前端友好的语法
   - 类型安全的编译期检查
   - 跨数据库的统一表达

3. **分层编译架构**：
   - 前端：`FilterFactor` → `CEL 表达式`
   - 后端：`CEL 表达式` → `SQL 条件`（按数据库方言渲染）
   - 存储层：`SQL 条件` → `参数化查询`

4. **标签系统特性**：
   - 支持层级标签（`work/project`）
   - 点击标签自动切换过滤
   - 树形视图和平面视图切换

5. **性能优化**：
   - 无限滚动 + 游标分页
   - React Query 缓存策略
   - 批量关联数据加载
   - 参数化 SQL 查询

这种设计使得：
- **用户体验流畅**：即时反馈、状态保留、可分享链接
- **开发体验良好**：类型安全、职责清晰、易于扩展
- **系统性能优秀**：缓存利用、批量操作、参数化查询
- **多数据库支持**：CEL 抽象层隔离了数据库差异
