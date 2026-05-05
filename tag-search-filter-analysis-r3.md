# Memos Profile 场景标签统计一致性分析报告

## 概述

本文档聚焦分析 **Profile 场景**（访问 `/u/{username}` 页面）中，当 `statsUserName` 未就绪时，`tagCount` 回退到默认列表查询的**触发条件**、**持续窗口**和**用户可见影响**，并补充分析与搜索过滤状态同步时可能出现的**短暂不一致**问题。

---

## 一、核心组件与数据流

### 1.1 涉及组件

| 组件 | 位置 | 职责 |
|------|------|------|
| `MainLayout` | `web/src/layouts/MainLayout.tsx` | 管理 `statsUserName`、渲染 `MemoExplorer` |
| `UserProfile` | `web/src/pages/UserProfile.tsx` | 管理列表数据、`creatorName` 过滤 |
| `useFilteredMemoStats` | `web/src/hooks/useFilteredMemoStats.ts` | 双数据源选择逻辑 |
| `MemoExplorer` | `web/src/components/MemoExplorer/MemoExplorer.tsx` | 展示 `tagCount` 给 `TagsSection` |

### 1.2 关键数据流对比

**Profile 场景有两条独立的数据流**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据流 A：列表数据                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  UserProfile.tsx                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  const { data: user, isLoading } = useUser("users/steven");       │    │
│  │                                                                      │    │
│  │  // isLoading 时返回 null（不渲染列表）                               │    │
│  │  if (isLoading) return null;                                        │    │
│  │                                                                      │    │
│  │  // user 就绪后才渲染                                                 │    │
│  │  const memoFilter = useMemoFilters({                                │    │
│  │    creatorName: user?.name,  // "users/steven"                      │    │
│  │    includePinned: true,                                              │    │
│  │  });                                                                  │    │
│  │  // 生成 filter: "creator_id == steven_id"                          │    │
│  │                                                                      │    │
│  │  <PagedMemoList filter={memoFilter} />  // ✅ 只显示 steven 的笔记   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据流 B：tagCount 数据                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MainLayout.tsx (无 isLoading 检查！)                                        │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  const [profileUserName, setProfileUserName] = useState(undefined);│    │
│  │                                                                      │    │
│  │  // useEffect 中异步获取 user                                         │    │
│  │  useEffect(() => {                                                   │    │
│  │    userServiceClient.getUser({ name: "users/steven" })             │    │
│  │      .then((user) => setProfileUserName(user.name));                │    │
│  │  }, [...]);                                                          │    │
│  │                                                                      │    │
│  │  const statsUserName = useMemo(() => {                              │    │
│  │    if (context === "profile") return profileUserName;  // undefined │    │
│  │  }, [...]);                                                          │    │
│  │                                                                      │    │
│  │  // 立即渲染，不等待 statsUserName 就绪                                │    │
│  │  const { tags } = useFilteredMemoStats({                           │    │
│  │    userName: statsUserName,  // undefined                            │    │
│  │    context: "profile",                                               │    │
│  │  });                                                                  │    │
│  │                                                                      │    │
│  │  <MemoExplorer tagCount={tags} />  // ⚠️ 使用 fallback 数据          │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 关键差异点

| 特性 | 数据流 A（列表） | 数据流 B（tagCount） |
|------|------------------|----------------------|
| 加载状态检查 | ✅ `if (isLoading) return null` | ❌ **无检查，立即渲染** |
| creator 过滤 | ✅ `creatorName: user?.name` | ❌ **fallback 时无过滤** |
| 数据一致性 | 始终正确 | **T2-T3 阶段不一致** |

---

## 二、触发条件分析

### 2.1 useFilteredMemoStats 的数据源选择逻辑

**文件**: `web/src/hooks/useFilteredMemoStats.ts`

