# Memos 标签完整链路分析与搜索过滤一致性核对

## 概述

本文档深入分析 memos 项目中标签从**内容解析写入** → **后端统计汇总** → **前端 tagCount 展示更新**的完整数据流转链路，并重点核对**标签变更时的联动刷新机制**以及**与搜索过滤联动时列表刷新的一致性**问题。

---

## 一、标签完整数据流转链路

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                     用户操作层                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  创建笔记 ──┐                                                                              │
│             │                                                                              │
│  编辑笔记 ──┼──► 内容包含 #tag 语法                                                        │
│             │                                                                              │
│  删除笔记 ──┘                                                                              │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                     前端处理层                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  MemoEditor 组件                                                                           │
│  ├── 调用 useCreateMemo / useUpdateMemo mutation                                          │
│  └── 发送 CreateMemoRequest / UpdateMemoRequest (包含 content 字段)                       │
│                                                                                             │
│  缓存层 (React Query)                                                                       │
│  ├── memoKeys.lists()                                                                      │
│  ├── memoKeys.list({ filter: ... })                                                        │
│  └── userKeys.stats()                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ (gRPC API)
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                     后端 API 层                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  CreateMemo / UpdateMemo / DeleteMemo                                                     │
│  ├── 接收请求参数                                                                          │
│  └── 调用 Store 层                                                                         │
│                                                                                             │
│  GetUserStats / ListAllUserStats (统计接口)                                                 │
│  └── 从 memo.Payload.Tags 统计标签计数                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                     后端存储层                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  Store: CreateMemo / UpdateMemo                                                           │
│  └── 调用 RebuildMemoPayload 重建 payload                                                  │
│                                                                                             │
│  memopayload.Runner                                                                         │
│  └── RebuildMemoPayload                                                                    │
│      └── MarkdownService.ExtractAll(content)                                               │
│          ├── 解析 #tag → Tags[]                                                            │
│          └── 解析属性 (hasLink, hasCode, hasTaskList...) → Property                       │
│                                                                                             │
│  数据库: memo 表                                                                           │
│  ├── content 字段 (原始 Markdown 文本)                                                     │
│  └── payload 字段 (JSON: { tags: ["work", "meeting"], property: {...} })                 │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、标签解析写入机制

### 2.1 前端标签解析（仅用于渲染）

**文件**: `web/src/utils/remark-plugins/remark-tag.ts`

前端使用 Remark 插件解析 `#tag` 语法，**仅用于渲染时的样式和交互**，不影响后端存储。

```typescript
// 标签字符验证规则
function isTagChar(char: string): boolean {
  if (/\p{L}/u.test(char)) return true;      // Unicode 字母
  if (/\p{N}/u.test(char)) return true;      // Unicode 数字
  if (/\p{S}/u.test(char)) return true;      // 符号 (包括 emoji)
  return char === "_" || char === "-" || char === "/" || char === "&";
}

// 解析规则
// - 必须以 # 开头，后跟有效标签字符
// - 排除: ## (标题)、#  (空格后)
// - 支持: #工作、#work/meeting、#work&life
```

### 2.2 后端标签解析（用于存储）

**文件**: `internal/markdown/parser/tag.go`

后端使用 **Goldmark** Markdown 解析器，通过自定义扩展解析 `#tag` 语法。这是**真正用于存储到数据库**的解析逻辑。

```go
// 标签字符验证（Unicode 感知）
func isValidTagRune(r rune) bool {
  if unicode.IsLetter(r) { return true }   // 任意语言字母
  if unicode.IsNumber(r) { return true }   // 任意语言数字
  if unicode.IsSymbol(r) { return true }   // 符号、emoji
  if unicode.IsMark(r) { return true }     // 组合标记 (变音符号等)
  if r == '\u200D' { return true }         // Zero Width Joiner (emoji 序列)
  if r == '_' || r == '-' || r == '/' || r == '&' { return true }
  return false
}

// 解析触发条件
func (*tagParser) Trigger() []byte {
  return []byte{'#'}  // 遇到 # 字符时触发
}

// 解析规则
// 1. 排除: ## (标题)、# 后面有空格
// 2. 连续读取有效字符，直到遇到无效字符或达到 100 字符限制
// 3. 支持层级标签: work/project/task (通过 / 分隔)
```

### 2.3 Payload 重建流程

**文件**: `server/runner/memopayload/runner.go`

```go
func RebuildMemoPayload(_ context.Context, memo *store.Memo, markdownService markdown.Service) error {
  if memo.Payload == nil {
    memo.Payload = &storepb.MemoPayload{}
  }

  // 单次解析提取所有元数据（效率更高）
  data, err := markdownService.ExtractAll([]byte(memo.Content))
  if err != nil {
    return errors.Wrap(err, "failed to extract markdown metadata")
  }

  // 更新 payload
  memo.Payload.Tags = data.Tags
  memo.Payload.Property = data.Property
  return nil
}
```

