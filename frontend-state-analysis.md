# Memos 前端状态管理分析报告

## 一、架构总览

Memos 前端采用**分层混合状态管理架构**，核心分为三个层次：

```
┌─────────────────────────────────────────────────────────────────┐
│                    服务端状态层 (Server State)                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  React Query v5                                           │  │
│  │  • Query Client: 全局缓存配置 (query-client.ts)            │  │
│  │  • Custom Hooks: 数据查询 + 乐观更新 (useMemoQueries.ts)   │  │
│  │  • SSE 同步: 实时缓存失效 (useLiveMemoRefresh.ts)          │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                    全局 UI 状态层 (Global UI State)             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  React Context Provider 树                                │  │
│  │  • 顶层 (main.tsx): Instance / Auth / View                │  │
│  │  • 路由层 (App.tsx): MemoFilter                           │  │
│  │  • 组件层: EditorContext 等                               │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                    本地状态层 (Local State)                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  独立模块 + localStorage                                 │  │
│  │  • auth-state.ts: Token 三层缓存                          │  │
│  │  • 组件 useState/useReducer: 临时 UI 状态                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、React Context 挂载层级（已修正）

### 2.1 实际层级结构

**关键发现**: Context 分两层挂载，而非全部在 main.tsx。

**证据1: main.tsx 顶层 Provider** (`web/src/main.tsx:60-78`)

```tsx
function Main() {
  return (
    <ErrorBoundary>
      <QueryClientProvider client={queryClient}>       {/* 最外层 */}
        <InstanceProvider>                             {/* 1. 实例配置 */}
          <AuthProvider>                               {/* 2. 认证状态 */}
            <ViewProvider>                             {/* 3. 视图设置 */}
              <AppInitializer>
                <RouterProvider router={router} />     {/* 路由入口 */}
                <Toaster />
              </AppInitializer>
            </ViewProvider>
          </AuthProvider>
        </InstanceProvider>
        <ReactQueryDevtools />
      </QueryClientProvider>
    </ErrorBoundary>
  );
}
```

**证据2: router/index.tsx 路由配置** (`web/src/router/index.tsx:52-112`)

```tsx
export const routeConfig: RouteObject[] = [
  {
    path: "/",
    element: <App />,           // 根路径渲染 App 组件
    children: [
      { path: Routes.AUTH, ... },
      { path: Routes.ENTRY, ... },
      // ... 其他子路由
    ],
  },
];
```

**证据3: App.tsx 路由层 Provider** (`web/src/App.tsx:59-63`)

```tsx
const App = () => {
  // ... 初始化逻辑
  
  return (
    <MemoFilterProvider>         {/* 4. 过滤器状态 —— 在路由层挂载 */}
      <Outlet />                 {/* 子路由出口 */}
    </MemoFilterProvider>
  );
};
```

### 2.2 完整挂载顺序

```
DOM Root
  │
  ▼
QueryClientProvider (main.tsx)
  │
  ├─► InstanceProvider (main.tsx)
  │    └─► AuthProvider (main.tsx)
  │         └─► ViewProvider (main.tsx)
  │              └─► AppInitializer
  │                   └─► RouterProvider
  │                        │
  │                        ▼
  │                     App 组件 (路由层)
  │                          │
  │                          ▼
  │                     MemoFilterProvider (App.tsx)
  │                          │
  │                          ▼
  │                        Outlet
  │                          │
  │                    ┌─────┴─────┐
  │                    │           │
  │                    ▼           ▼
  │                认证页面      主应用页面
  │                (SignIn等)   (Home, Explore等)
  │
  └─► ReactQueryDevtools