```typescript
export const useFilteredMemoStats = (options: UseFilteredMemoStatsOptions = {}): FilteredMemoStats => {
  const { userName, context } = options;

  // 数据源 1：后端统计接口（优先）
  const { data: userStats, isLoading: isLoadingUserStats } = useUserStats(userName);

  // 数据源 2：memo 列表查询（fallback）
  const exploreVisibilityFilter = currentUser != null 
    ? 'visibility in ["PUBLIC", "PROTECTED"]' 
    : 'visibility in ["PUBLIC"]';
  
  // ⚠️ 关键问题：profile 场景的 memoQueryParams 是空的！
  const memoQueryParams = context === "explore" 
    ? { filter: exploreVisibilityFilter, pageSize: 1000 }   // explore 有 filter
    : {};                                                     // profile/home/archived 无 filter
  
  const { data: memosResponse, isLoading: isLoadingMemos } = useMemos(memoQueryParams);

  // 选择逻辑
  const data = useMemo(() => {
    let tagCount: Record<string, number> = {};

    if (context === "explore") {
      // 分支 1：explore 场景 - 使用 visibility filter 的 memo 列表
      for (const memo of memosResponse?.memos ?? []) {
        for (const tag of memo.tags ?? []) {
          tagCount[tag] = (tagCount[tag] ?? 0) + 1;
        }
      }
    } else if (userName && userStats) {
      // 分支 2：home/profile - 使用后端统计（优先）
      if (userStats.tagCount) {
        tagCount = userStats.tagCount;  // ✅ 正确数据
      }
    } else if (memosResponse?.memos) {
      // 分支 3：fallback - 使用无 filter 的 memo 列表
      // ⚠️ 这是 profile 场景未就绪时的分支！
      for (const memo of memosResponse.memos) {
        for (const tag of memo.tags ?? []) {
          tagCount[tag] = (tagCount[tag] || 0) + 1;
        }
      }
    }

    return { statistics: { activityStats, timeBasis }, tags: tagCount, loading };
  }, [context, userName, userStats, memosResponse, ...]);
};
```

### 2.2 三个分支的触发条件

| 分支 | 条件 | 数据来源 | 数据范围 |
|------|------|----------|----------|
| **分支 1** | `context === "explore"` | `useMemos({ filter: visibilityFilter, pageSize: 1000 })` | 公开/保护的笔记 |
| **分支 2** | `context !== "explore"` **且** `userName && userStats` | `useUserStats(userName)` → 后端 `GetUserStats` | ✅ 指定用户的所有笔记 |
| **分支 3** | `context !== "explore"` **且** `!(userName && userStats)` **且** `memosResponse?.memos` | `useMemos({})`（无参数） | ⚠️ 见下方分析 |

### 2.3 分支 3 的数据范围（关键 Bug）

**问题代码**：
```typescript
const memoQueryParams = context === "explore" 
  ? { filter: exploreVisibilityFilter, pageSize: 1000 }   // explore 有 filter
  : {};                                                     // profile 场景是空的！
```

**Profile 场景下**：
- `memoQueryParams = {}`
- `useMemos({})` 调用 `ListMemosRequest` 时**没有 filter 参数**

**后端 ListMemos 无 filter 时的行为**：

**文件**: `server/router/api/v1/memo_service.go`（参考之前的分析）

```go
func (s *APIV1Service) ListMemos(ctx context.Context, request *v1pb.ListMemosRequest) (*v1pb.ListMemosResponse, error) {
  // ...
  
  // 权限过滤：基于用户身份
  if currentUser == nil {
    // 未登录用户只能看公开笔记
    memoFind.VisibilityList = []store.Visibility{store.Public}
  } else {
    if memoFind.CreatorID == nil {
      // 登录用户可以看：
      // - 自己的笔记 (creator_id == currentUser.ID)
      // - 公开/保护的笔记 (visibility in ["PUBLIC", "PROTECTED"])
      filter := fmt.Sprintf(
        `creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, 
        currentUser.ID
      )
      memoFind.Filters = append(memoFind.Filters, filter)
    }
    // ...
  }
  // ...
}
```

**数据范围分析**：

| 当前登录用户 | 访问路径 | fallback 数据范围 | 期望数据范围 |
|--------------|----------|-------------------|--------------|
| `alice` (已登录) | `/u/steven` | `alice` 的笔记 + 公开/保护笔记 | `steven` 的笔记 |
| `steven` (已登录) | `/u/steven` | `steven` 的笔记 + 公开/保护笔记 | `steven` 的笔记 |
| 未登录 | `/u/steven` | 公开笔记 | `steven` 的公开笔记 |

**关键发现**：

当 `alice` 访问 `/u/steven` 时：
- **列表数据**（UserProfile）：只显示 `steven` 的笔记（正确）
- **tagCount**（fallback）：来自 `alice` 的笔记 + 公开笔记（**错误！**）

这是一个 **Bug**：fallback 时的数据范围完全不符合 profile 场景的预期。

### 2.4 回退触发的具体场景

Profile 场景下回退到分支 3 的条件：

```
条件 = !(userName && userStats)
     = (!userName) || (!userStats)