**文件**: `internal/markdown/markdown.go`

```go
func (s *service) ExtractAll(content []byte) (*ExtractedData, error) {
  root, err := s.parse(content)
  if err != nil { return nil, err }

  data := &ExtractedData{
    Tags:     []string{},
    Mentions: []string{},
    Property: &storepb.MemoPayload_Property{},
  }

  // 单次 AST 遍历收集所有数据
  err = gast.Walk(root, func(n gast.Node, entering bool) (gast.WalkStatus, error) {
    if !entering { return gast.WalkContinue, nil }

    // 提取标签
    if tagNode, ok := n.(*mast.TagNode); ok {
      data.Tags = append(data.Tags, string(tagNode.Tag))
    }

    // 提取 @mention
    if mentionNode, ok := n.(*mast.MentionNode); ok {
      data.Mentions = append(data.Mentions, strings.ToLower(string(mentionNode.Username)))
    }

    // 检查首块是否为 H1 标题（作为 title）
    if !firstBlockChecked && n.Parent() != nil && n.Parent().Kind() == gast.KindDocument {
      firstBlockChecked = true
      if heading, ok := n.(*gast.Heading); ok && heading.Level == 1 {
        data.Property.Title = extractHeadingText(n, content)
      }
    }

    // 提取属性
    switch n.Kind() {
    case gast.KindLink:
      data.Property.HasLink = true
    case gast.KindCodeBlock, gast.KindFencedCodeBlock, gast.KindCodeSpan:
      data.Property.HasCode = true
    case east.KindTaskCheckBox:
      data.Property.HasTaskList = true
      if checkBox, ok := n.(*east.TaskCheckBox); ok {
        if !checkBox.IsChecked {
          data.Property.HasIncompleteTasks = true
        }
      }
    }

    return gast.WalkContinue, nil
  })

  // 去重（保留原始大小写）
  data.Tags = uniquePreserveCase(data.Tags)
  data.Mentions = uniquePreserveCase(data.Mentions)

  return data, nil
}
```

### 2.4 存储结构

```go
// memo 表核心字段
type Memo struct {
  ID        int32
  CreatorID int32
  Content   string                    // 原始 Markdown 文本
  Payload   *storepb.MemoPayload      // 解析后的元数据
  // ... 其他字段
}

// Payload 结构 (存储为 JSON)
message MemoPayload {
  repeated string tags = 1;            // ["work", "meeting", "project/alpha"]
  Property property = 2;
}

message MemoPayload_Property {
  bool has_link = 1;
  bool has_code = 2;
  bool has_task_list = 3;
  bool has_incomplete_tasks = 4;
  string title = 5;                    // 首行 H1 标题
}
```

---

## 三、标签统计接口

### 3.1 GetUserStats 接口

**文件**: `server/router/api/v1/user_service_stats.go`

```go
func (s *APIV1Service) GetUserStats(ctx context.Context, request *v1pb.GetUserStatsRequest) (*v1pb.UserStats, error) {
  // ... 用户解析 ...

  // 查询用户所有笔记
  normalStatus := store.Normal
  memoFind := &store.FindMemo{
    CreatorID:       &userID,
    ExcludeComments: true,   // 排除评论
    ExcludeContent:  true,   // 不需要加载 content（标签在 payload 中）
    RowStatus:       &normalStatus,
  }

  // ... 权限过滤 ...

  // 统计变量
  tagCount := make(map[string]int32)
  totalMemoCount := int32(0)

  // 批量遍历所有笔记
  limit := 1000
  offset := 0
  memoFind.Limit = &limit
  memoFind.Offset = &offset

  for {
    memos, err := s.Store.ListMemos(ctx, memoFind)
    if err != nil { return nil, err }
    if len(memos) == 0 { break }

    totalMemoCount += int32(len(memos))

    for _, memo := range memos {
      // 从 payload 统计标签
      if memo.Payload != nil {
        for _, tag := range memo.Payload.Tags {
          tagCount[tag]++
        }
        // 统计属性...
      }
    }

    offset += limit
  }

  // 返回统计结果
  userStats := &v1pb.UserStats{
    Name:           fmt.Sprintf("%s/stats", BuildUserName(user.Username)),
    TagCount:       tagCount,              // 标签计数
    TotalMemoCount: totalMemoCount,
    // ... 其他字段
  }

  return userStats, nil
}
```

### 3.2 统计数据来源

