# Memos RSS 订阅与公开 Profile 安全分析报告

## 1. 概述

本报告分析 memos 项目中 RSS 订阅和公开 Profile 功能的实现机制，重点关注：
- **公开访问入口**：哪些端点对外部 reader 开放
- **内容过滤机制**：如何筛选可公开的内容
- **权限裁剪策略**：用户数据如何进行脱敏处理
- **潜在安全边界**：需要注意的安全考量

---

## 2. RSS 订阅机制

### 2.1 路由注册

**文件**: `server/router/rss/rss.go:67-70`

```go
func (s *RSSService) RegisterRoutes(g *echo.Group) {
	g.GET("/explore/rss.xml", s.GetExploreRSS)
	g.GET("/u/:username/rss.xml", s.GetUserRSS)
}
```

**公开访问端点**：
| 端点 | 说明 | 认证要求 |
|------|------|----------|
| `/explore/rss.xml` | 全站公开 memo 的聚合 feed | 无需认证 |
| `/u/:username/rss.xml` | 特定用户的公开 memo feed | 无需认证 |

### 2.2 内容过滤逻辑

#### 2.2.1 Explore RSS (全站公开)

**文件**: `server/router/rss/rss.go:72-108`

```go
func (s *RSSService) GetExploreRSS(c echo.Context) error {
	// ...
	normalStatus := store.Normal
	limit := maxRSSItemCount  // 100
	memoFind := store.FindMemo{
		RowStatus:      &normalStatus,
		VisibilityList: []store.Visibility{store.Public},  // 关键：只取 PUBLIC
		Limit:          &limit,
	}
	memoList, err := s.Store.ListMemos(ctx, &memoFind)
	// ...
}
```

**过滤条件**：
- `RowStatus = Normal` (排除已归档 memo)
- `Visibility = Public` (**严格限制为公开可见**)
- 最多 100 条

#### 2.2.2 User RSS (用户级公开)

**文件**: `server/router/rss/rss.go:110-158`

```go
func (s *RSSService) GetUserRSS(c echo.Context) error {
	// 1. 先验证用户存在
	user, err := s.Store.GetUser(ctx, &store.FindUser{
		Username: &username,
	})
	if user == nil {
		return echo.NewHTTPError(http.StatusNotFound, "User not found")
	}

	// 2. 查询该用户的公开 memo
	normalStatus := store.Normal
	limit := maxRSSItemCount
	memoFind := store.FindMemo{
		CreatorID:      &user.ID,        // 限定用户
		RowStatus:      &normalStatus,
		VisibilityList: []store.Visibility{store.Public},  // 同样只取 PUBLIC
		Limit:          &limit,
	}
	// ...
}
```

**额外验证**：
- 先检查用户是否存在，不存在返回 404
- 通过 `CreatorID` 限定只返回指定用户的 memo

### 2.3 RSS Feed 内容生成

**文件**: `server/router/rss/rss.go:160-298`

```go
func (s *RSSService) generateRSSFromMemoList(...) {
	// ...
	for i := 0; i < itemCountLimit; i++ {
		memo := memoList[i]
		
		// 1. 标题生成：取第一行，去除 Markdown 标题语法
		title := s.generateItemTitle(memo.Content)
		
		// 2. 内容渲染：Markdown → HTML
		htmlContent, err := s.getRSSItemDescription(memo.Content)
		
		// 3. 作者信息
		if creator, ok := creatorMap[memo.CreatorID]; ok {
			authorName := creator.Nickname
			if authorName == "" {
				authorName = creator.Username
			}
			item.Author = &feeds.Author{
				Name:  authorName,
				Email: creator.Email,  // ⚠️ 注意：Email 会暴露
			}
		}
		
		// 4. 附件处理
		if attachments, ok := attachmentsByMemoID[memo.ID]; ok && len(attachments) > 0 {
			// 第一个附件作为 enclosure
			enclosure.Url = // S3/External 直接用 reference，本地文件构建 URL
			item.Enclosure = &enclosure
		}
	}
}
```

### 2.4 缓存机制

**文件**: `server/router/rss/rss.go:34-40, 344-408`

```go
const (
	maxRSSItemCount      = 100
	defaultCacheDuration = 1 * time.Hour  // 缓存 1 小时
	maxCacheSize         = 50              // 最多缓存 50 个 feed
)

type cacheEntry struct {
	content      string
	etag         string
	lastModified time.Time
	createdAt    time.Time
}
```