```

**两种情况**：

| 情况 | `userName` | `userStats` | 触发时机 |
|------|------------|-------------|----------|
| **情况 A** | `undefined` | - | T0-T2：`getUser` 请求未返回 |
| **情况 B** | `"users/steven"` | `undefined` | T2-T3：`getUserStats` 请求未返回 |

---

## 三、持续窗口时间线分析

### 3.1 完整时序图

```
时间轴 ─────────────────────────────────────────────────────────────────────►

T0: 页面加载
    │
    ├─── MainLayout ─────────────────────────────────────────────────────────
    │   ├── profileUserName = undefined (初始 state)
    │   ├── statsUserName = undefined
    │   ├── useFilteredMemoStats({ userName: undefined, context: "profile" })
    │   │   ├── useUserStats(undefined) → enabled: false，不执行
    │   │   ├── useMemos({}) → 发送 ListMemos 请求（无 filter）
    │   │   └── 等待 memosResponse
    │   └── MemoExplorer 已渲染（但数据为空或 loading）
    │
    ├─── UserProfile ────────────────────────────────────────────────────────
    │   ├── useUser("users/steven") → 发送 getUser 请求
    │   └── isLoading = true → return null（不显示列表）
    │
    │
T1: useEffect 执行（MainLayout）
    │
    ├─── MainLayout ─────────────────────────────────────────────────────────
    │   └── 发送 userServiceClient.getUser({ name: "users/steven" })
    │       注意：这是额外的请求！UserProfile 也在发同样的请求
    │
    │
T2: getUser 返回（两个组件的请求都返回）
    │
    ├─── MainLayout ─────────────────────────────────────────────────────────
    │   ├── setProfileUserName("users/steven")
    │   ├── statsUserName = "users/steven"
    │   ├── useFilteredMemoStats({ userName: "users/steven", context: "profile" })
    │   │   ├── useUserStats("users/steven") → enabled: true，发送 getUserStats
    │   │   ├── userStats = undefined（请求中）
    │   │   ├── memosResponse 可能已返回
    │   │   └── 条件：!("users/steven" && undefined) = true → 继续 fallback
    │   └── MemoExplorer 使用 fallback 数据
    │
    ├─── UserProfile ────────────────────────────────────────────────────────
    │   ├── isLoading = false
    │   ├── user = { name: "users/steven", ... }
    │   ├── useMemoFilters({ creatorName: "users/steven", ... })
    │   │   └── 生成 filter: "creator_id == steven_id"
    │   └── PagedMemoList 渲染 → ✅ 显示 steven 的笔记
    │
    │   ═══════════════════════════════════════════════════════════════════
    │   ⚠️ 关键不一致窗口（T2-T3）：
    │   - 列表显示 steven 的笔记（正确）
    │   - TagsSection 显示 fallback 数据（错误！）
    │   - 数据范围完全不匹配！
    │   ═══════════════════════════════════════════════════════════════════
    │
    │
T3: getUserStats 返回
    │
    ├─── MainLayout ─────────────────────────────────────────────────────────
    │   ├── userStats = { tagCount: { "work": 3, "meeting": 1, ... }, ... }
    │   ├── useFilteredMemoStats 重新计算
    │   │   └── 条件：!("users/steven" && userStats) = false → 使用分支 2
    │   ├── tagCount = userStats.tagCount（✅ steven 的统计）
    │   └── MemoExplorer 更新为正确数据
    │
    └─── 状态一致 ✅