| 统计项 | 数据来源 | 说明 |
|--------|----------|------|
| `TagCount` | `memo.Payload.Tags` | 从解析后的 JSON 字段统计 |
| `HasLink` | `memo.Payload.Property.HasLink` | 内容包含链接 |
| `HasCode` | `memo.Payload.Property.HasCode` | 内容包含代码块或行内代码 |
| `HasTaskList` | `memo.Payload.Property.HasTaskList` | 内容包含任务列表 |

**关键点**：
- 统计**不依赖** `content` 字段的实时解析
- 统计**依赖** `payload` 字段的预解析结果
- 这意味着如果 `payload` 过期，统计数据也会不准确

---

## 四、前端 tagCount 获取与展示

### 4.1 双数据源策略

**文件**: `web/src/hooks/useFilteredMemoStats.ts`

前端根据不同的 `context` 使用**不同的数据源**获取 tagCount：

```typescript
export const useFilteredMemoStats = (options: UseFilteredMemoStatsOptions = {}): FilteredMemoStats => {
  const { userName, context } = options;
  const currentUser = useCurrentUser();
  const { timeBasis } = useView();

  // home/profile: 使用后端统计接口（全量标签，无分页限制）
  const { data: userStats, isLoading: isLoadingUserStats } = useUserStats(userName);

  // explore: 使用 memo 列表实时统计（受 visibility filter 限制）
  const exploreVisibilityFilter = currentUser != null 
    ? 'visibility in ["PUBLIC", "PROTECTED"]' 
    : 'visibility in ["PUBLIC"]';
  const memoQueryParams = context === "explore" 
    ? { filter: exploreVisibilityFilter, pageSize: 1000 } 
    : {};
  const { data: memosResponse, isLoading: isLoadingMemos } = useMemos(memoQueryParams);

  // 选择数据源
  const data = useMemo(() => {
    let tagCount: Record<string, number> = {};

    if (context === "explore") {
      // 从获取的 memo 列表实时统计
      for (const memo of memosResponse?.memos ?? []) {
        for (const tag of memo.tags ?? []) {
          tagCount[tag] = (tagCount[tag] ?? 0) + 1;
        }
      }
    } else if (userName && userStats) {
      // 直接使用后端统计结果
      if (userStats.tagCount) {
        tagCount = userStats.tagCount;
      }
    } else if (memosResponse?.memos) {
      // fallback: 从 memo 列表统计
      for (const memo of memosResponse.memos) {
        for (const tag of memo.tags ?? []) {
          tagCount[tag] = (tagCount[tag] || 0) + 1;
        }
      }
    }

    return { statistics: { activityStats, timeBasis }, tags: tagCount, loading };
  }, [context, userName, userStats, memosResponse, ...]);

  return data;
};
```

### 4.2 数据源对比

| Context | 数据源 | 统计范围 | 分页限制 | 说明 |
|---------|--------|----------|----------|------|
| `home` | `userStats.tagCount` | 用户所有笔记 | 无 | 后端全量统计，最准确 |
| `profile` | `userStats.tagCount` | 用户所有笔记 | 无 | 同上 |
| `explore` | `memosResponse.memos` 实时统计 | 公开/保护笔记 | 1000 条 | 受 visibility filter 限制 |
| `archived` | `memosResponse.memos` 实时统计 | 当前用户归档笔记 | 默认 | fallback 路径 |

### 4.3 Query Key 结构

**文件**: `web/src/hooks/useUserQueries.ts`

```typescript
export const userKeys = {
  all: ["users"] as const,
  stats: () => [...userKeys.all, "stats"] as const,
  userStats: (name: string) => [...userKeys.stats(), name] as const,
  // ...
};

// useUserStats 使用的 Query Key
// - 特定用户: ["users", "stats", "users/steven"]
// - 未指定:   ["users", "stats"]
```

**文件**: `web/src/hooks/useMemoQueries.ts`

```typescript
export const memoKeys = {
  all: ["memos"] as const,
  lists: () => [...memoKeys.all, "list"] as const,
  list: (filters: Partial<ListMemosRequest>) => [...memoKeys.lists(), filters] as const,
  // ...
};

// useMemos / useInfiniteMemos 使用的 Query Key
// 示例: ["memos", "list", { filter: "tag in ['work']", pageSize: 20 }]
```

**关键差异**：
- **userStats** 的 Query Key **不包含**搜索/标签过滤条件
- **memo list** 的 Query Key **包含**完整的 filter 参数
- 这是导致一致性问题的根本原因之一

### 4.4 展示层

**文件**: `web/src/components/MemoExplorer/MemoExplorer.tsx`