**缓存特性**：
- **Key 命名**：`explore` 或 `user:{username}`
- **ETag**：基于内容 SHA256 哈希（取前 8 字节）
- **过期策略**：1 小时自动过期
- **淘汰策略**：LRU（超过 50 条时删除最旧的）
- **HTTP 缓存头**：
  - `Cache-Control: public, max-age=3600`
  - `ETag: "hash"`
  - `Last-Modified: ...`
- **条件请求支持**：`If-None-Match` → 304 Not Modified

---

## 3. 公开 API 端点权限 (ACL 配置)

### 3.1 公共方法白名单

**文件**: `server/router/api/v1/acl_config.go:1-47`

```go
var PublicMethods = map[string]struct{}{
	// Auth Service - 登录相关
	"/memos.api.v1.AuthService/SignIn":       {},
	"/memos.api.v1.AuthService/RefreshToken": {},

	// Instance Service - 实例信息
	"/memos.api.v1.InstanceService/GetInstanceProfile": {},
	"/memos.api.v1.InstanceService/GetInstanceSetting": {},

	// User Service - 用户信息（关键）
	"/memos.api.v1.UserService/CreateUser":       {},
	"/memos.api.v1.UserService/GetUser":          {},  // 获取用户信息
	"/memos.api.v1.UserService/BatchGetUsers":    {},
	"/memos.api.v1.UserService/GetUserAvatar":    {},
	"/memos.api.v1.UserService/GetUserStats":     {},  // 用户统计
	"/memos.api.v1.UserService/ListAllUserStats": {},

	// Identity Provider Service
	"/memos.api.v1.IdentityProviderService/ListIdentityProviders": {},

	// Memo Service - 核心内容访问（关键）
	"/memos.api.v1.MemoService/GetMemo":              {},
	"/memos.api.v1.MemoService/ListMemos":            {},  // 列表
	"/memos.api.v1.MemoService/ListMemoComments":     {},  // 评论
	"/memos.api.v1.MemoService/GetLinkMetadata":      {},
	"/memos.api.v1.MemoService/BatchGetLinkMetadata": {},

	// Share Token
	"/memos.api.v1.MemoService/GetMemoByShare": {},
}
```

**关键结论**：
- `ListMemos`、`GetMemo`、`ListMemoComments` 都是公开的
- `GetUser`、`GetUserStats` 也是公开的
- **但这只是 API 层的准入，实际权限由 Service 层进一步过滤**

---

## 4. Memo 可见性过滤逻辑 (Service 层)

### 4.1 Visibility 枚举定义

**文件**: `proto/api/v1/memo_service.proto:142-147`

```protobuf
enum Visibility {
  VISIBILITY_UNSPECIFIED = 0;
  PRIVATE = 1;    // 私有：仅创作者可见
  PROTECTED = 2;  // 受保护：仅登录用户可见
  PUBLIC = 3;     // 公开：所有人可见（包括未登录）
}
```

**Store 层定义**: `store/memo.go:12-33`

```go
type Visibility string

const (
	Public    Visibility = "PUBLIC"
	Protected Visibility = "PROTECTED"
	Private   Visibility = "PRIVATE"
)
```

### 4.2 ListMemos 过滤逻辑

**文件**: `server/router/api/v1/memo_service.go:157-321`

```go
func (s *APIV1Service) ListMemos(ctx context.Context, request *v1pb.ListMemosRequest) (*v1pb.ListMemosResponse, error) {
	// ...
	currentUser, err := s.fetchCurrentUser(ctx)  // 获取当前用户（可能为 nil）

	// 归档状态的特殊处理
	if request.State == v1pb.State_ARCHIVED {
		// Archived memos are only visible to their creator.
		if currentUser == nil {
			return &v1pb.ListMemosResponse{}, nil  // 未登录用户看不到归档
		}
		memoFind.CreatorID = &currentUser.ID  // 只能看自己的归档
	}

	// ⚠️ 核心可见性过滤逻辑
	if currentUser == nil {
		// 未登录用户：只能看 PUBLIC
		memoFind.VisibilityList = []store.Visibility{store.Public}
	} else {
		if memoFind.CreatorID == nil {
			// 没有指定创作者：看自己的 + 他人的 PUBLIC/PROTECTED
			filter := fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
			memoFind.Filters = append(memoFind.Filters, filter)
		} else if *memoFind.CreatorID != currentUser.ID {
			// 指定了创作者，但不是自己：只能看 PUBLIC/PROTECTED
			memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
		}
		// else: 看自己的，无 visibility 限制（能看到 PRIVATE）
	}
	// ...
}
```