```

### 3.2 各阶段状态总结

| 阶段 | 时间 | `statsUserName` | `userStats` | 分支选择 | 列表状态 | tagCount 状态 |
|------|------|-----------------|-------------|----------|----------|---------------|
| **T0-T1** | 页面加载 | `undefined` | - | 等待 memosResponse | 不显示 (null) | 空/loading |
| **T1-T2** | getUser 请求中 | `undefined` | - | 分支 3 (fallback) | 不显示 (null) | 错误范围数据 |
| **T2-T3** | getUserStats 请求中 | `"users/steven"` | `undefined` | 分支 3 (fallback) | ✅ 显示 steven 的笔记 | ⚠️ **错误数据** |
| **T3+** | 所有就绪 | `"users/steven"` | 有值 | 分支 2 (后端统计) | ✅ 正确 | ✅ 正确 |

### 3.3 关键不一致窗口（T2-T3）

**这是用户可见的主要问题**：

```
UserProfile（列表）                    MainLayout（MemoExplorer）
───────────────────────────────────    ───────────────────────────────────
显示 steven 的笔记：                    显示 fallback 的标签：
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│ 笔记 1: #work #meeting          │    │ Tags:                           │
│ 笔记 2: #work #todo             │    │  - personal (2)  ⬅️ 错误！     │
│ 笔记 3: #work #project          │    │  - shopping (1)  ⬅️ 错误！     │
└─────────────────────────────────┘    │  - work (1)      ⬅️ 数量错误   │
                                        └─────────────────────────────────┘
用户点击 "personal" 标签：
  - 列表过滤条件: creator_id == steven_id && tag in ["personal"]
  - 结果：空列表！（因为 steven 没有 personal 标签的笔记）
  - 用户体验困惑
```

### 3.4 持续时间估计

| 阶段 | 典型持续时间 | 说明 |
|------|-------------|------|
| T0-T2 | 100-500ms | 1 个网络请求（getUser） |
| **T2-T3** | **100-500ms** | **又 1 个网络请求（getUserStats）** |
| 总不一致窗口 | **200-1000ms** | 用户可感知 |

**额外问题**：
- `MainLayout` 和 `UserProfile` 都在发送 `getUser` 请求！
- 这是**重复请求**，可能延长总时间

---

## 四、搜索过滤状态同步时的短暂不一致

### 4.1 场景：用户已在 profile 页面（T3+ 之后）

假设用户已经在 `/u/steven` 页面，所有数据已就绪。现在用户执行搜索/过滤操作。

### 4.2 交互流程

```
用户点击标签 "work"（在 TagsSection 中）
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 步骤 1：MemoFilterContext 更新                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  filters = [{ factor: "tagSearch", value: "work" }]                        │
│  URL 同步: ?filter=tagSearch:work                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 步骤 2：列表数据更新（UserProfile）                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  const memoFilter = useMemoFilters({                                        │
│    creatorName: "users/steven",                                             │
│    includePinned: true,                                                      │
│  });                                                                          │
│                                                                              │
│  // 依赖项: [creatorName, includeShortcuts, includePinned,                  │
│  //           visibilities, selectedShortcut, filters]                      │
│  //                                                                          │
│  // filters 变化 → memoFilter 重新计算                                       │
│  memoFilter = "creator_id == steven_id && tag in ["work"]"                 │
│                                                                              │
│  PagedMemoList.filter = memoFilter                                          │
│  useInfiniteMemos({ filter: memoFilter })                                   │
│                                                                              │
│  // Query Key 变化：                                                         │
│  // 之前: ["memos", "list", { filter: "creator_id == steven_id" }]         │
│  // 之后: ["memos", "list", { filter: "creator_id == steven_id && ..." }] │
│  //                                                                          │
│  // 缓存失效 → 重新请求 → 列表更新                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 步骤 3：tagCount 数据（MainLayout）                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  const { tags } = useFilteredMemoStats({                                    │
│    userName: "users/steven",                                                 │
│    context: "profile",                                                       │
│  });                                                                          │
│                                                                              │
│  // 依赖项: [context, userName, userStats, memosResponse,                   │
│  //           isLoadingUserStats, isLoadingMemos, timeBasis]               │
│  //                                                                          │
│  // ⚠️ 注意：filters 不在依赖项中！                                          │
│  //                                                                          │
│  // 选择逻辑：                                                                │
│  //   context === "profile" → 不是 explore                                   │
│  //   userName && userStats → 是（都就绪）                                   │
│  //   → 使用分支 2：userStats.tagCount                                       │
│  //                                                                          │
│  // ✅ 但这是全局统计，不受当前过滤条件影响！                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 依赖项对比