```typescript
interface Props {
  className?: string;
  context?: MemoExplorerContext;
  features?: MemoExplorerFeatures;
  statisticsData: StatisticsData;
  tagCount: Record<string, number>;  // 接收 tagCount
}

const MemoExplorer = (props: Props) => {
  const { ... tagCount } = props;

  return (
    <aside>
      {/* 搜索框 */}
      {features.search && <SearchBar />}
      
      <div className="mt-1 px-1 w-full">
        {/* 统计视图 */}
        {features.statistics && <StatisticsView statisticsData={statisticsData} />}
        
        {/* 快捷方式 */}
        {features.shortcuts && currentUser && <ShortcutsSection />}
        
        {/* 标签列表（使用 tagCount） */}
        {features.tags && <TagsSection readonly={context === "explore"} tagCount={tagCount} />}
      </div>
    </aside>
  );
};
```

**文件**: `web/src/components/MemoExplorer/TagsSection.tsx`

```typescript
// TagsSection 接收 tagCount，按使用频率排序展示
const TagsSection = ({ tagCount, readonly = false }: { tagCount: Record<string, number>; readonly?: boolean }) => {
  const sortedTags = useMemo(() => {
    return Object.entries(tagCount)
      .sort((a, b) => b[1] - a[1])  // 按计数降序
      .map(([tag]) => tag);
  }, [tagCount]);

  // ... 渲染标签列表
};
```

---

## 五、标签变更时的联动刷新机制

### 5.1 刷新触发点汇总

标签变更可能由以下操作触发：

| 操作 | 触发方式 | 刷新内容 |
|------|----------|----------|
| 创建笔记 | mutation onSuccess | `memoKeys.lists()` + `userKeys.stats()` |
| 更新笔记 | mutation onSuccess | `memoKeys.lists()` + `userKeys.stats()` |
| 删除笔记 | mutation onSuccess | `memoKeys.lists()` + `userKeys.stats()` |
| 其他设备更新 | SSE `memo.updated` | `memoKeys.detail()` + `memoKeys.lists()` |
| 其他设备创建 | SSE `memo.created` | `memoKeys.lists()` + `userKeys.stats()` |
| 其他设备删除 | SSE `memo.deleted` | `memoKeys.lists()` + `userKeys.stats()` |

### 5.2 Mutation 刷新逻辑

**文件**: `web/src/hooks/useMemoQueries.ts`

```typescript
export function useCreateMemo() {
  return useMutation({
    mutationFn: async (memoToCreate: Memo) => {
      return await memoServiceClient.createMemo({ memo: memoToCreate });
    },
    onSuccess: (newMemo) => {
      // 1. 刷新所有 memo 列表
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      
      // 2. 添加到详情缓存
      queryClient.setQueryData(memoKeys.detail(newMemo.name), newMemo);
      
      // 3. 刷新用户统计（包括 tagCount）
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });  // ✅
    },
  });
}

export function useUpdateMemo() {
  return useMutation({
    mutationFn: async ({ update, updateMask }) => {
      return await memoServiceClient.updateMemo({ memo: update, updateMask });
    },
    onMutate: async ({ update }) => {
      // 乐观更新...
    },
    onSuccess: (updatedMemo) => {
      // 1. 更新详情缓存
      queryClient.setQueryData(memoKeys.detail(updatedMemo.name), updatedMemo);
      
      // 2. 乐观更新列表缓存
      patchMemoInCollectionQueries(queryClient, updatedMemo);
      
      // 3. 刷新所有列表
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      
      // 4. 刷新用户统计（包括 tagCount）
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });  // ✅
    },
  });
}

export function useDeleteMemo() {
  return useMutation({
    mutationFn: async (name: string) => {
      return await memoServiceClient.deleteMemo({ name });
    },
    onSuccess: (name) => {
      // 1. 移除详情缓存
      queryClient.removeQueries({ queryKey: memoKeys.detail(name) });
      
      // 2. 刷新列表
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      
      // 3. 刷新统计
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });  // ✅
    },
  });
}
```

**主动编辑时的刷新是完整的**：
- ✅ `memoKeys.lists()` - 所有列表
- ✅ `userKeys.stats()` - 所有统计（包括 tagCount）

### 5.3 SSE 实时刷新逻辑

**文件**: `web/src/hooks/useLiveMemoRefresh.ts`