**过滤矩阵**：

| 用户身份 | 查询场景 | 可见的 Visibility |
|----------|----------|-------------------|
| 未登录 (visitor) | 任意 | **仅 PUBLIC** |
| 已登录 (user) | 不指定 creator | 自己的所有 + 他人的 PUBLIC/PROTECTED |
| 已登录 (user) | 指定 creator=他人 | **PUBLIC + PROTECTED** |
| 已登录 (user) | 指定 creator=自己 | 所有 (PUBLIC/PROTECTED/PRIVATE) |

### 4.3 GetMemo 单条查询过滤

**文件**: `server/router/api/v1/memo_service.go:323-388`

```go
func (s *APIV1Service) GetMemo(ctx context.Context, request *v1pb.GetMemoRequest) (*v1pb.Memo, error) {
	// 1. 先获取 memo
	memo, err := s.Store.GetMemo(ctx, &store.FindMemo{UID: &memoUID})
	
	// 2. 归档检查
	if memo.RowStatus == store.Archived {
		user, _ := s.fetchCurrentUser(ctx)
		if user == nil || memo.CreatorID != user.ID {
			return nil, status.Errorf(codes.NotFound, "memo not found")  // 伪装成不存在
		}
	}

	// 3. 可见性检查
	if memo.Visibility != store.Public {
		user, _ := s.fetchCurrentUser(ctx)
		if user == nil {
			return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
		}
		if memo.Visibility == store.Private && memo.CreatorID != user.ID {
			return nil, status.Errorf(codes.PermissionDenied, "permission denied")
		}
		// PROTECTED：登录用户即可
	}
	// ...
}
```

**单条访问决策树**：

```
GetMemo(uid)
    │
    ├── memo 不存在 → NotFound
    │
    ├── memo.RowStatus == Archived
    │       ├── 未登录 → NotFound (隐藏存在性)
    │       └── 非创作者 → NotFound (隐藏存在性)
    │
    └── memo.Visibility 检查
            ├── PUBLIC → ✓ 允许访问
            │
            ├── PROTECTED
            │       ├── 已登录 → ✓ 允许
            │       └── 未登录 → Unauthenticated
            │
            └── PRIVATE
                    ├── 是创作者 → ✓ 允许
                    └── 非创作者 → PermissionDenied
```

### 4.4 ListMemoComments 评论过滤

**文件**: `server/router/api/v1/memo_service.go:716-862`

```go
func (s *APIV1Service) ListMemoComments(ctx context.Context, request *v1pb.ListMemoCommentsRequest) (*v1pb.ListMemoCommentsResponse, error) {
	// ...
	currentUser, _ := s.fetchCurrentUser(ctx)
	
	var memoFilter string
	if currentUser == nil {
		memoFilter = `visibility == "PUBLIC"`  // 未登录：只看公开评论
	} else {
		// 已登录：自己的评论 + 公开/受保护的评论
		memoFilter = fmt.Sprintf(`creator_id == %d || visibility in ["PUBLIC", "PROTECTED"]`, currentUser.ID)
	}
	
	// 传递给 MemoRelation 查询
	memoRelations, err := s.Store.ListMemoRelations(ctx, &store.FindMemoRelation{
		RelatedMemoID: &memo.ID,
		Type:          &memoRelationComment,
		MemoFilter:    &memoFilter,  // 应用过滤
		// ...
	})
	// ...
}
```

---

## 5. 用户数据权限裁剪

### 5.1 敏感字段过滤

**文件**: `server/router/api/v1/memo_service.go:1213-1246`

```go
func convertUserFromStore(user *store.User, viewer *store.User) *v1pb.User {
	userpb := &v1pb.User{
		Name:        BuildUserName(user.Username),
		State:       convertStateFromStore(user.RowStatus),
		CreateTime:  timestamppb.New(time.Unix(user.CreatedTs, 0)),
		UpdateTime:  timestamppb.New(time.Unix(user.UpdatedTs, 0)),
		Role:        convertUserRoleFromStore(user.Role),
		Username:    user.Username,
		DisplayName: user.Nickname,
		AvatarUrl:   user.AvatarURL,
		Description: user.Description,
	}
	
	// ⚠️ Email 字段的权限控制
	if canViewerAccessUserEmail(viewer, user) {
		userpb.Email = user.Email
	}
	// ...
}

func canViewerAccessUserEmail(viewer, user *store.User) bool {
	if viewer == nil || user == nil {
		return false
	}
	// 只有管理员或用户本人能看到 Email
	return viewer.Role == store.RoleAdmin || viewer.ID == user.ID
}
```