| Hook | 依赖项 | 是否包含 `filters` |
|------|--------|-------------------|
| `useMemoFilters` | `[creatorName, ..., filters]` | ✅ 是 |
| `useFilteredMemoStats` | `[context, userName, userStats, memosResponse, ...]` | ❌ **否** |

### 4.4 结果状态

| 组件 | 过滤前 | 过滤后（点击 "work"） |
|------|--------|----------------------|
| **列表内容** | steven 的所有笔记 | steven 的 work 标签笔记（仅 3 条） |
| **TagsSection** | work: 3, meeting: 1, todo: 1, project: 1 | **work: 3, meeting: 1, todo: 1, project: 1**（无变化） |

### 4.5 这是设计选择还是 Bug？

**当前设计的意图**：
- TagsSection 显示**全局标签统计**
- 点击标签是**添加过滤条件**
- 类似电商网站的筛选器：显示所有品牌，点击后过滤

**用户体验问题**：
1. **认知不一致**：
   - 列表只有 3 条笔记（work 标签）
   - TagsSection 显示 meeting 有 1 条
   - 用户困惑："meeting 标签的笔记在哪里？"

2. **无法二次筛选**：
   - 用户搜索 "meeting" 后，希望在结果中按标签筛选
   - 但 TagsSection 显示的是全局标签，不是搜索结果内的标签

**对比：explore 场景**：

```typescript
// useFilteredMemoStats.ts 第 52-63 行
if (context === "explore") {
  // 从 memo 列表实时统计
  for (const memo of memosResponse?.memos ?? []) {
    for (const tag of memo.tags ?? []) {
      tagCount[tag] = (tagCount[tag] ?? 0) + 1;
    }
  }
}
```

但 `explore` 场景的 `memosResponse` 来自：
```typescript
const memoQueryParams = context === "explore" 
  ? { filter: exploreVisibilityFilter, pageSize: 1000 }  // 固定的 visibility filter
  : {};
```

**问题**：`explore` 场景的 `memoQueryParams` 也**不包含**当前的搜索/标签过滤条件！

所以**所有场景**下，`tagCount` 都不反映当前过滤结果。这是**设计选择**，但需要明确记录。

---

## 五、用户可见影响分析

### 5.1 问题分类

| 问题 ID | 问题类型 | 场景 | 用户可见 | 严重程度 |
|---------|----------|------|----------|----------|
| **P1** | Bug | Profile T2-T3 阶段 | ✅ 可见 | **高** |
| **P2** | 性能问题 | 重复 getUser 请求 | 不可见 | 中 |
| **P3** | 设计选择 | 过滤时 tagCount 不变 | ✅ 可见 | 低/中 |
| **P4** | 潜在问题 | getUserStats 失败时的 fallback | 可能可见 | 中 |

### 5.2 问题 P1 详细分析

**场景**：`alice` 访问 `/u/steven`

**T2-T3 阶段的数据对比**：

| 数据项 | 列表显示（正确） | TagsSection 显示（错误） |
|--------|------------------|--------------------------|
| 数据范围 | steven 的笔记 | alice 的笔记 + 公开笔记 |
| 标签来源 | steven 的 `memo.Payload.Tags` | alice 等的 `memo.Payload.Tags` |
| 点击标签结果 | 过滤 steven 的对应标签笔记 | 可能空列表（如果 alice 有 steven 没有的标签） |