```

### 2.3 各 Context 职责与挂载位置

| Context | 挂载位置 | 数据来源 | 生命周期 |
|---------|---------|---------|---------|
| **QueryClientProvider** | main.tsx 最外层 | React Query 内部 | 应用全程 |
| **InstanceProvider** | main.tsx | API (instance profile/settings) | 应用全程 |
| **AuthProvider** | main.tsx | localStorage + API | 应用全程 |
| **ViewProvider** | main.tsx | localStorage | 应用全程 |
| **MemoFilterProvider** | App.tsx 路由层 | URL search params | 路由内共享（认证页不可用） |

**MemoFilterProvider 挂载在路由层的设计意图**:
- 认证页面（/auth/signin 等）不需要过滤器状态
- 过滤器状态与 URL 绑定，路由切换时自然重置
- 避免认证流程中不必要的 Provider 开销

---

## 三、乐观更新（核心层定位）

### 3.1 结论：乐观更新完全在 React Query Hooks 层实现

**关键证据**: 乐观更新逻辑 100% 集中在 `web/src/hooks/useMemoQueries.ts` 的 `useUpdateMemo` mutation 中。

### 3.2 核心实现代码

**文件**: `web/src/hooks/useMemoQueries.ts:196-250`

```typescript
export function useUpdateMemo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ update, updateMask }) => {
      return memoServiceClient.updateMemo({
        memo: create(MemoSchema, update as Record<string, unknown>),
        updateMask: create(FieldMaskSchema, { paths: updateMask }),
      });
    },

    // ─────────────────────────────────────────────────────────────
    // 阶段 1: 乐观更新 (onMutate) —— API 请求发送前执行
    // ─────────────────────────────────────────────────────────────
    onMutate: async ({ update }) => {
      if (!update.name) return { previousMemo: undefined };

      // 1. 取消所有正在进行的 memo 查询，防止竞态条件
      await queryClient.cancelQueries({ queryKey: memoKeys.all });

      // 2. 保存当前状态快照（用于失败回滚）
      const previousMemo =
        queryClient.getQueryData<Memo>(memoKeys.detail(update.name)) || 
        findMemoInCollectionQueries(queryClient, update.name);
      
      const memoPatch: MemoPatch = { ...update, name: update.name };

      // 3. 乐观更新详情缓存
      if (previousMemo) {
        queryClient.setQueryData(
          memoKeys.detail(update.name), 
          { ...previousMemo, ...memoPatch }
        );
      }

      // 4. 乐观更新所有列表缓存（关键！）
      patchMemoInCollectionQueries(queryClient, memoPatch);

      return { previousMemo };
    },

    // ─────────────────────────────────────────────────────────────
    // 阶段 2: 失败回滚 (onError)
    // ─────────────────────────────────────────────────────────────
    onError: (_err, { update }, context) => {
      if (context?.previousMemo && update.name) {
        // 有快照 → 精确回滚
        queryClient.setQueryData(memoKeys.detail(update.name), context.previousMemo);
        patchMemoInCollectionQueries(queryClient, context.previousMemo);
      } else {
        // 无快照 → 全量失效
        queryClient.invalidateQueries({ queryKey: memoKeys.all });
      }
    },

    // ─────────────────────────────────────────────────────────────
    // 阶段 3: 成功确认 (onSuccess)
    // ─────────────────────────────────────────────────────────────
    onSuccess: (updatedMemo) => {
      // 用服务端返回的数据同步缓存（可能包含服务端生成的字段）
      queryClient.setQueryData(memoKeys.detail(updatedMemo.name), updatedMemo);
      patchMemoInCollectionQueries(queryClient, updatedMemo);
      
      // 触发列表重取，确保排序/筛选正确
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      
      // 关联数据失效
      if (updatedMemo.parent) {
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(updatedMemo.parent) });
      }
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });
    },
  });
}
```

### 3.3 列表缓存修补机制（乐观更新的关键）

**文件**: `web/src/hooks/useMemoQueries.ts:32-109`

`patchMemoInCollectionQueries` 函数实现了对所有 memo 查询的批量更新：

```typescript
function patchMemoInCollectionQueries(
  queryClient: ReturnType<typeof useQueryClient>, 
  update: MemoPatch
) {
  // 遍历所有以 memoKeys.all 为前缀的查询
  queryClient.setQueriesData<MemoCollectionQueryData>(
    { queryKey: memoKeys.all }, 
    (data) => patchMemoListQueryData(data, update)
  );
}

