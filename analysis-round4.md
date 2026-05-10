# Memos 深度分析（Round 4）— 分享页评论静默失败分析

## 问题：ListMemoComments 失败后的前端处理

### 1.1 完整失败链路分析

```
后端 PROTECTED/PRIVATE 主 memo
              │
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        前端分享页面                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MemoDetail.tsx                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                                                               │  │
│  │  // 1. 主 memo 查询（成功，因为 useSharedMemo 走 GetMemoByShare） │  │
│  │  const { data: memoFromShare, error: shareError, isLoading }  │  │
│  │    = useSharedMemo(shareToken, { enabled: isShareMode });     │  │
│  │       │                                                       │  │
│  │       └─ 成功，memo 已获取                                     │  │
│  │                                                               │  │
│  │  // 2. 评论查询（失败，因为 ListMemoComments 不识别 share_token） │  │
│  │  const { data: commentsResponse }                             │  │
│  │    = useMemoComments(memoName, { enabled: !!memo });          │  │
│  │       │                                                       │  │
│  │       └─ ❌ 后端返回 Unauthenticated                          │  │
│  │          (checkMemoReadAccess 失败)                          │  │
│  │                                                               │  │
│  │  // 3. 错误被静默忽略！                                       │  │
│  │  const comments = commentsResponse?.memos || [];              │  │
│  │       │                                                       │  │
│  │       └─ data 为 undefined → 降级为 []                        │  │
│  │                                                               │  │
│  │  // 4. useMemoDetailError 只处理主 memo 的错误                │  │
│  │  useMemoDetailError({ error: shareError });                  │  │
│  │       │                                                       │  │
│  │       └─ 不消费 commentsResponse 的 error！                  │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  <MemoCommentSection memo={displayMemo} comments={comments} />      │
│                              │                                      │
│                              ▼                                      │
│  MemoCommentSection.tsx                                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  // comments = []                                             │  │
│  │  // 不区分：                                                   │  │
│  │  //   - 真的没有评论 (0 条)                                    │  │
│  │  //   - 查询失败 (API 返回 401)                                │  │
│  │                                                               │  │
│  │  {comments.length === 0 ? (                                   │  │
│  │    showCreateButton && (                                      │  │
│  │      // 显示"写评论"按钮（仅登录用户）                          │  │
│  │      // 但分享页面用户通常未登录 → 什么都不显示！              │  │
│  │    )                                                          │  │
│  │  ) : (                                                        │  │
│  │    // 显示评论数量和评论列表                                    │  │
│  │  )}                                                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  结果：                                                              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ✅ 主 memo 内容正常显示                                        │  │
│  │  ✅ 主 memo 附件正常显示（通过 withShareAttachmentLinks）       │  │
│  │  ❌ 评论区域完全空白（无提示）                                  │  │
│  │  ❌ 没有"加载失败"、"评论不可见"等任何错误提示                  │  │
│  │  ❌ 用户不知道是"没有评论"还是"无法加载评论"                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键代码分析

#### useMemoComments 的返回结构

```typescript
// useMemoQueries.ts:271-286

export function useMemoComments(name: string, options?: { enabled?: boolean; pageSize?: number }) {
  return useQuery({
    queryKey: [...memoKeys.comments(name), options?.pageSize ?? 0],
    queryFn: async () => {
      const response = await memoServiceClient.listMemoComments(
        create(ListMemoCommentsRequestSchema, {
          name,
          pageSize: options?.pageSize ?? 0,
        }),
      );
      return response;
    },
    enabled: options?.enabled ?? true,
    staleTime: 1000 * 60, // 1 minute
  });
}
```

**返回值**：
- `data`：成功时的响应数据
- `error`：失败时的错误对象
- `isLoading` / `isFetching`：加载状态
- `isError`：是否有错误

#### MemoDetail 只消费 data，忽略 error

```typescript
// MemoDetail.tsx:51-54

const { data: commentsResponse } = useMemoComments(memoName, {
  enabled: !!memo,
});
// ⚠️ 没有解构 error、isLoading、isError！
// 解构只取了 data

const comments = commentsResponse?.memos || [];
//         ^^^^^^^^^^^^^^^^^^^^^
//         data 为 undefined 时 → 降级为 []
```

**React Query 的默认行为**：
- `queryFn` 抛出异常 → `data = undefined`，`error = 异常对象`
- `error` 未被使用，`isError` 未被检查
- `data?.memos` 短路求值为 `undefined`
- `|| []` 静默降级为空数组

#### Query Client 的重试策略

```typescript
// query-client.ts:9-28