**用户体验场景**：

1. **场景 A：alice 和 steven 有不同的标签**
   ```
   alice 的标签: personal, shopping
   steven 的标签: work, meeting, project
   
   列表显示: steven 的 work/meeting/project 笔记
   TagsSection 显示: personal (2), shopping (1)
   
   用户点击 "personal":
     → 过滤条件: creator_id == steven_id && tag in ["personal"]
     → 结果: 空列表
     → 用户: "为什么没有笔记？"
   ```

2. **场景 B：alice 和 steven 有相同标签但数量不同**
   ```
   alice 的 work 标签: 5 条笔记
   steven 的 work 标签: 3 条笔记
   
   列表显示: 3 条 work 笔记
   TagsSection 显示: work (5)
   
   用户困惑: "列表只有 3 条，为什么标签显示 5 条？"
   ```

3. **场景 C：用户是 steven 自己访问自己的 profile**
   ```
   此时 fallback 数据恰好是正确的（都是 steven 的笔记）
   但如果有其他公开笔记混入，仍然可能不一致
   ```

### 5.3 问题 P2：重复请求

**代码位置**：

- `MainLayout.tsx` 第 42-46 行：
  ```typescript
  userServiceClient
    .getUser({ name: `users/${username}` })
    .then((user) => setProfileUserName(user.name))
  ```

- `UserProfile.tsx` 第 78 行：
  ```typescript
  const { data: user, isLoading, error } = useUser(`users/${username}`, { enabled: !!username });
  ```

**问题**：
- 两个组件同时发送相同的 `getUser` 请求
- 浪费网络资源
- 可能延长总加载时间

**影响**：
- 性能问题（不可见）
- 但会延长 T0-T2 阶段，间接增加不一致窗口的持续时间

### 5.4 问题 P4：getUserStats 失败

**场景**：网络错误或服务器错误导致 `getUserStats` 请求失败

**后果**：
- `userStats` 保持 `undefined`
- `useFilteredMemoStats` 继续使用 fallback 分支
- fallback 数据是错误的（不是 profile 用户的）
- **用户会一直看到错误的标签统计**

---

## 六、关键代码索引

### 6.1 问题代码位置

| 问题 | 文件 | 行号 | 代码片段 |
|------|------|------|----------|
| **P1: fallback 无 creator 过滤** | `useFilteredMemoStats.ts` | 43-45 | `memoQueryParams = context === "explore" ? { ... } : {}` |
| **P1: 无 isLoading 检查** | `MainLayout.tsx` | 67-80 | 直接渲染 `MemoExplorer`，无等待 |
| **P2: 重复 getUser** | `MainLayout.tsx` | 42-46 | `userServiceClient.getUser(...)` |
| **P2: 重复 getUser** | `UserProfile.tsx` | 78 | `useUser("users/" + username)` |
| **P3: 无 filters 依赖** | `useFilteredMemoStats.ts` | 107 | 依赖项不含 `filters` |

### 6.2 关键逻辑

**数据源选择** (`useFilteredMemoStats.ts` 第 52-104 行)：
```typescript
if (context === "explore") {
  // 分支 1: explore 场景 - 使用 visibility filter
} else if (userName && userStats) {
  // 分支 2: 使用后端统计（正确数据）
} else if (memosResponse?.memos) {
  // 分支 3: fallback - 无 filter 的 memo 列表（问题所在！）
}
```

**Query Key 对比**：
```typescript
// 列表（UserProfile）
memoKeys.list({ filter: "creator_id == steven_id && tag in ['work']" })
// 包含完整的过滤条件

// tagCount fallback（MainLayout）
memoKeys.list({})  // 空参数！
// 不包含任何过滤条件
```

---

## 七、修复建议

### 7.1 问题 P1 的修复方案

**方案 A：让 MainLayout 也等待 user 就绪**

