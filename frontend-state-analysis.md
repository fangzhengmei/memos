# Memos 前端状态管理分析报告

## 一、架构总览

Memos 前端采用**分层混合状态管理架构**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    服务端状态层 (Server State)                   │
│  React Query v5 — 缓存、同步、重试、失效、乐观更新               │
├─────────────────────────────────────────────────────────────────┤
│                    全局 UI 状态层 (Global UI State)             │
│  React Context — 跨组件共享                                       │
├─────────────────────────────────────────────────────────────────┤
│                    本地状态层 (Local State)                      │
│  独立模块 + localStorage + 组件 useState/useReducer             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、React Context 挂载层级（已修正）

### 2.1 实际层级结构

**关键发现**: Context 分两层挂载。

**证据1: main.tsx 顶层 Provider** (`web/src/main.tsx:60-78`)

```tsx
<QueryClientProvider>
  <InstanceProvider>       {/* 1. 实例配置 */}
    <AuthProvider>         {/* 2. 认证状态 */}
      <ViewProvider>       {/* 3. 视图设置 */}
        <RouterProvider />
      </ViewProvider>
    </AuthProvider>
  </InstanceProvider>
</QueryClientProvider>
```

**证据2: router/index.tsx 路由配置** (`web/src/router/index.tsx:52-112`)

```tsx
export const routeConfig: RouteObject[] = [
  {
    path: "/",
    element: <App />,                    // 根路径渲染 App
    children: [
      { path: Routes.AUTH, ... },        // /auth/* 是 App 的直接子路由
      { path: Routes.ENTRY, ... },       // 主应用路由
    ],
  },
];
```

**证据3: App.tsx 路由层 Provider** (`web/src/App.tsx:59-63`)

```tsx
const App = () => {
  return (
    <MemoFilterProvider>    {/* 4. 过滤器状态 —— 在路由层挂载 */}
      <Outlet />            {/* 所有子路由（包括 /auth/*）都在范围内 */}
    </MemoFilterProvider>
  );
};
```

### 2.2 完整挂载顺序

```
QueryClientProvider
  └─► InstanceProvider
       └─► AuthProvider
            └─► ViewProvider
                 └─► RouterProvider
                      │
                      ▼
                 App 组件
                      │
                      ▼
                 MemoFilterProvider
                      │
                      ▼
                    Outlet
              ┌───────┴───────┐
              │               │
              ▼               ▼
         /auth/*          /home, /explore 等
         (认证路由)        (主应用路由)
```

### 2.3 各 Context 职责与挂载位置