```typescript
// SSE 事件类型
const SSE_EVENT_TYPES = {
  memoCreated: "memo.created",
  memoUpdated: "memo.updated",
  memoDeleted: "memo.deleted",
  memoCommentCreated: "memo.comment.created",
  reactionUpserted: "reaction.upserted",
  reactionDeleted: "reaction.deleted",
} as const;

// 事件处理函数
function handleSSEEvent(event: SSEChangeEvent, queryClient: ReturnType<typeof useQueryClient>) {
  switch (event.type) {
    case SSE_EVENT_TYPES.memoCreated:
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });  // ✅
      break;

    case SSE_EVENT_TYPES.memoUpdated:
      queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      // ⚠️ 注意：这里没有刷新 userKeys.stats()！
      if (event.parent) {
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
      }
      break;

    case SSE_EVENT_TYPES.memoDeleted:
      queryClient.removeQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });  // ✅
      break;

    // ... 其他事件
  }
}
```

**问题发现**：
- ✅ `memo.created` - 刷新 `userKeys.stats()`
- ⚠️ `memo.updated` - **没有刷新** `userKeys.stats()`
- ✅ `memo.deleted` - 刷新 `userKeys.stats()`

### 5.4 SSE memo.updated 不刷新统计的影响

**场景**：
1. 用户 A 在设备 1 上编辑笔记，添加了一个新标签 `#new-tag`
2. 后端处理：重建 payload，更新 `memo.Payload.Tags`
3. SSE 推送 `memo.updated` 事件到设备 2
4. 设备 2 的处理：
   - ✅ 刷新 `memoKeys.detail(event.name)` - 笔记详情更新
   - ✅ 刷新 `memoKeys.lists()` - 列表内容更新
   - ❌ **不刷新** `userKeys.stats()` - tagCount 不更新

**结果**：
- 设备 2 的列表内容会显示新标签
- 设备 2 的 **TagsSection tagCount 不会更新**（仍然缺少 `#new-tag`）
- 只有当设备 2 主动刷新页面或触发其他刷新时，tagCount 才会更新

---

## 六、与搜索过滤联动时的一致性核对

### 6.1 问题场景

这是本报告的核心问题：**当用户使用标签或搜索过滤时，列表内容和 TagsSection 的 tagCount 是否一致？**

让我们通过一个具体示例分析：

#### 示例数据

假设用户有以下笔记：

| 笔记 ID | 内容 | 标签 |
|---------|------|------|
| 1 | 今天的工作会议 #work #meeting | ["work", "meeting"] |
| 2 | 明天的工作安排 #work #todo | ["work", "todo"] |
| 3 | 个人购物清单 #personal | ["personal"] |
| 4 | 项目计划 #work #project | ["work", "project"] |

**全局 tagCount**：
- work: 3
- meeting: 1
- todo: 1
- personal: 1
- project: 1

#### 场景 1：用户点击标签 "work" 进行过滤

**当前行为**：

1. **列表内容**（PagedMemoList）：
   - Filter: `tag in ["work"]`
   - 显示笔记 1、2、4（正确）
   - Query Key: `["memos", "list", { filter: "tag in ['work']" }]`

2. **TagsSection**：
   - `home/profile` 上下文：使用 `userStats.tagCount`
     - 显示: work: 3, meeting: 1, todo: 1, personal: 1, project: 1
     - **这是全局统计，不是过滤结果的统计**
   - `explore` 上下文：从 `useMemos({ filter: visibilityFilter })` 统计
     - 同样是**全局可见性范围内的统计**，不是当前搜索过滤的结果

**不一致性**：
- 列表显示：3 条笔记（含 work 标签）
- TagsSection 显示：所有标签的全局计数
- 用户可能期望：只显示这 3 条笔记中出现的标签及其计数

#### 场景 2：用户搜索关键词 "会议"

**当前行为**：

1. **列表内容**：
   - Filter: `content.contains("会议")`
   - 显示笔记 1（正确）

2. **TagsSection**：
   - 仍然显示全局统计：work: 3, meeting: 1, todo: 1, personal: 1, project: 1
   - 实际上只有笔记 1 匹配，它的标签是 ["work", "meeting"]

**不一致性更明显**：
- 用户搜索"会议"，只有 1 条笔记匹配
- 但 TagsSection 显示所有标签，包括 personal（根本不在匹配结果中）

### 6.2 根本原因分析

**文件**: `web/src/hooks/useFilteredMemoStats.ts`

```typescript
// explore 上下文的 memo 查询参数
const exploreVisibilityFilter = currentUser != null 
  ? 'visibility in ["PUBLIC", "PROTECTED"]' 
  : 'visibility in ["PUBLIC"]';
const memoQueryParams = context === "explore" 
  ? { filter: exploreVisibilityFilter, pageSize: 1000 }  // ⚠️ 固定的 visibility filter
  : {};
const { data: memosResponse, isLoading: isLoadingMemos } = useMemos(memoQueryParams);
```

**问题**：
1. **home/profile 上下文**：使用 `userStats.tagCount`
   - 后端统计接口**不接受** filter 参数
   - 返回的是**用户所有笔记**的统计
   - 无法感知前端的搜索/标签过滤