function patchMemoListQueryData<T>(data: T | undefined, update: MemoPatch): T | undefined {
  if (!data) return data;

  // 支持普通列表查询
  if (isMemoListResponse(data)) {
    return patchMemoListResponse(data, update) as T;
  }

  // 支持无限滚动查询（InfiniteData 结构）
  if (isInfiniteMemoListData(data)) {
    let changed = false;
    const pages = data.pages.map((page) => {
      const patchedPage = patchMemoListResponse(page, update);
      if (patchedPage !== page) changed = true;
      return patchedPage;
    });
    return (changed ? { ...data, pages } : data) as T;
  }

  return data;
}
```

**关键点**:
- 使用 `setQueriesData` 批量更新所有 memo 相关查询
- 同时支持普通 `ListMemosResponse` 和无限滚动 `InfiniteData<ListMemosResponse>`
- 深度遍历所有 pages，找到并更新匹配的 memo

### 3.4 调用链（哪些地方触发乐观更新）

乐观更新只通过 `useUpdateMemo` hook 暴露，以下组件调用它：

| 调用方 | 文件 | 用途 |
|-------|------|------|
| **useMemoActions** | `components/MemoView/hooks/useMemoActions.ts:5` | 取消置顶 |
| **useMemoActionHandlers** | `components/MemoActionMenu/hooks.ts:28` | 切换置顶、归档/恢复 |
| **TaskListItem** | `components/MemoContent/TaskListItem.tsx:17` | 勾选任务列表项 |

**示例调用**: `web/src/components/MemoContent/TaskListItem.tsx:19-66`

```typescript
export const TaskListItem: React.FC<TaskListItemProps> = ({ checked, ...props }) => {
  const { memo } = useMemoViewContext();
  const { readonly } = useMemoViewDerived();
  const { mutate: updateMemo } = useUpdateMemo();  // 获取 mutation

  const handleChange = async (newChecked: boolean) => {
    if (readonly || !memo) return;
    
    // 计算新内容
    const newContent = toggleTaskAtIndex(memo.content, taskIndex, newChecked);
    
    // 调用 mutation —— 自动触发乐观更新
    updateMemo({
      update: { name: memo.name, content: newContent },
      updateMask: ["content", "update_time"],
    });
  };

  return <Checkbox checked={checked} onCheckedChange={handleChange} />;
};
```

**调用方无需关心乐观更新逻辑**，这是 `useUpdateMemo` hook 内部封装的实现细节。

### 3.5 乐观更新流程图

```
用户操作（勾选任务/切换置顶）
           │
           ▼
┌─────────────────────────────────────────┐
│  TaskListItem / useMemoActionHandlers   │
│  调用 updateMemo.mutate()               │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  useUpdateMemo (React Query Mutation)   │  ← 乐观更新实现层
└───────────────┬─────────────────────────┘
                │
    ┌───────────┴───────────┐
    │                       │
    ▼                       ▼
 onMutate               API 请求
    │                       │
    │                 ┌─────┴─────┐
    │                 │           │
    │                 ▼           ▼
    │              成功          失败
    │                 │           │
    ▼                 ▼           ▼
┌─────────┐      ┌─────────┐  ┌─────────┐
│取消请求  │      │onSuccess│  │ onError │
│保存快照  │      │同步缓存  │  │ 回滚缓存 │
│乐观更新  │      │失效列表  │  │或全量失效│
└────┬────┘      └─────────┘  └─────────┘
     │
     ▼
 UI 立即响应（用户看到变化）