const shouldRetry = (failureCount: number, error: unknown): boolean => {
  if (error instanceof ConnectError && error.code === Code.Unauthenticated) return false;
  //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //     认证错误不重试，直接失败
  return failureCount < 1;
};

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: shouldRetry,
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
  },
});
```

**对评论查询的影响**：
- `ListMemoComments` 返回 `Unauthenticated`
- `shouldRetry` 检测到 `Code.Unauthenticated`，不重试
- 查询立即失败，`data = undefined`
- 当用户刷新页面或网络重连时，`refetchOnWindowFocus` / `refetchOnReconnect` 会再次尝试
- 但每次都失败，每次都静默降级为 `[]`

#### useMemoDetailError 不处理评论错误

```typescript
// MemoDetail.tsx:43-45

// 只处理主 memo 的错误
useMemoDetailError({
  error: error as Error | null,  // ← 来自 useMemo / useSharedMemo 的错误
});

// 不处理：
// - useMemoComments 的 error
// - useMemo(父 memo) 的 error
```

```typescript
// useMemoDetailError.ts:10-29

const useMemoDetailError = ({ error }: UseMemoDetailErrorOptions) => {
  useEffect(() => {
    if (!error) return;

    if (error instanceof ConnectError) {
      if (error.code === Code.Unauthenticated || 
          error.code === Code.PermissionDenied || 
          error.code === Code.NotFound) {
        navigateTo("/404", { replace: true });  // ← 跳转到 404
        return;
      }
      toast.error(error.message);  // ← 显示错误提示
      return;
    }
    toast.error(error.message);
  }, [error, navigateTo]);
};
```

**问题**：
- 主 memo 查询如果失败（如 share_token 过期），会跳转到 404
- 但评论查询失败，`useMemoDetailError` 完全不知道
- 没有任何错误边界捕获评论查询的错误

#### MemoCommentSection 对空列表的处理

```typescript
// MemoCommentSection.tsx:17-77

const MemoCommentSection = ({ memo, comments, parentPage }: Props) => {
  const currentUser = useCurrentUser();
  const showCreateButton = currentUser && !showEditor;
  //      ^^^^^^^^^^^^^^^
  //      分享页面用户通常未登录 → showCreateButton = false

  return (
    <div className="pt-8 pb-16 w-full">
      <div className="relative mx-auto grow w-full min-h-full flex flex-col justify-start items-start gap-y-1">
        
        {comments.length === 0 ? (
          // ⚠️ 分支 1：空列表（不区分"真的无评论" vs "查询失败"）
          showCreateButton && (
            // 登录用户：显示"写评论"按钮
            <div className="w-full flex flex-row justify-center items-center py-6">
              <Button variant="ghost" onClick={() => setShowEditor(true)}>
                <span className="text-muted-foreground">{t("memo.comment.write-a-comment")}</span>
                <MessageCircleIcon className="ml-2 w-5 h-auto text-muted-foreground" />
              </Button>
            </div>
          )
          // 未登录用户：上面的条件为 false → 什么都不渲染！
          // 结果：评论区域完全空白
          
        ) : (
          // ⚠️ 分支 2：有评论 → 显示评论数量和列表
          <div className="w-full flex flex-row justify-between items-center h-8 pl-3 mb-2">
            <div className="flex flex-row justify-start items-center">
              <MessageCircleIcon className="w-5 h-auto text-muted-foreground mr-1" />
              <span className="text-muted-foreground text-sm">{t("memo.comment.self")}</span>
              <span className="text-muted-foreground text-sm ml-1">({comments.length})</span>
            </div>
            {showCreateButton && (
              <Button variant="ghost">{t("memo.comment.write-a-comment")}</Button>
            )}
          </div>
        )}
        
        // 渲染评论列表（comments = [] 时不渲染任何东西）
        {comments.map((comment) => (
          <div className="w-full" key={`${comment.name}-${comment.updateTime}`}>
            <MemoView memo={comment} parentPage={parentPage} showCreator compact />
          </div>
        ))}
      </div>
    </div>
  );
};
```

**关键问题**：
1. `comments.length === 0` 时：
   - 登录用户：显示"写评论"按钮
   - 未登录用户（分享页面）：什么都不显示
2. 没有区分"真的无评论"和"查询失败"
3. 没有加载状态指示
4. 没有错误状态指示

---

## 问题 2："评论不可见但无提示"的触发条件和影响边界

### 2.1 触发条件矩阵

| 场景 | 主 memo visibility | 用户登录状态 | 评论 API 结果 | 前端显示 | 是否有提示 |
|------|-------------------|-------------|--------------|---------|-----------|
| **1** | PUBLIC | 未登录 | ✅ 返回 PUBLIC 评论 | 显示评论 | ✅ 正常 |
| **2** | PUBLIC | 已登录 | ✅ 返回 PUBLIC+PROTECTED+自己的 | 显示评论 | ✅ 正常 |
| **3** | PROTECTED | 已登录（任意） | ✅ 返回 PUBLIC+PROTECTED+自己的 | 显示评论 | ✅ 正常 |
| **4** | PROTECTED | 未登录（分享链接） | ❌ Unauthenticated | **评论区域空白** | ❌ **无提示** |
| **5** | PRIVATE | 已登录（创建者） | ✅ 返回所有评论 | 显示评论 | ✅ 正常 |
| **6** | PRIVATE | 已登录（非创建者） | ❌ PermissionDenied | **评论区域空白** | ❌ **无提示** |
| **7** | PRIVATE | 未登录（分享链接） | ❌ Unauthenticated | **评论区域空白** | ❌ **无提示** |
| **8** | 任何 | 任何 | 网络错误 | **评论区域空白** | ❌ **无提示** |

### 2.2 最典型触发场景：分享 PROTECTED/PRIVATE memo

```
用户操作流程:

1. 备忘录创建者（已登录）
   ┌─────────────────────────────────────┐
   │ memo: 季度销售报告                   │
   │ visibility: PROTECTED                │
   │                                      │
   │ 评论 1: Alice - "这个数据需要确认"    │
   │   ├─ visibility: PROTECTED           │
   │   └─ 附件: sales_chart.png (LOCAL)   │
   │                                      │
   │ 评论 2: Bob - "附上最新数据"          │
   │   ├─ visibility: PUBLIC              │
   │   └─ 附件: data.pdf (LOCAL)         │
   │                                      │
   │ 点击"分享" → 生成 share_token        │
   └──────────────────┬──────────────────┘
                      │
                      ▼
2. 分享链接（未登录用户访问）
   ┌─────────────────────────────────────────────────────────┐
   │ URL: https://memos.example/memos/shares/xyz123           │
   │                                                         │
   │ Step 1: GetMemoByShare(token=xyz123)                     │
   │         ✅ 成功，获取主 memo                              │
   │                                                         │
   │ Step 2: ListMemoComments(memo_uid)                       │
   │         ❌ checkMemoReadAccess: 主 memo=PROTECTED        │
   │            user=nil → Unauthenticated                   │
   │                                                         │
   │ Step 3: 前端                                            │
   │         ├─ data=undefined                                │
   │         ├─ error=ConnectError(Unauthenticated)          │
   │         └─ error 被忽略                                  │
   │                                                         │
   │ Step 4: comments = []                                   │
   │                                                         │
   │ Step 5: MemoCommentSection 渲染                          │
   │         ├─ comments.length === 0                         │
   │         ├─ currentUser = null (未登录)                   │
   │         └─ showCreateButton = false                      │
   │                                                         │
   │ 最终显示:                                               │
   │   ✅ 主 memo 内容                                       │
   │   ✅ 主 memo 附件（LOCAL 类型通过 share_token 可访问）   │
   │   ❌ 评论区域：完全空白，没有任何提示                    │
   │                                                         │
   │ 用户看到的是什么？                                       │
   │   - 备忘录内容正常                                       │
   │   - 好像"这篇备忘录没有评论"                             │
   │   - 但实际上有 2 条评论！                                │
   │   - 没有"评论不可见"、"需要登录查看"等提示               │
   │                                                         │
   └─────────────────────────────────────────────────────────┘
```

### 2.3 评论附件的双重静默失败

```
PUBLIC 评论的 LOCAL 附件（Bob 的评论）:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  实际上应该是什么？                                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 评论 2: Bob - "附上最新数据"                              │  │
│  │   附件: data.pdf (LOCAL)                                 │  │
│  │   visibility: PUBLIC                                     │  │
│  │                                                         │  │
│  │ 主 memo 是 PROTECTED：                                   │
│  │   - ListMemoComments API 被拒绝 (401)                   │  │
│  │   - 评论完全无法获取                                     │  │
│  │                                                         │
│  │ 即使 API 成功了，附件也有问题：                          │  │
│  │   - 评论附件不走 withShareAttachmentLinks               │  │
│  │   - 附件 URL: /file/attachments/comment-attach/data.pdf │  │
│  │   - 没有 ?share_token=...                               │  │
│  │                                                         │
│  │ checkAttachmentPermission:                               │
│  │   attachment.MemoID = 评论 memo 的 ID (comment_id)      │
│  │   memo.Visibility = PUBLIC                              │
│  │   → ✅ 可以访问！                                        │
│  │                                                         │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  实际发生的情况：                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ ListMemoComments: Unauthenticated                       │  │
│  │ comments = []                                            │  │
│  │                                                         │  │
│  │ 结果：                                                   │  │
│  │   ❌ 连评论文本都看不到                                  │  │
│  │   ❌ 更不用说附件了                                      │  │
│  │   ❌ 完全静默，没有任何错误提示                          │  │
│  │                                                         │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