**用户字段暴露策略**：

| 字段 | 未登录用户 | 已登录普通用户(他人) | 已登录用户(自己) | 管理员 |
|------|-----------|---------------------|-----------------|--------|
| name | ✓ | ✓ | ✓ | ✓ |
| username | ✓ | ✓ | ✓ | ✓ |
| displayName (nickname) | ✓ | ✓ | ✓ | ✓ |
| avatarUrl | ✓ | ✓ | ✓ | ✓ |
| description | ✓ | ✓ | ✓ | ✓ |
| role | ✓ | ✓ | ✓ | ✓ |
| state | ✓ | ✓ | ✓ | ✓ |
| **email** | ✗ | ✗ | ✓ | ✓ |
| createTime | ✓ | ✓ | ✓ | ✓ |
| updateTime | ✓ | ✓ | ✓ | ✓ |

**关键结论**：
- **Email 是唯一被裁剪的敏感字段**
- 其他用户信息（用户名、昵称、头像、描述、角色）**全部公开**

### 5.2 RSS 中的 Email 暴露问题

**回顾**：`server/router/rss/rss.go:261-271`

```go
if creator, ok := creatorMap[memo.CreatorID]; ok {
	authorName := creator.Nickname
	if authorName == "" {
		authorName = creator.Username
	}
	item.Author = &feeds.Author{
		Name:  authorName,
		Email: creator.Email,  // ⚠️ RSS 中直接使用了 Email！
	}
}
```

**问题分析**：
1. RSS feed 生成时，`creator.Email` 被直接放入 `<author>` 元素
2. 这意味着**即使是未登录用户访问 RSS，也能看到 memo 创作者的 Email**
3. 这与 API 层 `convertUserFromStore` 的策略不一致

**RSS 2.0 规范中 author 元素**：
```xml
<author>email@example.com (Author Name)</author>
```

---

## 6. 前端公开页面实现

### 6.1 Explore 页面 (全站公开)

**文件**: `web/src/pages/Explore.tsx:1-41`

```tsx
const Explore = () => {
  const currentUser = useCurrentUser();

  // 前端也做了可见性限制（作为后端的补充）
  const visibilities = currentUser 
    ? [Visibility.PUBLIC, Visibility.PROTECTED]  // 已登录：PUBLIC + PROTECTED
    : [Visibility.PUBLIC];                          // 未登录：仅 PUBLIC

  const memoFilter = useMemoFilters({
    includeShortcuts: false,
    includePinned: false,
    visibilities,  // 传递可见性过滤器
  });

  // ...
  return (
    <PagedMemoList
      renderer={(memo: Memo) => (
        <MemoView ... showCreator showVisibility compact />
      )}
      filter={memoFilter}
      showCreator
    />
  );
};
```

### 6.2 UserProfile 页面 (用户主页)

**文件**: `web/src/pages/UserProfile.tsx:1-156`

```tsx
const UserProfile = () => {
  const username = useParams().username;
  const { data: user } = useUser(`users/${username}`, { enabled: !!username });

  // 注意：这里没有显式传入 visibilities
  // 依赖后端根据 creatorID + 当前用户身份自动过滤
  const memoFilter = useMemoFilters({
    creatorName: user?.name,  // 指定创作者
    includeShortcuts: false,
    includePinned: true,
  });

  // ...
  return (
    <ProfileHeader user={user} ... />
    <PagedMemoList filter={memoFilter} ... />
  );
};
```

### 6.3 useMemoFilters Hook 实现

**文件**: `web/src/hooks/useMemoFilters.ts:1-96`

```tsx
export const useMemoFilters = (options: UseMemoFiltersOptions = {}): string | undefined => {
  const { creatorName, includeShortcuts = false, includePinned = false, visibilities } = options;
  // ...
  
  const conditions: string[] = [];

  // 1. 创作者过滤
  if (creatorName) {
    const creatorFilter = buildMemoCreatorFilter(creatorName);
    if (creatorFilter) {
      conditions.push(creatorFilter);
    }
  }

  // 2. 可见性过滤（如果显式传入）
  if (visibilities && visibilities.length > 0) {
    const visibilityValues = visibilities.map((v) => `"${getVisibilityName(v)}"`).join(", ");
    conditions.push(`visibility in [${visibilityValues}]`);
  }

  return conditions.length > 0 ? conditions.join(" && ") : undefined;
};
```