```

### 3.6 乐观更新适用边界

**有乐观更新**:
- `useUpdateMemo` —— 更新操作，数据变化可预测

**无乐观更新**:
- `useCreateMemo` (`useMemoQueries.ts:177-194`) —— 缺少服务端生成的 name，只能在 `onSuccess` 后 `setQueryData`
- `useDeleteMemo` (`useMemoQueries.ts:252-269`) —— 直接 `invalidateQueries`，更简单直接

---

## 四、列表缓存（三层结构）

### 4.1 结论：列表缓存在 React Query 层实现，通过三层机制保持一致性

列表缓存不是在单个组件中实现，而是一个分层系统：

```
┌──────────────────────────────────────────────────────────────┐
│                    层 1: 数据获取与缓存                       │
│  web/src/hooks/useMemoQueries.ts                            │
│  • useInfiniteMemos(): 无限滚动查询                          │
│  • useMemos(): 普通列表查询                                  │
│  • memoKeys: 缓存键工厂                                      │
│  • staleTime/gcTime: 缓存时效配置                           │
└──────────────────────┬───────────────────────────────────────┘
                       │ 读取/写入
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    层 2: 缓存失效与同步                       │
│  web/src/hooks/useLiveMemoRefresh.ts (SSE)                  │
│  • 监听服务端事件，跨标签页/设备实时失效                      │
│  • useUpdateMemo 等 mutation 的 onSuccess                   │
│  • 重连补偿：断线重连后刷新活动视图                           │
└──────────────────────┬───────────────────────────────────────┘
                       │ 失效后触发重取
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    层 3: UI 消费层                           │
│  web/src/components/PagedMemoList/PagedMemoList.tsx         │
│  • 调用 useInfiniteMemos 获取数据                            │
│  • 扁平化 pages 为 memos 数组                                │
│  • 无限滚动 + 自动预取                                       │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 层 1: 数据获取与缓存（核心）

**文件**: `web/src/hooks/useMemoQueries.ts:111-139`

```typescript
export function useMemos(request: Partial<ListMemosRequest> = {}) {
  return useQuery({
    queryKey: memoKeys.list(request),          // 缓存键包含过滤条件
    queryFn: async () => memoServiceClient.listMemos(...),
    // 默认使用 query-client.ts 的全局配置: staleTime=30s, gcTime=5min
  });
}

export function useInfiniteMemos(
  request: Partial<ListMemosRequest> = {}, 
  options?: { enabled?: boolean }
) {
  return useInfiniteQuery({
    queryKey: memoKeys.list(request),
    queryFn: async ({ pageParam }) => memoServiceClient.listMemos({
      ...request,
      pageToken: pageParam || "",
    }),
    initialPageParam: "",
    getNextPageParam: (lastPage) => lastPage.nextPageToken || undefined,
    staleTime: 1000 * 60,           // 覆盖为 60 秒
    gcTime: 1000 * 60 * 5,          // 5 分钟
    enabled: options?.enabled ?? true,
  });
}
```

**缓存键结构** (`useMemoQueries.ts:11-19`):

```typescript
export const memoKeys = {
  all: ["memos"] as const,                                    // 最顶层
  lists: () => [...memoKeys.all, "list"] as const,            // 所有列表
  list: (filters) => [...memoKeys.lists(), filters] as const, // 带过滤条件的列表
  details: () => [...memoKeys.all, "detail"] as const,        // 所有详情
  detail: (name) => [...memoKeys.details(), name] as const,   // 单个详情
  // ...
};
```

**层级优势**: 可以精确控制失效范围
- `memoKeys.lists()` —— 失效所有列表查询
- `memoKeys.all` —— 失效所有 memo 相关查询

### 4.3 层 2: 缓存失效与同步（三种机制）

**机制 A: Mutation 主动失效**

在 `useCreateMemo`、`useUpdateMemo`、`useDeleteMemo` 的 `onSuccess` 中：

```typescript
// useCreateMemo (useMemoQueries.ts:185-192)
onSuccess: (newMemo) => {
  queryClient.invalidateQueries({ queryKey: memoKeys.lists() });  // 列表重取
  queryClient.setQueryData(memoKeys.detail(newMemo.name), newMemo);  // 预填充详情
  queryClient.invalidateQueries({ queryKey: userKeys.stats() });
}

// useDeleteMemo (useMemoQueries.ts:260-267)
onSuccess: (name) => {
  queryClient.removeQueries({ queryKey: memoKeys.detail(name) });  // 移除详情
  queryClient.invalidateQueries({ queryKey: memoKeys.lists() });  // 列表重取
  queryClient.invalidateQueries({ queryKey: userKeys.stats() });
}
```

