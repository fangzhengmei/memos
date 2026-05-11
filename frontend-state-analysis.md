# Memos 前端状态管理分析报告

## 一、整体架构概览

Memos 前端采用**分层混合状态管理架构**，结合了多种状态管理模式来处理不同类型的数据：

```
┌─────────────────────────────────────────────────────────────┐
│                     应用层状态 (App State)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  React Context (全局UI状态)                           │   │
│  │  • AuthContext        • InstanceContext              │   │
│  │  • ViewContext        • MemoFilterContext            │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                     服务端状态 (Server State)                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  React Query v5 (服务端数据缓存 + 同步)               │   │
│  │  • 数据获取与缓存     • 乐观更新                       │   │
│  │  • 自动重取           • SSE 实时同步                  │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                     组件层状态 (Component State)             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  组件本地状态                                         │   │
│  │  • useState / useReducer                            │   │
│  │  • 复杂组件: useReducer + Context (如 MemoEditor)    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、React Query 服务端状态管理

### 2.1 全局 Query Client 配置

**文件**: `web/src/lib/query-client.ts:14-29`

```typescript
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 30,           // 30秒新鲜时间
      gcTime: 1000 * 60 * 5,          // 5分钟垃圾回收时间
      retry: shouldRetry,              // 自定义重试逻辑
      refetchOnWindowFocus: true,      // 窗口聚焦时重取
      refetchOnReconnect: true,        // 网络重连时重取
    },
    mutations: {
      retry: shouldRetry,
    },
  },
});
```

**设计要点**:
- `staleTime: 30s`: 平衡协作性与性能，30秒内数据视为新鲜
- `gcTime: 5min`: 缓存数据保留5分钟
- `refetchOnWindowFocus`: 确保用户返回时看到最新数据
- 自定义 `shouldRetry`: 认证错误不重试（由拦截器处理）

### 2.2 Query Keys 工厂模式

**文件**: `web/src/hooks/useMemoQueries.ts:10-19`

采用 Query Keys 工厂模式，确保缓存键的一致性：

```typescript
export const memoKeys = {
  all: ["memos"] as const,
  lists: () => [...memoKeys.all, "list"] as const,
  list: (filters: Partial<ListMemosRequest>) => [...memoKeys.lists(), filters] as const,
  details: () => [...memoKeys.all, "detail"] as const,
  detail: (name: string) => [...memoKeys.details(), name] as const,
  comments: (name: string) => [...memoKeys.all, "comments", name] as const,
  linkMetadata: (url: string) => [...memoKeys.all, "linkMetadata", url] as const,
};
```

**优势**:
- 类型安全
- 易于进行部分缓存失效
- 统一的缓存层级结构

---

## 三、乐观更新实现

### 3.1 核心实现位置

**文件**: `web/src/hooks/useMemoQueries.ts:196-250`

乐观更新在 `useUpdateMemo` mutation 中实现，分为三个阶段：

```typescript
export function useUpdateMemo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ update, updateMask }) => { ... },
    
    // 阶段1: 乐观更新 (onMutate)
    onMutate: async ({ update }) => {
      // 1. 取消正在进行的请求，防止竞态条件
      await queryClient.cancelQueries({ queryKey: memoKeys.all });
      
      // 2. 保存当前状态快照（用于回滚）
      const previousMemo = 
        queryClient.getQueryData<Memo>(memoKeys.detail(update.name)) || 
        findMemoInCollectionQueries(queryClient, update.name);
      
      // 3. 乐观更新详情缓存
      if (previousMemo) {
        queryClient.setQueryData(
          memoKeys.detail(update.name), 
          { ...previousMemo, ...memoPatch }
        );
      }
      
      // 4. 乐观更新所有列表缓存
      patchMemoInCollectionQueries(queryClient, memoPatch);
      
      return { previousMemo };
    },
    
    // 阶段2: 错误回滚 (onError)
    onError: (_err, { update }, context) => {
      if (context?.previousMemo && update.name) {
        queryClient.setQueryData(memoKeys.detail(update.name), context.previousMemo);
        patchMemoInCollectionQueries(queryClient, context.previousMemo);
      } else {
        queryClient.invalidateQueries({ queryKey: memoKeys.all });
      }
    },
    
    // 阶段3: 服务端确认 (onSuccess)
    onSuccess: (updatedMemo) => {
      // 用服务器返回的数据同步缓存
      queryClient.setQueryData(memoKeys.detail(updatedMemo.name), updatedMemo);
      patchMemoInCollectionQueries(queryClient, updatedMemo);
      // 刷新列表确保一致性
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
    },
  });
}
```

### 3.2 列表缓存修补机制

**文件**: `web/src/hooks/useMemoQueries.ts:32-109`

核心函数 `patchMemoInCollectionQueries` 实现了对所有 memo 相关查询的批量更新：

```typescript
function patchMemoInCollectionQueries(
  queryClient: ReturnType<typeof useQueryClient>, 
  update: MemoPatch
) {
  queryClient.setQueriesData<MemoCollectionQueryData>(
    { queryKey: memoKeys.all }, 
    (data) => patchMemoListQueryData(data, update)
  );
}
```

这个函数会：
1. 遍历所有以 `memoKeys.all` 为前缀的查询
2. 支持普通查询 (`ListMemosResponse`) 和无限滚动查询 (`InfiniteData<ListMemosResponse>`)
3. 深度遍历所有 pages 找到并更新匹配的 memo

### 3.3 乐观更新流程图

```
用户点击更新
    │
    ▼