**前端过滤策略总结**：
- **Explore 页面**：显式传入 `visibilities`，前端主动限制
- **UserProfile 页面**：不传 `visibilities`，完全依赖后端根据 `creatorID` + 当前用户身份自动过滤
- **后端兜底**：无论前端传什么，`ListMemos` Service 层都会再次应用可见性过滤

---

## 7. 数据流与安全边界总览

### 7.1 公开访问入口架构

```
                    ┌─────────────────────────────────────────┐
                    │           外部 Reader (未登录)            │
                    └─────────────────┬───────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  RSS Endpoints  │    │   API (gRPC)    │    │   Frontend SPA  │
    │  /explore/rss   │    │  /api/v1/*      │    │  /explore       │
    │  /u/:user/rss   │    │                 │    │  /u/:username   │
    └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
             │                      │                      │
             │                      │                      │
             ▼                      ▼                      ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                    ACL 白名单 (acl_config.go)                    │
    │  PublicMethods = {                                               │
    │    "/memos.api.v1.MemoService/ListMemos",                       │
    │    "/memos.api.v1.MemoService/GetMemo",                         │
    │    "/memos.api.v1.UserService/GetUser",                         │
    │    ...                                                            │
    │  }                                                                │
    └────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                   Service 层权限过滤 (第二道防线)                  │
    │                                                                   │
    │  ListMemos:                                                      │
    │    ┌─────────────────────────────────────────────────┐          │
    │    │  currentUser == nil                             │          │
    │    │    → VisibilityList = [PUBLIC]                 │          │
    │    │                                                 │          │
    │    │  currentUser != nil && creatorID != self       │          │
    │    │    → VisibilityList = [PUBLIC, PROTECTED]     │          │
    │    │                                                 │          │
    │    │  creatorID == self                              │          │
    │    │    → 无限制 (可见 PRIVATE)                      │          │
    │    └─────────────────────────────────────────────────┘          │
    │                                                                   │
    │  convertUserFromStore:                                           │
    │    ┌─────────────────────────────────────────────────┐          │
    │    │  Email 字段:                                     │          │
    │    │    viewer == nil → 不暴露                       │          │
    │    │    viewer != self && !admin → 不暴露            │          │
    │    │    viewer == self || admin → 暴露               │          │
    │    └─────────────────────────────────────────────────┘          │
    └────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                        Store 层 (数据库查询)                       │
    │  FindMemo {                                                      │
    │    VisibilityList: [...]   // 由 Service 层传入                  │
    │    CreatorID: &id         // 可选                                │
    │    Filters: [...]         // CEL 表达式                         │
    │  }                                                                │
    └─────────────────────────────────────────────────────────────────┘
```

### 7.2 关键安全决策点

| 层级 | 组件 | 职责 | 关键判断 |
|------|------|------|----------|
| L1 | Echo 路由 | 路由分发 | RSS 端点无 auth 中间件 |
| L2 | ACL 拦截器 | API 准入 | 检查 `PublicMethods` 白名单 |
| L3 | Service 层 | 业务权限 | 根据 `currentUser` + `creatorID` + `visibility` 过滤 |
| L4 | Store 层 | 数据查询 | 执行实际的 SQL/Like 过滤 |

---

## 8. 发现的问题与风险

### 8.1 RSS Feed 中 Email 暴露 (高关注)

**位置**: `server/router/rss/rss.go:268-270`

**代码**:
```go
item.Author = &feeds.Author{
	Name:  authorName,
	Email: creator.Email,  // ⚠️ 未检查访问者身份
}
```

**问题描述**:
1. RSS 是公开访问的（`/explore/rss.xml`、`/u/:username/rss.xml`）
2. 生成 RSS feed 时，创作者的 Email 被直接放入 `<author>` 元素
3. 这与 `convertUserFromStore` 中对 Email 的权限控制不一致

**影响**:
- 任何能访问 RSS 的外部 reader 都能获取发布公开 memo 用户的 Email
- Email 可能被用于 spam、钓鱼等攻击

**建议修复**:
方案 A：完全移除 RSS 中的 Email（推荐，因为 RSS reader 通常不需要）
```go
item.Author = &feeds.Author{
	Name: authorName,
	// Email: creator.Email,  // 移除或仅在特定条件下暴露
}
```