**机制 B: SSE 实时同步（跨标签页/设备）**

**文件**: `web/src/hooks/useLiveMemoRefresh.ts:219-254`

```typescript
function handleSSEEvent(event: SSEChangeEvent, queryClient: ...) {
  switch (event.type) {
    case "memo.created":
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });
      break;

    case "memo.updated":
      queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      if (event.parent) {
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
      }
      break;

    case "memo.deleted":
      queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });
      break;

    // ... reaction 事件
  }
}
```

**SSE 挂载位置**: `web/src/main.tsx:51` —— 在 `AppInitializer` 中全局调用

```typescript
function AppInitializer({ children }) {
  // ...
  useLiveMemoRefresh();  // 全局 SSE 连接，应用全程活跃
  // ...
}
```

**机制 C: 断线重连补偿**

**文件**: `web/src/hooks/useLiveMemoRefresh.ts:122-128`

```typescript
if (hasConnectedOnceRef.current) {
  // 断线期间可能丢失事件，重连后刷新活动视图
  queryClient.invalidateQueries({ 
    queryKey: memoKeys.all, 
    refetchType: "active"  // 只重取当前活跃的查询
  });
  queryClient.invalidateQueries({ 
    queryKey: userKeys.stats(), 
    refetchType: "active" 
  });
}
```

### 4.4 层 3: UI 消费层

**文件**: `web/src/components/PagedMemoList/PagedMemoList.tsx:84-149`

```typescript
const PagedMemoList = (props: Props) => {
  const { filters } = useMemoFilterContext();

  // 1. 获取无限滚动数据
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isLoading } = 
    useInfiniteMemos({
      state: props.state || State.NORMAL,
      orderBy: props.orderBy || "create_time desc",
      filter: props.filter,
      pageSize: props.pageSize || DEFAULT_LIST_MEMOS_PAGE_SIZE,
    });

  // 2. 扁平化所有 pages
  const memos = useMemo(() => 
    data?.pages.flatMap((page) => page.memos) || [], 
    [data]
  );

  // 3. 自定义排序（如置顶优先）
  const sortedMemoList = useMemo(
    () => (props.listSort ? props.listSort(memos) : memos), 
    [memos, props.listSort]
  );

  // 4. 预取创建者信息（性能优化）
  useEffect(() => {
    if (!data?.pages || !props.showCreator) return;
    const lastPage = data.pages[data.pages.length - 1];
    const uniqueCreators = Array.from(
      new Set(lastPage.memos.map((memo) => memo.creator))
    );
    for (const creator of uniqueCreators) {
      queryClient.prefetchQuery({
        queryKey: userKeys.detail(creator),
        queryFn: async () => userServiceClient.getUser({ name: creator }),
        staleTime: 1000 * 60 * 5,
      });
    }
  }, [data?.pages, props.showCreator, queryClient]);

  // 5. 自动获取更多（页面不满一屏时）
  useAutoFetchWhenNotScrollable({
    hasNextPage, isFetchingNextPage, memoCount: sortedMemoList.length,
    onFetchNext: fetchNextPage,
  });

  // 6. 无限滚动监听
  useEffect(() => {
    if (!hasNextPage) return;
    const handleScroll = () => {
      const nearBottom = window.innerHeight + window.scrollY >= 
        document.body.offsetHeight - 300;
      if (nearBottom && !isFetchingNextPage) {
        fetchNextPage();
      }
    };
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (/* 渲染列表 */);
};
```

### 4.5 列表缓存策略对比

| 数据类型 | staleTime | gcTime | 失效触发方式 |
|---------|-----------|--------|-------------|
| **普通列表** (`useMemos`) | 30s (全局默认) | 5min | mutation + SSE |
| **无限列表** (`useInfiniteMemos`) | 60s (覆盖) | 5min | mutation + SSE |
| **Memo 详情** (`useMemo`) | 10s | 5min | 更新后 setQueryData + SSE |
| **用户详情** (`useUser`) | 5min | 5min | 预取优化，变化少 |
| **通知** (`useNotifications`) | 30s | 5min | 频繁更新 |
| **链接元数据** (`useLinkMetadata`) | 24h | 24h | 基本不变 |