| Context | 挂载位置 | 数据来源 | 覆盖范围 |
|---------|---------|---------|---------|
| **QueryClientProvider** | main.tsx 最外层 | React Query 内部 | 应用全程 |
| **InstanceProvider** | main.tsx | API | 应用全程 |
| **AuthProvider** | main.tsx | localStorage + API | 应用全程 |
| **ViewProvider** | main.tsx | localStorage | 应用全程 |
| **MemoFilterProvider** | App.tsx 路由层 | URL search params | **所有子路由（包括 /auth/*）** |

**修正说明**:
- 之前错误地认为认证路由不在 MemoFilterProvider 范围内
- 实际路由结构显示 `/auth/*` 是 App 组件的直接子路由，通过 `<Outlet />` 渲染
- 因此 MemoFilterProvider 覆盖所有路由，包括认证页面

---

## 三、乐观更新层

### 3.1 最终结论要点

| 维度 | 结论 |
|-----|------|
| **实现位置** | React Query hooks 层 — `web/src/hooks/useMemoQueries.ts` |
| **具体函数** | `useUpdateMemo()` mutation 的 `onMutate` 生命周期 |
| **覆盖范围** | 详情缓存 + 所有列表缓存（普通 + 无限滚动） |
| **触发方式** | 调用 `useUpdateMemo()` 的组件自动获得乐观更新 |
| **适用边界** | 仅 `useUpdateMemo` 有乐观更新；`useCreateMemo` / `useDeleteMemo` 无 |

### 3.2 核心实现（三阶段）

**文件**: `web/src/hooks/useMemoQueries.ts:196-250`

以下为**真实实现摘录**（非伪代码）：

```typescript
export function useUpdateMemo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ update, updateMask }: { update: Partial<Memo>; updateMask: string[] }) => {
      const memo = await memoServiceClient.updateMemo({
        memo: create(MemoSchema, update as Record<string, unknown>),
        updateMask: create(FieldMaskSchema, { paths: updateMask }),
      });
      return memo;
    },

    // 阶段 1: onMutate — API 请求前执行
    onMutate: async ({ update }) => {
      if (!update.name) {
        return { previousMemo: undefined };
      }

      await queryClient.cancelQueries({ queryKey: memoKeys.all });  // 取消竞态请求

      const previousMemo =
        queryClient.getQueryData<Memo>(memoKeys.detail(update.name)) ||
        findMemoInCollectionQueries(queryClient, update.name);  // 双源查找快照
      const memoPatch: MemoPatch = { ...update, name: update.name };

      if (previousMemo) {
        queryClient.setQueryData(memoKeys.detail(update.name), { ...previousMemo, ...memoPatch });
      }
      patchMemoInCollectionQueries(queryClient, memoPatch);

      return { previousMemo };
    },

    // 阶段 2: onError — 失败回滚
    onError: (_err, { update }, context) => {
      if (context?.previousMemo && update.name) {
        queryClient.setQueryData(memoKeys.detail(update.name), context.previousMemo);
        patchMemoInCollectionQueries(queryClient, context.previousMemo);
      } else {
        queryClient.invalidateQueries({ queryKey: memoKeys.all });
      }
    },

    // 阶段 3: onSuccess — 服务端确认
    onSuccess: (updatedMemo) => {
      queryClient.setQueryData(memoKeys.detail(updatedMemo.name), updatedMemo);  // 1. 同步详情
      patchMemoInCollectionQueries(queryClient, updatedMemo);                    // 2. 同步列表
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });             // 3. 失效列表触发重取
      if (updatedMemo.parent) {                                                  // 4. 条件性失效评论
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(updatedMemo.parent) });
      }
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });             // 5. 失效用户统计
    },
  });
}
```

**`onSuccess` 完整 5 步关联失效**（真实实现）：
1. 用服务端返回数据同步详情缓存
2. 用服务端返回数据同步所有列表缓存
3. 触发列表查询失效（确保排序/筛选正确）
4. 若 memo 有 `parent`（评论场景），失效该父 memo 的评论列表
5. 失效用户统计缓存

### 3.3 列表缓存修补机制

**文件**: `web/src/hooks/useMemoQueries.ts:32-109`

```typescript
function patchMemoInCollectionQueries(queryClient, update) {
  queryClient.setQueriesData<MemoCollectionQueryData>(
    { queryKey: memoKeys.all },              // 匹配所有 memo 相关查询
    (data) => patchMemoListQueryData(data, update)
  );
}

function patchMemoListQueryData<T>(data, update): T | undefined {
  if (isMemoListResponse(data)) {           // 普通列表
    return patchMemoListResponse(data, update) as T;
  }
  if (isInfiniteMemoListData(data)) {       // 无限滚动列表
    return {
      ...data,
      pages: data.pages.map(page => patchMemoListResponse(page, update))
    } as T;
  }
  return data;
}
```

**关键点**: 使用 `setQueriesData` 批量更新所有 memo 相关查询，同时支持普通列表和无限滚动的 `InfiniteData` 结构。

### 3.4 调用链

| 调用方 | 文件 | 用途 |
|-------|------|------|
| **TaskListItem** | `components/MemoContent/TaskListItem.tsx:17` | 勾选任务 |
| **useMemoActionHandlers** | `components/MemoActionMenu/hooks.ts:28` | 切换置顶、归档/恢复 |
| **useMemoActions** | `components/MemoView/hooks/useMemoActions.ts:5` | 取消置顶 |

**调用方无需关心乐观更新逻辑**，这是 `useUpdateMemo` hook 内部封装的实现细节。

---

## 四、列表缓存层

### 4.1 最终结论要点

| 维度 | 结论 |
|-----|------|
| **核心层** | React Query 缓存 — `web/src/hooks/useMemoQueries.ts` |
| **查询函数** | `useMemos()` 普通列表 / `useInfiniteMemos()` 无限滚动 |
| **缓存键策略** | `memoKeys.list(filters)` — 过滤条件作为缓存键的一部分 |
| **缓存时效** | 普通列表 30s / 无限列表 60s（gcTime 均为 5min） |
| **失效机制** | 三层：mutation 主动失效 + SSE 实时同步 + 断线重连补偿 |
| **UI 消费层** | `PagedMemoList` 组件 — 扁平化 pages + 无限滚动监听 |

### 4.2 三层结构

```
┌──────────────────────────────────────────────────────────┐
│  层 1: 数据获取层 — useMemoQueries.ts                     │
│  • useMemos() / useInfiniteMemos()                        │
│  • memoKeys 缓存键工厂                                    │
│  • staleTime/gcTime 配置                                  │
└──────────────────────┬───────────────────────────────────┘
                       │ 读取/写入
                       ▼
┌──────────────────────────────────────────────────────────┐
│  层 2: 缓存失效层                                        │
│  • Mutation onSuccess — 主动 invalidate                  │
│  • SSE (useLiveMemoRefresh) — 跨标签页/设备实时失效      │
│  • 断线重连补偿 — 重连后刷新活动视图                      │
└──────────────────────┬───────────────────────────────────┘
                       │ 失效后触发重取
                       ▼
┌──────────────────────────────────────────────────────────┐
│  层 3: UI 消费层 — PagedMemoList.tsx                     │
│  • 调用 useInfiniteMemos                                 │
│  • data.pages.flatMap() 扁平化                           │
│  • 无限滚动监听 + 自动预取                                │
└──────────────────────────────────────────────────────────┘
```

### 4.3 层 1: 数据获取

**文件**: `web/src/hooks/useMemoQueries.ts:111-139`

```typescript
export const memoKeys = {
  all: ["memos"] as const,
  lists: () => [...memoKeys.all, "list"] as const,
  list: (filters) => [...memoKeys.lists(), filters] as const,  // 过滤条件作为缓存键
};

export function useInfiniteMemos(request, options) {
  return useInfiniteQuery({
    queryKey: memoKeys.list(request),
    queryFn: async ({ pageParam }) => memoServiceClient.listMemos({
      ...request,
      pageToken: pageParam || "",
    }),
    initialPageParam: "",
    getNextPageParam: (lastPage) => lastPage.nextPageToken || undefined,
    staleTime: 1000 * 60,      // 60s
    gcTime: 1000 * 60 * 5,     // 5min
  });
}
```

### 4.4 层 2: 缓存失效（三种机制）

**机制 A: Mutation 主动失效**

```typescript
// useCreateMemo onSuccess
queryClient.invalidateQueries({ queryKey: memoKeys.lists() });

// useDeleteMemo onSuccess
queryClient.removeQueries({ queryKey: memoKeys.detail(name) });
queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
```

**机制 B: SSE 实时同步**

**文件**: `web/src/hooks/useLiveMemoRefresh.ts:219-254`

```typescript
function handleSSEEvent(event, queryClient) {
  switch (event.type) {
    case "memo.created":
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      break;
    case "memo.updated":
      queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      break;
    case "memo.deleted":
      queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      break;
  }
}
```

**SSE 挂载位置**: `web/src/main.tsx:51` — 全局调用，应用全程活跃。

**机制 C: 断线重连补偿**

**文件**: `web/src/hooks/useLiveMemoRefresh.ts:122-128`

```typescript
if (hasConnectedOnceRef.current) {
  queryClient.invalidateQueries({ 
    queryKey: memoKeys.all, 
    refetchType: "active"  // 只重取当前活跃的查询
  });
}
```

### 4.5 层 3: UI 消费

**文件**: `web/src/components/PagedMemoList/PagedMemoList.tsx:33-149`

**两处 `hasNextPage` 门槛的真实实现：

```typescript
// ─────────────────────────────────────────────────────────────
// 门槛 1: 自动获取 Hook (行 33-82)
// 以下为提炼示例（真实实现 50 行，含定时器、useEffect 等细节已省略）
// ─────────────────────────────────────────────────────────────
function useAutoFetchWhenNotScrollable({ hasNextPage, isFetchingNextPage, memoCount, onFetchNext }) {
  // ... 真实代码含 autoFetchTimeoutRef、isPageScrollable、两个 useEffect ...

  const checkAndFetchIfNeeded = useCallback(async () => {
    // ... 真实代码有定时器逻辑 ...

    // 门槛条件（逐行摘录自真实代码行 58）：
    const shouldFetch = !isPageScrollable() && hasNextPage && !isFetchingNextPage && memoCount > 0;

    if (shouldFetch) {
      await onFetchNext();
    }
  }, [hasNextPage, isFetchingNextPage, memoCount, isPageScrollable, onFetchNext]);
}

// ─────────────────────────────────────────────────────────────
// 门槛 2: 手动滚动监听 (行 136-149)
// 以下为逐行摘录自真实代码（无省略）
// ─────────────────────────────────────────────────────────────
useEffect(() => {
  // 真实代码行 138: 没有下一页直接 return，不监听滚动
  if (!hasNextPage) return;

  const handleScroll = () => {
    // 真实代码行 141: 距离底部 300px
    const nearBottom = window.innerHeight + window.scrollY >= document.body.offsetHeight - 300;
    // 真实代码行 142: 且不在请求中
    if (nearBottom && !isFetchingNextPage) {
      fetchNextPage();
    }
  };

  window.addEventListener("scroll", handleScroll);
  return () => window.removeEventListener("scroll", handleScroll);
}, [hasNextPage, isFetchingNextPage, fetchNextPage]);
```

**真实 `hasNextPage` 门槛总结**（两处实现）：

| 触发方式 | 门槛条件 | 真实代码行 |
|---------|---------|-----------|
| **自动获取** | `!isPageScrollable() && hasNextPage && !isFetchingNextPage && memoCount > 0` | 58 |
| **滚动触发** | `if (!hasNextPage) return;` + `nearBottom && !isFetchingNextPage` | 138 + 142 |

### 4.6 缓存策略对比

| 数据类型 | staleTime | gcTime | 失效触发方式 |
|---------|-----------|--------|-------------|
| 普通列表 | 30s | 5min | mutation + SSE |
| 无限列表 | 60s | 5min | mutation + SSE |
| Memo 详情 | 10s | 5min | setQueryData + SSE |

---

## 五、关键证据索引

| 结论 | 证据文件 | 关键行 |
|-----|---------|-------|
| MemoFilter 在路由层挂载 | `web/src/App.tsx` | 59-63 |
| /auth 是 App 子路由 | `web/src/router/index.tsx` | 58-74 |
| 乐观更新在 useUpdateMemo 实现 | `web/src/hooks/useMemoQueries.ts` | 196-250 |
| 列表缓存修补机制 | `web/src/hooks/useMemoQueries.ts` | 32-109 |
| SSE 实时失效 | `web/src/hooks/useLiveMemoRefresh.ts` | 219-254 |
| SSE 全局挂载 | `web/src/main.tsx` | 51 |
| PagedMemoList 消费列表数据 | `web/src/components/PagedMemoList/PagedMemoList.tsx` | 84-149 |

---

## 六、文件索引

| 功能模块 | 文件路径 | 关键函数/组件 |
|---------|---------|--------------|
| **Query Client 配置** | `web/src/lib/query-client.ts` | `queryClient` |
| **Memo 查询 + 乐观更新** | `web/src/hooks/useMemoQueries.ts` | `useUpdateMemo`, `useInfiniteMemos` |
| **SSE 实时同步** | `web/src/hooks/useLiveMemoRefresh.ts` | `useLiveMemoRefresh`, `handleSSEEvent` |
| **列表消费组件** | `web/src/components/PagedMemoList/PagedMemoList.tsx` | `PagedMemoList` |
| **顶层 Provider** | `web/src/main.tsx` | `Main` |
| **路由层 Provider** | `web/src/App.tsx` | `App` |
| **过滤器状态** | `web/src/contexts/MemoFilterContext.tsx` | `MemoFilterProvider` |