方案 B：添加与 API 层一致的权限检查（但 RSS 场景下 viewer 通常为 nil）

### 8.2 用户信息公开范围 (中等关注)

**当前行为**:
- 用户名 (`username`)、昵称 (`displayName`)、头像 (`avatarUrl`)、描述 (`description`)、角色 (`role`) 全部公开
- 只有 `email` 被保护

**风险点**:
- 如果用户使用真实姓名作为 username/nickname，这些信息会完全公开
- `description` 可能包含敏感信息
- 攻击者可以通过 `ListAllUserStats` 枚举所有用户

**建议**:
- 考虑添加实例级设置控制用户信息的公开程度
- 或者在用户设置中添加"profile 可见性"选项

### 8.3 归档备忘录的存在性隐藏 (设计决策)

**位置**: `server/router/api/v1/memo_service.go:338-346`

```go
if memo.RowStatus == store.Archived {
	user, _ := s.fetchCurrentUser(ctx)
	if user == nil || memo.CreatorID != user.ID {
		return nil, status.Errorf(codes.NotFound, "memo not found")
	}
}
```

**分析**:
- 这是一个**安全设计**：非创作者访问已归档 memo 时返回 404，而非 403
- 优点：隐藏了 memo 的存在性（攻击者无法判断某个 UID 是否真的存在）
- 注意：这只影响 `GetMemo`，`ListMemos` 对归档的处理是直接过滤掉

---

## 9. 测试验证建议

### 9.1 权限矩阵测试用例

| 测试场景 | 预期结果 | 实际结果 |
|----------|----------|----------|
| 未登录访问 `/explore/rss.xml` | 只包含 PUBLIC memo | ⚠️ 但包含创作者 Email |
| 未登录调用 `ListMemos` | 只返回 PUBLIC memo | ✓ |
| 未登录调用 `GetMemo` (PUBLIC) | 返回 memo | ✓ |
| 未登录调用 `GetMemo` (PROTECTED) | Unauthenticated | ✓ |
| 未登录调用 `GetMemo` (PRIVATE) | PermissionDenied | ✓ |
| 已登录用户 A 调用 `GetMemo` (用户 B 的 PROTECTED) | 返回 memo | ✓ |
| 已登录用户 A 调用 `GetMemo` (用户 B 的 PRIVATE) | PermissionDenied | ✓ |
| 未登录调用 `GetUser` | 返回用户信息，但 email 为空 | ✓ |

### 9.2 关键测试代码位置

- `store/test/memo_test.go` - memo 基础测试
- `store/test/memo_filter_test.go` - 过滤器测试

---

## 10. 总结

### 10.1 安全设计优点

1. **多层防御**：ACL 白名单 + Service 层过滤 + Store 层查询，三重保障
2. **可见性分级**：PUBLIC / PROTECTED / PRIVATE 三级权限，设计合理
3. **敏感字段保护**：Email 字段有明确的权限控制
4. **存在性隐藏**：归档 memo 对非创作者返回 404，避免信息泄露

### 10.2 需要关注的问题

1. **RSS Email 泄露**：`server/router/rss/rss.go` 中 `generateRSSFromMemoList` 函数直接暴露创作者 Email
2. **用户信息公开范围大**：除 Email 外的用户信息全部公开

### 10.3 修复建议优先级

| 优先级 | 问题 | 建议措施 |
|--------|------|----------|
| P0 | RSS Email 泄露 | 移除或条件化 RSS author 中的 Email 字段 |
| P2 | 用户信息公开范围 | 考虑添加实例/用户级的 profile 可见性控制 |

---

## 附录 A: 关键文件索引

| 文件路径 | 说明 |
|----------|------|
| `server/router/rss/rss.go` | RSS 路由和 feed 生成 |
| `server/router/api/v1/acl_config.go` | 公开 API 白名单 |
| `server/router/api/v1/memo_service.go` | Memo 可见性过滤逻辑 |
| `server/router/api/v1/user_service.go` | 用户数据转换和权限裁剪 |
| `store/memo.go` | Visibility 枚举和 Memo 模型 |
| `web/src/pages/Explore.tsx` | 前端 Explore 页面 |
| `web/src/pages/UserProfile.tsx` | 前端用户 Profile 页面 |
| `web/src/hooks/useMemoFilters.ts` | 前端过滤器 Hook |

---

**报告生成时间**: 2026-05-05  
**分析代码版本**: 当前工作目录