类似 `UserProfile.tsx` 的做法：
```typescript
// MainLayout.tsx
const { data: user, isLoading } = useUser(...);

// 可以选择：
// 1. isLoading 时不渲染 MemoExplorer
// 2. 或者显示 loading 状态
```

**但这不能解决根本问题**：`getUserStats` 仍然需要额外的请求，T2-T3 窗口仍然存在。

**方案 B：让 fallback 也包含 creator 过滤**

修改 `useFilteredMemoStats`，让 `profile` 场景的 fallback 也使用正确的 creator 过滤：

```typescript
// 伪代码思路
const memoQueryParams = useMemo(() => {
  if (context === "explore") {
    return { filter: exploreVisibilityFilter, pageSize: 1000 };
  }
  
  // profile/home 场景：如果有 creatorName，添加到 filter
  if (context === "profile" || context === "home") {
    const filters: string[] = [];
    
    // 添加 creator 过滤（如果有）
    if (userName) {
      const creatorFilter = buildMemoCreatorFilter(userName);
      if (creatorFilter) {
        filters.push(creatorFilter);
      }
    }
    
    return { 
      filter: filters.length > 0 ? filters.join(" && ") : undefined,
      pageSize: 1000 
    };
  }
  
  return {};  // archived 场景保持原样
}, [context, userName, ...]);
```

**问题**：`useFilteredMemoStats` 目前**不依赖** `userName` 来计算 `memoQueryParams`（看第 43-45 行，是固定的逻辑）。需要重构。

**方案 C：移除 fallback，始终等待后端统计**

修改选择逻辑：
```typescript
if (context === "explore") {
  // explore 保持不变
} else if (userName && userStats) {
  // 使用后端统计
} else if (userName && isLoadingUserStats) {
  // 显示 loading 状态
} else if (memosResponse?.memos) {
  // 只有 archived 场景才使用 fallback
}
```

这需要在 UI 上处理 loading 状态。

### 7.2 问题 P2 的修复方案

**方案：共享 user 数据**

- 让 `MainLayout` 也使用 `useUser` hook
- 或者通过 Context 共享 user 数据
- 避免重复请求

### 7.3 问题 P3 的设计选择

这需要产品确认：

- **选项 1：保持当前设计**
  - TagsSection 显示全局标签
  - 点击标签添加过滤条件

- **选项 2：过滤时更新 tagCount**
  - 让 tagCount 反映当前过滤结果
  - 类似"在结果中继续筛选"的交互

如果选择选项 2，需要：
1. 让 `useFilteredMemoStats` 接收当前 filter 参数
2. explore 场景的 `memoQueryParams` 需要包含当前过滤条件
3. 或者使用后端统计接口（需要修改接口支持 filter）

---

## 八、总结

### 8.1 核心发现

| 问题 | 类型 | 描述 |
|------|------|------|
| **1. Profile 场景 fallback 数据错误** | Bug | `useFilteredMemoStats` 在 profile 场景下的 fallback 使用 `useMemos({})`（无 filter），返回当前用户可见的所有笔记，不是 profile 用户的笔记 |
| **2. T2-T3 不一致窗口** | Bug | UserProfile 已经显示正确列表时，MemoExplorer 仍然显示错误的 tagCount，持续 100-500ms |
| **3. 重复 getUser 请求** | 性能 | MainLayout 和 UserProfile 都发送相同的 getUser 请求 |
| **4. 过滤时 tagCount 不更新** | 设计选择 | 这是当前设计，但需要确认是否符合产品预期 |

### 8.2 最严重问题

**问题 1 和问题 2 组合**：
- 当 `alice` 访问 `/u/steven` 时
- 列表显示 `steven` 的笔记（正确）
- TagsSection 显示 `alice` 的笔记的标签（错误）
- 用户点击错误的标签会得到空列表
- **这会直接导致用户困惑和功能不可用**

### 8.3 建议优先级

1. **高优先级**：修复 profile 场景的 fallback 数据范围，使其包含正确的 creator 过滤
2. **中优先级**：消除重复的 getUser 请求
3. **低/中优先级**：确认过滤时 tagCount 是否应该更新的产品设计