PROTECTED 评论的 LOCAL 附件（Alice 的评论）:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  即使 API 成功获取到评论：                                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 评论 1: Alice - "这个数据需要确认"                        │  │
│  │   附件: sales_chart.png (LOCAL)                         │  │
│  │   visibility: PROTECTED                                 │  │
│  │                                                         │  │
│  │ 附件 URL: /file/attachments/comment-attach/chart.png    │
│  │         ← 没有 ?share_token=...                         │
│  │                                                         │
│  │ checkAttachmentPermission:                               │
│  │   attachment.MemoID = 评论 memo 的 ID (comment_id)      │
│  │   memo.Visibility = PROTECTED                           │
│  │   share_token? = "xyz123"                               │
│  │   MemoShare.MemoID = 主 memo ID ≠ comment_id            │
│  │                                                         │
│  │   → ❌ 401 Unauthorized                                 │
│  │                                                         │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  实际情况：连 API 都失败了                                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ comments = []                                           │  │
│  │ 什么都看不到                                             │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 影响边界分析

#### 受影响的场景

| 影响类型 | 说明 |
|---------|------|
| **用户困惑** | 以为备忘录真的没有评论，实际上有评论但不可见 |
| **信息不对称** | 分享者和被分享者看到的内容不一致 |
| **数据丢失感** | 分享链接"丢失"了评论，用户可能以为数据损坏 |
| **安全边界混淆** | 设计上是有意的权限隔离，但体验上像 bug |

#### 不受影响的场景

| 场景 | 状态 |
|------|------|
| 主 memo 内容 | ✅ 正常显示 |
| 主 memo 附件（LOCAL/DATABASE） | ✅ 通过 share_token 可访问 |
| 主 memo 附件（EXTERNAL/S3） | ✅ 保持原 externalLink（绕过 Memos 后端） |
| PUBLIC memo 的分享链接 | ✅ 公共评论可见（API 可访问） |
| 已登录用户访问分享链接 | ✅ 自己有权限的评论可见 |
| 普通详情页（非分享模式） | ✅ 正常权限控制，有错误提示 |

### 2.5 静默失败的三层叠加

```
静默失败的三层叠加:

Layer 1: 查询错误被忽略
┌─────────────────────────────────────────────────────────────┐
│ useMemoComments 返回 error，但 MemoDetail 不解构 error       │
│ comments = commentsResponse?.memos || []  → 静默降级为 []    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Layer 2: 错误未被上报
┌─────────────────────────────────────────────────────────────┐
│ useMemoDetailError 只处理主 memo 的 error                    │
│ 评论查询的 error 完全没有被捕获、没有 toast、没有跳转 404      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Layer 3: UI 不区分"无评论" vs "加载失败"
┌─────────────────────────────────────────────────────────────┐
│ MemoCommentSection 对 comments=[] 的处理：                    │
│   - 已登录：显示"写评论"按钮（暗示确实没有评论）               │
│   - 未登录：什么都不显示（评论区域完全空白）                   │
│   - 没有加载中状态指示                                        │
│   - 没有错误状态指示                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
              最终：用户完全不知道出了问题
```

---

## 关键代码索引

| 分析点 | 文件位置 | 行号范围 |
|-------|---------|---------|
| useMemoComments 实现 | `web/src/hooks/useMemoQueries.ts` | 271-286 |
| MemoDetail 评论查询调用 | `web/src/pages/MemoDetail.tsx` | 51-54 |
| useMemoDetailError 实现 | `web/src/hooks/useMemoDetailError.ts` | 10-29 |
| MemoCommentSection 空列表处理 | `web/src/components/MemoCommentSection.tsx` | 34-56 |
| Query Client 重试策略 | `web/src/lib/query-client.ts` | 9-28 |
| ListMemoComments 后端实现 | `server/router/api/v1/memo_service.go` | 759-910 |
| checkMemoReadAccess | `server/router/api/v1/memo_service.go` | 42-71 |
| withShareAttachmentLinks | `web/src/hooks/useMemoShareQueries.ts` | 95-100 |