┌─────────────────────┐
│   onMutate 触发      │
└──────────┬──────────┘
           │
    ┌──────▼──────┐
    │ 取消进行中请求 │  ← 防止竞态条件
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ 保存状态快照  │  ← 用于回滚
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ 乐观更新缓存  │  ← UI 立即响应
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │  发送API请求 │
    └──────┬──────┘
           │
      ┌────┴────┐
      │         │
   成功 │         │ 失败
      ▼         ▼
┌─────────┐  ┌──────────┐
│ onSuccess│  │  onError  │
│ 同步缓存  │  │  回滚缓存  │
│ 刷新列表  │  │  或失效    │
└─────────┘  └──────────┘
```

---

## 四、列表缓存策略

### 4.1 无限滚动列表查询

**文件**: `web/src/hooks/useMemoQueries.ts:121-139`

```typescript
export function useInfiniteMemos(
  request: Partial<ListMemosRequest> = {}, 
  options?: { enabled?: boolean }
) {
  return useInfiniteQuery({
    queryKey: memoKeys.list(request),
    queryFn: async ({ pageParam }) => {
      return memoServiceClient.listMemos(create(ListMemosRequestSchema, {
        ...request,
        pageToken: pageParam || "",
      }));
    },
    initialPageParam: "",
    getNextPageParam: (lastPage) => lastPage.nextPageToken || undefined,
    staleTime: 1000 * 60,           // 1分钟
    gcTime: 1000 * 60 * 5,          // 5分钟
    enabled: options?.enabled ?? true,
  });
}
```

### 4.2 分页列表组件

**文件**: `web/src/components/PagedMemoList/PagedMemoList.tsx:92-149`

```typescript
const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isLoading } = 
  useInfiniteMemos({...});

// 扁平化所有页面数据
const memos = useMemo(() => 
  data?.pages.flatMap((page) => page.memos) || [], 
  [data]
);

// 自动获取更多内容（当页面不可滚动时）
useAutoFetchWhenNotScrollable({...});