---

## 五、状态分层设计决策总结

### 5.1 分层原则

```
┌─────────────────────────────────────────────────────────┐
│                    状态类型 → 存储位置                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  服务端数据  ──────►  React Query                       │
│  (Memos, Users, Stats)     (缓存、同步、重试、失效)       │
│                                                         │
│  全局 UI 状态  ─────►  React Context                    │
│  (Auth, Instance, View)     (跨组件共享，避免 props 钻取) │
│                                                         │
│  URL 驱动状态  ─────►  React Context + URL              │
│  (MemoFilter)                 (与搜索参数双向同步)        │
│                                                         │
│  组件复杂状态  ─────►  useReducer + Context             │
│  (MemoEditor)               (Action 模式，可预测状态转换)  │
│                                                         │
│  跨标签同步    ─────►  独立模块                         │
│  (Token)                    (BroadcastChannel)          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 关键证据索引

| 结论 | 证据文件 | 关键行 |
|-----|---------|-------|
| MemoFilter 在路由层挂载 | `web/src/App.tsx` | 59-63 |
| 乐观更新在 useUpdateMemo 实现 | `web/src/hooks/useMemoQueries.ts` | 196-250 |
| 乐观更新调用方: TaskListItem | `web/src/components/MemoContent/TaskListItem.tsx` | 17, 59-65 |
| 列表缓存键工厂 | `web/src/hooks/useMemoQueries.ts` | 11-19 |
| SSE 实时失效 | `web/src/hooks/useLiveMemoRefresh.ts` | 219-254 |
| SSE 全局挂载 | `web/src/main.tsx` | 51 |
| PagedMemoList 消费列表数据 | `web/src/components/PagedMemoList/PagedMemoList.tsx` | 92-149 |

---

## 六、文件索引

| 功能模块 | 文件路径 | 关键函数/组件 |
|---------|---------|--------------|
| **Query Client 全局配置** | `web/src/lib/query-client.ts` | `queryClient` (14-29) |
| **Memo 查询 + 乐观更新** | `web/src/hooks/useMemoQueries.ts` | `useUpdateMemo` (196), `patchMemoInCollectionQueries` (107), `useInfiniteMemos` (121) |
| **SSE 实时同步** | `web/src/hooks/useLiveMemoRefresh.ts` | `useLiveMemoRefresh` (70), `handleSSEEvent` (219) |
| **列表消费组件** | `web/src/components/PagedMemoList/PagedMemoList.tsx` | `PagedMemoList` (84) |
| **乐观更新调用: 任务勾选** | `web/src/components/MemoContent/TaskListItem.tsx` | `TaskListItem` (13) |
| **乐观更新调用: 操作菜单** | `web/src/components/MemoActionMenu/hooks.ts` | `useMemoActionHandlers` (22) |
| **乐观更新调用: MemoView** | `web/src/components/MemoView/hooks/useMemoActions.ts` | `useMemoActions` (4) |
| **顶层 Provider** | `web/src/main.tsx` | `Main` (60) |
| **路由层 Provider** | `web/src/App.tsx` | `App` (10) |
| **实例配置 Context** | `web/src/contexts/InstanceContext.tsx` | `InstanceProvider` (56) |
| **认证状态 Context** | `web/src/contexts/AuthContext.tsx` | `AuthProvider` (27) |
| **视图设置 Context** | `web/src/contexts/ViewContext.tsx` | `ViewProvider` (22) |
| **过滤器状态 Context** | `web/src/contexts/MemoFilterContext.tsx` | `MemoFilterProvider` (57) |
| **Token 三层缓存** | `web/src/auth-state.ts` | `getAccessToken` (52), `setAccessToken` (78) |
| **编辑器状态** | `web/src/components/MemoEditor/state/` | `editorReducer`, `EditorProvider` |