2. **explore 上下文**：使用 `useMemos({ filter: exploreVisibilityFilter })`
   - filter 是**固定的** visibility 条件
   - **不包含**当前的搜索/标签过滤条件
   - Query Key 与列表使用的不同

**Query Key 对比**：

| 用途 | Query Key | filter 参数 |
|------|-----------|-------------|
| 列表内容 (home) | `["memos", "list", { filter: "tag in ['work']", ... }]` | 包含标签/搜索过滤 |
| 列表内容 (explore) | `["memos", "list", { filter: "tag in ['work']", ... }]` | 包含标签/搜索过滤 |
| tagCount (home) | `["users", "stats", "users/steven"]` | 无 filter |
| tagCount (explore) | `["memos", "list", { filter: "visibility in [...]" }]` | 只有 visibility |

### 6.3 另一个潜在问题：分页限制

**文件**: `web/src/hooks/useFilteredMemoStats.ts` 第 44 行

```typescript
const memoQueryParams = context === "explore" 
  ? { filter: exploreVisibilityFilter, pageSize: 1000 }  // ⚠️ pageSize: 1000
  : {};
```

**问题**：
- `pageSize: 1000` - 如果公开笔记超过 1000 条
- `useMemos` 只会获取**第一页**
- `tagCount` 统计基于**不完整的数据**
- 而列表内容（使用 `useInfiniteMemos`）会正确加载所有分页

**影响**：
- explore 上下文下，笔记超过 1000 条时 tagCount 不准确
- home/profile 上下文（使用后端统计）没有这个问题

### 6.4 设计意图 vs 用户体验

**可能的设计意图**：

1. **全局标签导航**：
   - "让用户看到所有可用的标签，然后选择其中一个进行过滤"
   - 类似电商网站的筛选器：显示所有品牌，点击后过滤

2. **技术限制**：
   - 后端 `GetUserStats` 接口不支持 filter 参数
   - 添加支持需要修改 proto 定义和后端实现

**用户体验问题**：

1. **认知不一致**：
   - 用户看到列表只有 3 条笔记
   - 但 TagsSection 显示 personal 标签有 1 条
   - 用户可能会困惑："personal 标签的笔记在哪里？"

2. **无法进行二次筛选**：
   - 用户搜索"会议"后，希望进一步在结果中按标签筛选
   - 但 TagsSection 显示的是全局标签，不是搜索结果内的标签

---

## 七、一致性问题汇总

### 7.1 问题清单

| 问题 ID | 问题描述 | 位置 | 影响程度 |
|---------|----------|------|----------|
| P1 | SSE `memo.updated` 事件不刷新 `userKeys.stats()` | `useLiveMemoRefresh.ts` 第 207-213 行 | 中 |
| P2 | 搜索/标签过滤时，tagCount 是全局统计而非过滤结果统计 | `useFilteredMemoStats.ts` | 高 |
| P3 | explore 上下文 tagCount 受 `pageSize: 1000` 限制 | `useFilteredMemoStats.ts` 第 44 行 | 中 |
| P4 | home/profile 与 explore 上下文数据源不一致 | `useFilteredMemoStats.ts` | 低（设计差异） |

### 7.2 问题 P1 详细分析

**问题**：SSE `memo.updated` 事件处理中没有刷新 `userKeys.stats()`

**代码位置**：`web/src/hooks/useLiveMemoRefresh.ts`

```typescript
case SSE_EVENT_TYPES.memoUpdated:
  queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
  queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
  // ❌ 缺少: queryClient.invalidateQueries({ queryKey: userKeys.stats() });
  if (event.parent) {
    queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
  }
  break;
```

**影响**：
- 用户在其他设备编辑笔记（添加/删除标签）
- 当前设备的列表内容会更新
- 但当前设备的 **TagsSection tagCount 不会更新**
- 直到用户主动刷新或触发其他刷新操作

**修复建议**：
```typescript
case SSE_EVENT_TYPES.memoUpdated:
  queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
  queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
  queryClient.invalidateQueries({ queryKey: userKeys.stats() });  // ✅ 添加
  if (event.parent) {
    queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
  }
  break;
```

### 7.3 问题 P2 详细分析

**问题**：搜索/标签过滤时，tagCount 与列表内容不一致

**根本原因**：

| 组件 | 数据源 | 是否考虑当前过滤条件 |
|------|--------|---------------------|
| PagedMemoList (列表) | `useInfiniteMemos({ filter: 条件 })` | ✅ 是 |
| TagsSection (home) | `userStats.tagCount` | ❌ 否 |
| TagsSection (explore) | `useMemos({ filter: visibilityFilter })` | ❌ 否 |