// 无限滚动监听
useEffect(() => {
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
```

### 4.3 缓存失效与同步策略

**策略1: Mutation 后主动失效**

在 `useCreateMemo`、`useUpdateMemo`、`useDeleteMemo` 中：

```typescript
// useCreateMemo: 失效列表，添加新 memo 到详情缓存
onSuccess: (newMemo) => {
  queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
  queryClient.setQueryData(memoKeys.detail(newMemo.name), newMemo);
}

// useDeleteMemo: 移除详情缓存，失效列表
onSuccess: (name) => {
  queryClient.removeQueries({ queryKey: memoKeys.detail(name) });
  queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
}
```

**策略2: SSE 实时同步**

**文件**: `web/src/hooks/useLiveMemoRefresh.ts:219-254`

通过 Server-Sent Events 监听服务端变更，实时失效相关缓存：

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
      break;
      
    case "memo.deleted":
      queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      break;
      
    // ... 其他事件类型
  }
}
```

**策略3: 重连后主动同步**

**文件**: `web/src/hooks/useLiveMemoRefresh.ts:122-128`

```typescript
if (hasConnectedOnceRef.current) {
  // 重新连接后刷新活动视图，补偿期间丢失的事件
  queryClient.invalidateQueries({ 
    queryKey: memoKeys.all, 
    refetchType: "active" 
  });
  queryClient.invalidateQueries({ 
    queryKey: userKeys.stats(), 
    refetchType: "active" 
  });
}
```

### 4.4 预取优化

**文件**: `web/src/components/PagedMemoList/PagedMemoList.tsx:108-126`

```typescript
// 新数据到达时预取创建者信息
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
```

---

## 五、React Context 全局状态

### 5.1 Context 层级结构

**文件**: `web/src/main.tsx:60-78`

```tsx
function Main() {
  return (
    <QueryClientProvider client={queryClient}>
      <InstanceProvider>        {/* 实例配置 */}
        <AuthProvider>          {/* 认证状态 */}
          <ViewProvider>        {/* 视图设置 */}
            <AppInitializer>
              <MemoFilterProvider>  {/* 过滤器状态 (在 App.tsx 中) */}
                <RouterProvider router={router} />
              </MemoFilterProvider>
            </AppInitializer>
          </ViewProvider>
        </AuthProvider>
      </InstanceProvider>
    </QueryClientProvider>
  );
}
```

### 5.2 各 Context 职责

| Context | 文件 | 职责 | 数据来源 |
|---------|------|------|----------|
| **InstanceContext** | `contexts/InstanceContext.tsx` | 实例级别配置（profile、设置） | API 拉取 |
| **AuthContext** | `contexts/AuthContext.tsx` | 用户认证状态、设置、快捷方式 | localStorage + API |
| **ViewContext** | `contexts/ViewContext.tsx` | 列表排序设置（升序/降序、时间字段） | localStorage |
| **MemoFilterContext** | `contexts/MemoFilterContext.tsx` | Memo 筛选条件（标签、可见性等） | URL 搜索参数 |

### 5.3 AuthContext 实现细节

**文件**: `contexts/AuthContext.tsx:1-204`

关键特性：
1. **双源同步**: `useState` 管理 UI 状态，同时同步到 React Query 缓存
2. **初始化逻辑**: 先尝试刷新 token，再获取用户信息
3. **登出处理**: 同时清理 Context 和 Query Client 缓存

```typescript
// 同步更新的用户到 Context 和 React Query 缓存
const setCurrentUser = useCallback((user: User | undefined) => {
  setState((prev) => ({ ...prev, currentUser: user }));
  if (user) {
    queryClient.setQueryData(userKeys.currentUser(), user);
    queryClient.setQueryData(userKeys.detail(user.name), user);
  }
}, [queryClient]);
```

### 5.4 MemoFilterContext URL 同步

**文件**: `contexts/MemoFilterContext.tsx:62-93`

实现了过滤器状态与 URL 查询参数的双向同步：

```typescript
// 从 URL 初始化
const [filters, setFiltersState] = useState<MemoFilter[]>(() => {
  return parseFilterQuery(searchParams.get("filter"));
});

// URL 变化 -> 状态同步
useEffect(() => {
  const filterParam = searchParams.get("filter") || "";
  if (filterParam !== lastSyncedUrlRef.current) {
    const newFilters = parseFilterQuery(filterParam);
    setFiltersState(newFilters);
  }
}, [searchParams]);

// 状态变化 -> URL 同步
useEffect(() => {
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

---

## 六、组件级状态管理

### 6.1 MemoEditor 状态管理

**文件**: `web/src/components/MemoEditor/state/`

MemoEditor 采用 `useReducer + Context` 模式管理复杂的编辑器状态：

```typescript
// 状态结构 (types.ts)
interface EditorState {
  content: string;
  metadata: {
    visibility: Visibility;
    displayTime: string;
    attachments: Attachment[];
    relations: MemoRelation[];
    // ...
  };
  localFiles: LocalFile[];
  ui: {
    isLoading: Record<string, boolean>;
    isFocusMode: boolean;
    isComposing: boolean;
  };
  audioRecorder: { ... };
  timestamps: { ... };
}

// Reducer (reducer.ts)
export function editorReducer(state: EditorState, action: EditorAction): EditorState {
  switch (action.type) {
    case "UPDATE_CONTENT":
      return { ...state, content: action.payload };
    case "ADD_ATTACHMENT":
      return {
        ...state,
        metadata: {
          ...state.metadata,
          attachments: [...state.metadata.attachments, action.payload],
        },
      };
    // ... 16+ 种 action 类型
  }
}

// Context Provider (context.tsx)
export const EditorProvider: FC<EditorProviderProps> = ({ children, initialEditorState }) => {
  const [state, dispatch] = useReducer(editorReducer, initialEditorState || initialState);
  
  const value = useMemo<EditorContextValue>(
    () => ({ state, dispatch, actions: editorActions }),
    [state],
  );
  
  return <EditorContext.Provider value={value}>{children}</EditorContext.Provider>;
};
```

**设计优点**:
- 复杂状态集中管理，便于调试
- Action 类型化，减少错误
- Context + useMemo 优化渲染性能

---

## 七、Token 状态管理 (auth-state.ts)

**文件**: `web/src/auth-state.ts`

独立的 token 管理模块，采用 **内存缓存 + localStorage 持久化 + BroadcastChannel 跨标签页同步** 三层策略：

```
┌───────────────────────────────────────────────────────────┐
│                  内存变量 (accessToken)                    │
│              ┌─────────────────────────┐                  │
│              │  快速读取，无序列化开销    │                  │
│              │  模块级单例              │                  │
│              └────────────┬────────────┘                  │
│                           │                               │
│          ┌────────────────┴────────────────┐              │
│          │                                 │              │
│    ┌─────▼──────┐                  ┌───────▼──────┐       │
│    │ localStorage│                  │BroadcastChannel│      │
│    │ 持久化存储   │                  │ 跨标签页同步    │       │
│    │ 会话间共享   │                  │ 新token广播    │       │
│    └────────────┘                  └──────────────┘       │
└───────────────────────────────────────────────────────────┘
```

关键实现：

```typescript
// 模块级单例
let accessToken: string | null = null;
let tokenExpiresAt: Date | null = null;

// 跨标签页同步
const tokenChannel = new BroadcastChannel("memos_token_sync");
tokenChannel.onmessage = (event: MessageEvent<TokenBroadcastMessage>) => {
  const { token, expiresAt } = event.data ?? {};
  if (token && expiresAt) {
    accessToken = token;
    tokenExpiresAt = new Date(expiresAt);
  }
};

// 获取时的缓存策略
export const getAccessToken = (): string | null => {
  if (!accessToken) {
    // 内存无值，尝试从 localStorage 恢复
    const storedToken = localStorage.getItem(TOKEN_KEY);
    // ... 检查过期时间
  }
  return accessToken;
};

// 设置时的多源同步
export const setAccessToken = (token: string | null, expiresAt?: Date): void => {
  accessToken = token;           // 1. 更新内存
  if (token && expiresAt) {
    localStorage.setItem(TOKEN_KEY, token);  // 2. 持久化
    getTokenChannel()?.postMessage({ token, expiresAt: ... });  // 3. 广播
  }
};
```

---

## 八、数据同步完整链路

### 8.1 创建 Memo 数据流

```
用户输入
   │
   ▼
┌─────────────┐
│ MemoEditor  │  ← useReducer 管理本地状态
│ (Context)   │
└──────┬──────┘
       │ 提交
       ▼
┌──────────────────┐
│ useCreateMemo()  │  ← React Query Mutation
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  onSuccess:                         │
│  1. invalidateQueries(lists)        │  ← 触发列表重取
│  2. setQueryData(detail, newMemo)   │  ← 预填充详情缓存
│  3. invalidateQueries(userStats)    │  ← 用户统计失效
└──────────────────┬───────────────────┘
                   │
    ┌──────────────┴──────────────┐
    │                             │
    ▼                             ▼
┌──────────────┐           ┌─────────────────┐
│ 本地UI更新    │           │ SSE 广播到其他   │
│ (列表重取)    │           │ 浏览器标签页     │
└──────────────┘           └────────┬────────┘
                                    │
                                    ▼
                              ┌──────────────┐
                              │ handleSSEEvent│
                              │ invalidate   │
                              └──────────────┘
```

### 8.2 更新 Memo 数据流（含乐观更新）

```
用户编辑
   │
   ▼
┌───────────────────┐
│ useUpdateMemo()   │
└─────────┬─────────┘
          │
    ┌─────▼─────┐
    │ onMutate  │  ← 乐观更新开始
    └─────┬─────┘
          │
    ┌─────▼────────────────────┐
    │ 1. 取消进行中的请求        │
    │ 2. 保存 previousMemo 快照  │
    │ 3. setQueryData(detail)   │  ← UI 立即更新
    │ 4. patchMemoInCollection  │  ← 所有列表同步
    └─────┬────────────────────┘
          │
    ┌─────▼─────┐
    │  API 请求  │
    └─────┬─────┘
          │
    ┌─────┴─────┐
    │           │
  成功 │         │ 失败
    ▼           ▼
┌────────┐  ┌─────────┐
│onSuccess│  │ onError │
│同步缓存  │  │ 回滚缓存 │
│刷新列表  │  │         │
└────────┘  └─────────┘
```

---

## 九、关键设计决策总结

### 9.1 状态分层原则

| 状态类型 | 存储位置 | 理由 |
|---------|---------|------|
| 服务端数据 | React Query | 缓存、同步、重试、失效等由库管理 |
| 全局 UI 状态 | React Context | 跨组件共享，避免 props drilling |
| 组件本地状态 | useState/useReducer | 不需要跨组件共享，保持局部性 |
| Token 状态 | 自定义模块 (auth-state.ts) | 需要跨标签同步、独立于 React 生命周期 |
| 编辑器复杂状态 | useReducer + Context | Action 模式适合复杂状态转换 |

### 9.2 缓存策略对比

| 数据类型 | staleTime | gcTime | 失效策略 |
|---------|-----------|--------|---------|
| Memo 详情 | 10s | 5min | 更新后 setQueryData + SSE |
| Memo 列表 | 60s (无限) / 30s (默认) | 5min | 创建/更新/删除后 invalidate |
| 用户详情 | 5min | 5min | 预取优化，变化少 |
| 通知 | 30s | 5min | 频繁更新，较短 staleTime |
| 链接元数据 | 24h | 24h | 基本不变，长缓存 |

### 9.3 乐观更新的边界条件

乐观更新适用于：
- **更新操作** (`useUpdateMemo`) - 数据变化可预测
- **高频率操作** - 用户期望即时反馈

不适用于：
- **创建操作** (`useCreateMemo`) - 缺少服务端生成的 ID/name
- **删除操作** (`useDeleteMemo`) - 直接失效更简单
- **破坏性操作** - 回滚成本高的操作

---

## 十、文件索引

| 功能模块 | 主要文件 | 关键函数/组件 |
|---------|---------|--------------|
| **Query Client 配置** | `web/src/lib/query-client.ts` | `queryClient` |
| **Memo 查询与乐观更新** | `web/src/hooks/useMemoQueries.ts` | `useUpdateMemo`, `patchMemoInCollectionQueries` |
| **用户查询** | `web/src/hooks/useUserQueries.ts` | `userKeys` factory |
| **实时同步 (SSE)** | `web/src/hooks/useLiveMemoRefresh.ts` | `useLiveMemoRefresh`, `handleSSEEvent` |
| **认证状态** | `web/src/contexts/AuthContext.tsx` | `AuthProvider`, `useAuth` |
| **实例配置** | `web/src/contexts/InstanceContext.tsx` | `InstanceProvider` |
| **视图设置** | `web/src/contexts/ViewContext.tsx` | `ViewProvider` |
| **过滤器状态** | `web/src/contexts/MemoFilterContext.tsx` | `MemoFilterProvider` |
| **Token 管理** | `web/src/auth-state.ts` | `getAccessToken`, `setAccessToken` |
| **编辑器状态** | `web/src/components/MemoEditor/state/` | `editorReducer`, `EditorProvider` |
| **分页列表** | `web/src/components/PagedMemoList/PagedMemoList.tsx` | `PagedMemoList`, `useInfiniteMemos` |
| **应用入口** | `web/src/main.tsx` | `Main`, `AppInitializer` |