**两种可能的解决方案**：

#### 方案 A：让 tagCount 反映当前过滤结果（推荐用于用户体验）

修改 `useFilteredMemoStats`，让它使用与列表相同的 filter 条件：

```typescript
// 伪代码思路
export const useFilteredMemoStats = (options: { 
  userName?: string; 
  context?: MemoExplorerContext;
  currentFilter?: string;  // 新增：当前过滤条件
}) => {
  // ...
  
  // 如果有当前过滤条件，使用该条件统计
  if (currentFilter) {
    // 使用与列表相同的 filter 查询 memo
    const { data: filteredMemos } = useMemos({ filter: currentFilter, pageSize: 10000 });
    
    // 从过滤结果统计 tagCount
    for (const memo of filteredMemos?.memos ?? []) {
      for (const tag of memo.tags ?? []) {
        tagCount[tag] = (tagCount[tag] ?? 0) + 1;
      }
    }
  } else {
    // 使用原有逻辑
  }
};
```

**优点**：
- 用户体验一致
- 支持"在结果中继续筛选"的交互模式

**缺点**：
- 额外的 API 请求
- 如果结果很大，分页处理复杂

#### 方案 B：保持全局统计，但明确区分（当前设计）

保持当前行为，但确保用户理解：
- TagsSection 显示的是**全局标签统计**
- 点击标签是**添加过滤条件**，不是"在结果中筛选"

**优点**：
- 实现简单
- 后端统计接口高效

**缺点**：
- 存在认知不一致
- 无法进行二次筛选

### 7.4 问题 P3 详细分析

**问题**：explore 上下文 tagCount 受 `pageSize: 1000` 限制

**代码位置**：`web/src/hooks/useFilteredMemoStats.ts`

```typescript
const memoQueryParams = context === "explore" 
  ? { filter: exploreVisibilityFilter, pageSize: 1000 }  // ⚠️
  : {};
```

**影响**：
- 如果公开笔记超过 1000 条
- `useMemos` 只返回第一页
- tagCount 统计不完整

**修复建议**：

方案 A：使用后端统计接口（需要修改接口支持）
- 让 `ListAllUserStats` 或新接口接受 visibility filter
- 返回正确的统计

方案 B：增加 pageSize 或使用无限滚动
- 风险：数据量大时性能问题

方案 C：使用不同的数据源
- 让 explore 上下文也使用某种形式的后端统计

---

## 八、完整数据流时序图

### 8.1 创建笔记时的标签流程

```
┌──────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│  用户    │     │ MemoEditor   │     │ useCreateMemo  │     │  React Query │
└────┬─────┘     └──────┬───────┘     └───────┬────────┘     └──────┬───────┘
     │                  │                      │                     │
     │ 输入: "#work 新笔记" │                      │                     │
     │─────────────────►│                      │                     │
     │                  │                      │                     │
     │                  │  调用 createMemo      │                     │
     │                  │─────────────────────►│                     │
     │                  │                      │                     │
     │                  │                      │  gRPC: CreateMemo   │
     │                  │                      │────────────────────►│
     │                  │                      │                     │
     │                  │                      │  后端处理:          │
     │                  │                      │  - ExtractAll 解析   │
     │                  │                      │  - 重建 Payload     │
     │                  │                      │  - 存储到数据库      │
     │                  │                      │                     │
     │                  │                      │  返回: newMemo       │
     │                  │                      │◄────────────────────│
     │                  │                      │                     │
     │                  │                      │  onSuccess:          │
     │                  │                      │  - invalidate lists  │
     │                  │                      │  - invalidate stats  │
     │                  │                      │────────────────────►│
     │                  │                      │                     │
     │                  │                      │  缓存失效:           │
     │                  │                      │  - memoKeys.lists() │
     │                  │                      │  - userKeys.stats()  │
     │                  │                      │                     │
     │                  │                      │  重新获取数据:        │
     │                  │                      │  - ListMemos        │
     │                  │                      │  - GetUserStats     │
     │                  │                      │◄────────────────────│
     │                  │                      │                     │
     │  列表更新         │                      │                     │
     │◄─────────────────────────────────────────────────────────────│
     │                  │                      │                     │
     │  TagsSection 更新 │                      │                     │
     │◄─────────────────────────────────────────────────────────────│
```

### 8.2 SSE memo.updated 事件流程（当前有问题的）

```
┌──────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│ 设备 A   │     │   后端 API   │     │   SSE Hub      │     │   设备 B     │
└────┬─────┘     └──────┬───────┘     └───────┬────────┘     └──────┬───────┘
     │                  │                      │                     │
     │ 编辑笔记:          │                      │                     │
     │ 内容: "#work #todo" │                      │                     │
     │─────────────────►│                      │                     │
     │                  │                      │                     │
     │                  │  RebuildMemoPayload   │                     │
     │                  │  - 解析新标签          │                     │
     │                  │  - 更新 Payload.Tags   │                     │
     │                  │                      │                     │
     │                  │  广播 SSE 事件:        │                     │
     │                  │  type: "memo.updated" │                     │
     │                  │─────────────────────►│                     │
     │                  │                      │                     │
     │                  │                      │  推送事件到设备 B    │
     │                  │                      │────────────────────►│
     │                  │                      │                     │
     │                  │                      │                     │  处理事件:
     │                  │                      │                     │
     │                  │                      │                     │  invalidate:
     │                  │                      │                     │  - memoKeys.detail
     │                  │                      │                     │  - memoKeys.lists
     │                  │                      │                     │
     │                  │                      │                     │  ⚠️ 不 invalidate:
     │                  │                      │                     │  - userKeys.stats
     │                  │                      │                     │
     │                  │                      │                     │  结果:
     │                  │                      │                     │  ✅ 列表内容更新
     │                  │                      │                     │  ❌ tagCount 不更新
```

---

## 九、关键文件索引

### 后端文件

| 文件路径 | 职责 |
|----------|------|
| `internal/markdown/parser/tag.go` | 后端 #tag 语法解析器 |
| `internal/markdown/markdown.go` | ExtractAll 函数（解析标签、属性等） |
| `server/runner/memopayload/runner.go` | RebuildMemoPayload（重建 payload） |
| `server/router/api/v1/user_service_stats.go` | GetUserStats 统计接口 |

### 前端文件

| 文件路径 | 职责 |
|----------|------|
| `web/src/hooks/useFilteredMemoStats.ts` | tagCount 双数据源策略 |
| `web/src/hooks/useMemoQueries.ts` | memo 查询 + mutations + 缓存刷新 |
| `web/src/hooks/useUserQueries.ts` | userStats 查询 + tagCounts 聚合 |
| `web/src/hooks/useLiveMemoRefresh.ts` | SSE 实时刷新处理 |
| `web/src/components/MemoExplorer/MemoExplorer.tsx` | 接收 tagCount 并传给 TagsSection |
| `web/src/components/MemoExplorer/TagsSection.tsx` | 标签列表展示 + 点击过滤 |
| `web/src/utils/remark-plugins/remark-tag.ts` | 前端渲染用的标签解析 |

---

## 十、总结

### 10.1 标签完整链路

```
用户输入 #tag
    │
    ▼
MemoEditor 发送请求 (content 字段)
    │
    ▼
后端 CreateMemo/UpdateMemo
    │
    ▼
RebuildMemoPayload
    │
    ├──► MarkdownService.ExtractAll(content)
    │       ├── 解析 #tag → Tags[]
    │       └── 解析属性 → Property
    │
    ▼
存储到 memo.Payload (JSON 字段)
    │
    ▼
GetUserStats 统计接口
    │
    ├──► 遍历用户所有 memo
    └──► 从 memo.Payload.Tags 统计计数
    │
    ▼
前端 useUserStats 获取 tagCount
    │
    ▼
MemoExplorer → TagsSection 展示
```

### 10.2 一致性问题结论

| 问题 | 状态 | 建议 |
|------|------|------|
| SSE `memo.updated` 不刷新统计 | **Bug** | 添加 `invalidateQueries({ queryKey: userKeys.stats() })` |
| 搜索过滤时 tagCount 不一致 | **设计选择** | 需要确认产品意图。如果是问题，需要修改 `useFilteredMemoStats` 或后端接口 |
| explore 上下文 pageSize 限制 | **潜在 Bug** | 考虑使用后端统计接口或调整分页策略 |

### 10.3 核心设计特点

1. **前后端分离解析**：
   - 前端：Remark 插件（仅用于渲染）
   - 后端：Goldmark 扩展（用于存储和统计）
   - 两者解析规则一致

2. **Payload 预解析**：
   - 标签不依赖 `content` 字段的实时解析
   - 存储在 `memo.Payload.Tags` JSON 数组中
   - 统计高效，但需要确保 payload 同步更新

3. **双数据源统计**：
   - home/profile：后端 `GetUserStats` 接口（准确、高效）
   - explore：前端 `useMemos` 实时统计（受 visibility filter 限制）

4. **缓存刷新策略**：
   - 主动操作（创建/更新/删除）：完整刷新列表 + 统计
   - SSE 推送：列表刷新完整，但 `memo.updated` 事件漏刷新统计
