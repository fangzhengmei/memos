# Memos Inbox Activity 数据流分析报告

## 1. 概述

本报告详细分析了 Memos 项目中 **Inbox 消息**、**用户活动记录**和**通知入口**的完整数据流串联关系。从后端事件生成到前端通知展示，涵盖了所有关键组件和交互路径。

### 核心概念区分

| 概念 | 定义 | 存储位置 | 触发场景 |
|------|------|----------|----------|
| **Inbox 消息** | 用户收到的通知（评论、提及） | `inbox` 表 | 评论创建、用户被提及 |
| **Activity 历史（旧版）** | 通用活动日志记录（已废弃） | `activity` 表（已删除） | 旧版本所有用户活动 |
| **活动统计** | 用户备忘录创建/更新的时间统计 | 从 `memo` 表动态聚合 | 备忘录创建/更新时 |
| **SSE 实时事件** | 服务器推送到客户端的实时更新 | 内存 (SSEHub) | 备忘录增删改、评论、反应 |
| **Webhook** | 推送到外部系统的事件通知 | 无持久化 | 备忘录增删改、评论创建 |
| **邮件通知** | 发送给用户的邮件提醒 | 无持久化 | Inbox 消息创建（可选） |

---

## 2. 旧版本 Activity 历史表与版本迁移

### 2.1 Activity 表在旧版本中的角色

在 Memos 的早期版本中，`activity` 表是一个通用的活动日志表，用于记录所有用户活动。

**表结构定义** (`store/migration/sqlite/0.10/00__activity.sql`):

```sql
CREATE TABLE activity (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  creator_id INTEGER NOT NULL,           -- 活动创建者
  created_ts BIGINT NOT NULL DEFAULT (strftime('%s', 'now')),
  type TEXT NOT NULL DEFAULT '',          -- 活动类型
  level TEXT NOT NULL CHECK (level IN ('INFO', 'WARN', 'ERROR')) DEFAULT 'INFO',
  payload TEXT NOT NULL DEFAULT '{}'      -- JSON 格式的活动载荷
);
```

**Activity 表的职责**：
1. 记录所有用户活动（备忘录创建、评论、提及等）
2. 作为 Inbox 消息的"数据来源"——Inbox 表通过 `activityId` 引用 activity 表
3. 提供活动历史查询能力

### 2.2 版本迁移历史

Activity 表经历了三个阶段的迁移：

#### 阶段一：0.17 版本 - 清空 Activity 数据

**迁移脚本** (`store/migration/sqlite/0.17/01__delete_activities.sql`):

```sql
DELETE FROM activity;
```

**背景**：在 0.17 版本引入 `inbox` 表后，系统开始逐步淘汰 `activity` 表。此迁移清空了历史数据，为后续迁移做准备。

#### 阶段二：0.27 版本 - 迁移 Inbox 载荷

**迁移脚本** (`store/migration/sqlite/0.27/02__migrate_inbox_message_payload.sql`):

```sql
UPDATE inbox
SET message = json_set(
  json_remove(message, '$.activityId'),  -- 移除 activityId 引用
  '$.memoComment',
  json_object(
    'memoId',
    (
      -- 从 activity 表提取 memoId
      SELECT json_extract(activity.payload, '$.memoComment.memoId')
      FROM activity
      WHERE activity.id = json_extract(inbox.message, '$.activityId')
    ),
    'relatedMemoId',
    (
      -- 从 activity 表提取 relatedMemoId
      SELECT json_extract(activity.payload, '$.memoComment.relatedMemoId')
      FROM activity
      WHERE activity.id = json_extract(inbox.message, '$.activityId')
    )
  )
)
WHERE json_extract(message, '$.activityId') IS NOT NULL
  AND EXISTS (
    SELECT 1
    FROM activity
    WHERE activity.id = json_extract(inbox.message, '$.activityId')
  );
```

**迁移逻辑说明**：

```
迁移前的 Inbox.message 结构：
{
  "activityId": 123  // 引用 activity 表的 ID
}

迁移后的 Inbox.message 结构：
{
  "memoComment": {          // 直接内嵌载荷
    "memoId": 456,
    "relatedMemoId": 789
  }
}
```

#### 阶段三：0.27 版本 - 删除 Activity 表

**迁移脚本** (`store/migration/sqlite/0.27/03__drop_activity.sql`):

```sql
DROP TABLE activity;
```

### 2.3 迁移时间线总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Activity 表迁移时间线                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  0.10 版本                    0.17 版本                    0.27 版本        │
│  ──────────                   ──────────                   ──────────        │
│                                                                              │
│  ┌─────────────┐              ┌─────────────┐              ┌─────────────┐  │
│  │ 创建 activity│              │ 清空 activity│              │ 迁移 inbox   │  │
│  │ 表          │              │ 表数据       │              │ 载荷         │  │
│  └──────┬──────┘              └──────┬──────┘              └──────┬──────┘  │
│         │                            │                            │          │
│         ▼                            ▼                            ▼          │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                    Inbox 表数据结构演进                                   ││
│  ├─────────────────────────────────────────────────────────────────────────┤│
│  │                                                                           ││
│  │  0.10 ~ 0.17:                                                           ││
│  │  { "activityId": 123 }  ──────▶  引用 activity 表                     ││
│  │                                                                           ││
│  │  0.17 ~ 0.27:                                                           ││
│  │  数据已清空，但结构保持不变                                              ││
│  │                                                                           ││
│  │  0.27 之后:                                                              ││
│  │  { "memoComment": { "memoId": 456, "relatedMemoId": 789 } }          ││
│  │  或 { "memoMention": { "memoId": 456, "relatedMemoId": 789 } }       ││
│  │  ──────▶  直接内嵌载荷，不再依赖 activity 表                             ││
│  │                                                                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  最终状态：activity 表被彻底删除，Inbox 表自包含所有必要数据               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 当前版本活动统计数据链路

当前版本不再使用独立的 `activity` 表存储活动历史。取而代之的是：

1. **通知类活动**：通过 `inbox` 表存储（评论、提及）
2. **统计类活动**：从 `memo` 表动态聚合计算

### 3.1 后端统计数据生成

**代码路径**: `server/router/api/v1/user_service_stats.go`

#### GetUserStats - 单用户统计

```go
func (s *APIV1Service) GetUserStats(ctx context.Context, request *v1pb.GetUserStatsRequest) (*v1pb.UserStats, error) {
    // 1. 解析目标用户
    user, err := ResolveUserByName(ctx, s.Store, request.Name)
    
    // 2. 构建备忘录查询条件
    memoFind := &store.FindMemo{
        CreatorID:       &userID,
        ExcludeComments: true,    // 排除评论
        ExcludeContent:  true,    // 不需要内容
        RowStatus:       &normalStatus,
    }
    
    // 3. 权限过滤（根据当前用户身份决定可见性）
    if currentUser == nil {
        memoFind.VisibilityList = []store.Visibility{store.Public}
    } else if currentUser.ID != userID {
        memoFind.VisibilityList = []store.Visibility{store.Public, store.Protected}
    }
    
    // 4. 分批查询并聚合统计
    createdTimestamps := []*timestamppb.Timestamp{}
    updatedTimestamps := []*timestamppb.Timestamp{}
    tagCount := make(map[string]int32)
    // ... 其他统计字段
    
    for {
        memos, err := s.Store.ListMemos(ctx, memoFind)
        if len(memos) == 0 {
            break
        }
        
        for _, memo := range memos {
            // 聚合创建时间戳（用于热力图）
            createdTimestamps = append(createdTimestamps, 
                timestamppb.New(time.Unix(memo.CreatedTs, 0)))
            updatedTimestamps = append(updatedTimestamps, 
                timestamppb.New(time.Unix(memo.UpdatedTs, 0)))
            
            // 统计标签数量
            if memo.Payload != nil {
                for _, tag := range memo.Payload.Tags {
                    tagCount[tag]++
                }
                // 统计备忘录类型（链接、代码、待办等）
                if memo.Payload.Property != nil {
                    if memo.Payload.Property.HasLink {
                        linkCount++
                    }
                    // ...
                }
            }
        }
        offset += limit
    }
    
    // 5. 构建返回结果
    return &v1pb.UserStats{
        Name:                  fmt.Sprintf("%s/stats", BuildUserName(user.Username)),
        MemoCreatedTimestamps: createdTimestamps,  // 关键：创建时间戳数组
        MemoUpdatedTimestamps: updatedTimestamps,  // 关键：更新时间戳数组
        TagCount:              tagCount,
        TotalMemoCount:        totalMemoCount,
        MemoTypeStats: &v1pb.UserStats_MemoTypeStats{
            LinkCount: linkCount,
            CodeCount: codeCount,
            TodoCount: todoCount,
            UndoCount: undoCount,
        },
        PinnedMemos: pinnedMemos,
    }, nil
}
```

#### ListAllUserStats - 全站统计

与 `GetUserStats` 类似，但遍历所有用户的可见备忘录，用于 Explore 页面的统计展示。

### 3.2 前端统计数据处理

**代码路径**: `web/src/hooks/useFilteredMemoStats.ts`

```typescript
export const useFilteredMemoStats = (options: UseFilteredMemoStatsOptions = {}): FilteredMemoStats => {
  const { userName, context } = options;
  const currentUser = useCurrentUser();
  const { timeBasis } = useView();  // "create_time" 或 "update_time"

  // 1. 获取后端统计数据（用于 Home/Profile 页面）
  const { data: userStats, isLoading: isLoadingUserStats } = useUserStats(userName);

  // 2. 获取备忘录列表（用于 Explore 页面或 fallback）
  const { data: memosResponse, isLoading: isLoadingMemos } = useMemos(memoQueryParams);

  // 3. 聚合计算每日活动统计
  const data = useMemo(() => {
    let activityStats: Record<string, number> = {};  // key: "YYYY-MM-DD", value: count
    let tagCount: Record<string, number> = {};

    if (context === "explore") {
      // Explore 页面：从可见备忘录列表计算
      const displayDates = (memosResponse?.memos ?? [])
        .map((memo) => memoTimestampForBasis(memo, timeBasis))
        .filter((date): date is Date => date !== undefined)
        .map(toDateString);  // 转为 "YYYY-MM-DD"
      activityStats = countBy(displayDates);  // 按日期统计数量
    } else if (userName && userStats) {
      // Home/Profile 页面：使用后端返回的时间戳数组
      const sourceArray = wantUpdated ? updatedArray : createdArray;
      if (sourceArray.length > 0) {
        activityStats = countBy(
          sourceArray
            .map((ts) => (ts ? timestampDate(ts) : undefined))
            .filter((date): date is Date => date !== undefined)
            .map(toDateString),
        );
      }
      tagCount = userStats.tagCount;
    }
    // ... fallback 逻辑

    return { statistics: { activityStats, timeBasis }, tags: tagCount, loading };
  }, [context, userName, userStats, memosResponse, ...]);

  return data;
};
```

### 3.3 前端热力图展示

**代码路径**: `web/src/components/StatisticsView/StatisticsView.tsx`

```typescript
const StatisticsView = (props: Props) => {
  const { statisticsData } = props;
  const { activityStats, timeBasis } = statisticsData;
  // activityStats = { "2026-05-01": 5, "2026-05-02": 3, ... }

  return (
    <div className="group w-full mt-2 flex flex-col text-muted-foreground animate-fade-in">
      {/* 月份导航器 */}
      <MonthNavigator
        visibleMonth={visibleMonthString}
        onMonthChange={setVisibleMonthString}
        activityStats={activityStats}
        timeBasis={timeBasis}
      />

      {/* 月历热力图 */}
      <MonthCalendar
        month={visibleMonthString}
        data={activityStats}
        maxCount={calculateMaxCount(activityStats)}  // 用于颜色强度计算
        onClick={navigateToDateFilter}
        timeBasis={timeBasis}
      />
    </div>
  );
};
```

**CalendarCell 颜色强度逻辑** (`web/src/components/ActivityCalendar/utils.ts`):

```typescript
export const getCellIntensityClass = (day: CalendarDayCell, maxCount: number) => {
  if (day.count === 0 || maxCount === 0) {
    return "";  // 无活动：默认背景
  }
  
  // 根据当天活动数量与最大值的比例，确定颜色强度
  const ratio = day.count / maxCount;
  
  if (ratio >= 1) return "bg-primary/60";    // 最高强度
  if (ratio >= 0.75) return "bg-primary/45";
  if (ratio >= 0.5) return "bg-primary/30";
  if (ratio >= 0.25) return "bg-primary/20";
  return "bg-primary/10";  // 最低强度
};
```

### 3.4 活动统计完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           活动统计数据完整数据流                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                              后端数据层                                        │  │
│  ├──────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                              │  │
│  │   ┌──────────┐         ┌─────────────────────────────────────────┐         │  │
│  │   │  memo 表 │────────▶│  GetUserStats / ListAllUserStats       │         │  │
│  │   │          │         │  (server/router/api/v1/user_service_stats.go)    │  │
│  │   └──────────┘         └─────────────────────┬───────────────────┘         │  │
│  │                                                │                              │  │
│  │                                                ▼                              │  │
│  │                                    ┌───────────────────────┐                 │  │
│  │                                    │  UserStats Response   │                 │  │
│  │                                    │  ┌─────────────────┐  │                 │  │
│  │                                    │  │ memoCreated     │  │                 │  │
│  │                                    │  │ Timestamps[]    │──┼──────▶ 热力图   │  │
│  │                                    │  ├─────────────────┤  │                 │  │
│  │                                    │  │ memoUpdated     │  │                 │  │
│  │                                    │  │ Timestamps[]    │  │                 │  │
│  │                                    │  ├─────────────────┤  │                 │  │
│  │                                    │  │ tagCount        │──┼──────▶ 标签云   │  │
│  │                                    │  │ memoTypeStats   │  │                 │  │
│  │                                    │  └─────────────────┘  │                 │  │
│  │                                    └───────────────────────┘                 │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                              │                                       │
│                                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                              前端数据层                                        │  │
│  ├──────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                              │  │
│  │   ┌──────────────────────────────────────────────────────────────────────┐  │  │
│  │   │  useUserStats (web/src/hooks/useUserQueries.ts)                    │  │  │
│  │   │  └─▶ 调用 userServiceClient.getUserStats()                           │  │  │
│  │   └──────────────────────────────────────────────────────────────────────┘  │  │
│  │                                              │                               │  │
│  │                                              ▼                               │  │
│  │   ┌──────────────────────────────────────────────────────────────────────┐  │  │
│  │   │  useFilteredMemoStats (web/src/hooks/useFilteredMemoStats.ts)      │  │  │
│  │   │  ├─▶ 选择数据源（后端统计 或 备忘录列表）                             │  │  │
│  │   │  ├─▶ 按日期聚合：countBy(["2026-05-01", "2026-05-01", ...])       │  │  │
│  │   │  └─▶ 产出：{ "2026-05-01": 2, "2026-05-02": 3, ... }             │  │  │
│  │   └──────────────────────────────────────────────────────────────────────┘  │  │
│  │                                              │                               │  │
│  │                                              ▼                               │  │
│  │   ┌──────────────────────────────────────────────────────────────────────┐  │  │
│  │   │  StatisticsView + ActivityCalendar                                    │  │  │
│  │   │  ├─▶ MonthNavigator：月份切换、统计汇总                               │  │  │
│  │   │  ├─▶ MonthCalendar：月视图网格                                        │  │  │
│  │   │  └─▶ CalendarCell：根据 count/maxCount 计算颜色强度                  │  │  │
│  │   └──────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. ListUserNotifications 完整链路

### 4.1 后端处理流程

**代码路径**: `server/router/api/v1/user_service.go:1528-1595`

```go
func (s *APIV1Service) ListUserNotifications(ctx context.Context, request *v1pb.ListUserNotificationsRequest) (*v1pb.ListUserNotificationsResponse, error) {
    // ============================================
    // 阶段 1：权限验证
    // ============================================
    user, err := s.resolveUserFromName(ctx, request.Parent)  // 从 "users/xxx" 解析用户
    userID := user.ID

    currentUser, err := s.fetchCurrentUser(ctx)
    if currentUser.ID != userID {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }

    // ============================================
    // 阶段 2：从存储层获取 Inbox 数据
    // ============================================
    inboxes, err := s.Store.ListInboxes(ctx, &store.FindInbox{
        ReceiverID: &userID,  // 只查询当前用户的收件箱
    })

    // ============================================
    // 阶段 3：批量获取关联数据（优化 N+1 查询）
    // ============================================
    // 收集所有需要查询的用户 ID
    userIDs := make([]int32, 0, len(inboxes)*2)
    for _, inbox := range inboxes {
        userIDs = append(userIDs, inbox.ReceiverID, inbox.SenderID)
    }
    usersByID, err := s.listUsersByID(ctx, userIDs)  // 批量查询用户

    // 收集所有需要查询的备忘录 ID
    memosByID, err := s.listMemosByID(ctx, collectInboxMemoIDs(inboxes))  // 批量查询备忘录

    // ============================================
    // 阶段 4：转换为 API 层结构
    // ============================================
    notifications := []*v1pb.UserNotification{}
    for _, inbox := range inboxes {
        notification, err := s.convertInboxToUserNotificationWithUsersAndMemos(
            inbox, currentUser, usersByID, memosByID)
        if err != nil {
            if status.Code(err) == codes.NotFound {
                // 跳过引用已删除数据的通知
                slog.Warn("Skipping notification with missing user", ...)
                continue
            }
            return nil, status.Errorf(codes.Internal, "failed to convert inbox: %v", err)
        }
        if notification.Type == v1pb.UserNotification_TYPE_UNSPECIFIED {
            continue  // 跳过未知类型
        }
        notifications = append(notifications, notification)
    }

    // ============================================
    // 阶段 5：返回结果
    // ============================================
    return &v1pb.ListUserNotificationsResponse{
        Notifications: notifications,
    }, nil
}
```

### 4.2 Inbox 到 UserNotification 的转换

**代码路径**: `server/router/api/v1/user_service.go:1753-1811`

```go
func (s *APIV1Service) convertInboxToUserNotificationWithUsersAndMemos(
    inbox *store.Inbox, 
    viewer *store.User, 
    usersByID map[int32]*store.User, 
    memosByID map[int32]*store.Memo,
) (*v1pb.UserNotification, error) {
    // ============================================
    // 1. 基础字段映射
    // ============================================
    receiver := usersByID[inbox.ReceiverID]
    sender := usersByID[inbox.SenderID]

    notification := &v1pb.UserNotification{
        Name:       fmt.Sprintf("%s/notifications/%d", BuildUserName(receiver.Username), inbox.ID),
        Sender:     BuildUserName(sender.Username),
        SenderUser: convertUserFromStore(sender, viewer),  // 完整的用户信息
        CreateTime: timestamppb.New(time.Unix(inbox.CreatedTs, 0)),
    }

    // ============================================
    // 2. 状态转换
    // ============================================
    switch inbox.Status {
    case store.UNREAD:
        notification.Status = v1pb.UserNotification_UNREAD
    case store.ARCHIVED:
        notification.Status = v1pb.UserNotification_ARCHIVED
    default:
        notification.Status = v1pb.UserNotification_STATUS_UNSPECIFIED
    }

    // ============================================
    // 3. 类型和载荷转换
    // ============================================
    if inbox.Message != nil {
        switch inbox.Message.Type {
        case storepb.InboxMessage_MEMO_COMMENT:
            notification.Type = v1pb.UserNotification_MEMO_COMMENT
            payload, err := s.convertMemoCommentNotificationPayload(viewer, inbox.Message, memosByID)
            if payload != nil {
                notification.Payload = &v1pb.UserNotification_MemoComment{
                    MemoComment: payload,
                }
            }

        case storepb.InboxMessage_MEMO_MENTION:
            notification.Type = v1pb.UserNotification_MEMO_MENTION
            payload, err := s.convertMemoMentionNotificationPayload(viewer, inbox.Message, memosByID)
            if payload != nil {
                notification.Payload = &v1pb.UserNotification_MemoMention{
                    MemoMention: payload,
                }
            }

        default:
            notification.Type = v1pb.UserNotification_TYPE_UNSPECIFIED
        }
    }

    return notification, nil
}
```

### 4.3 载荷转换（含内容摘要）

**代码路径**: `server/router/api/v1/user_service.go:1841-1900+`

```go
func (s *APIV1Service) convertMemoCommentNotificationPayload(
    viewer *store.User, 
    message *storepb.InboxMessage, 
    memosByID map[int32]*store.Memo,
) (*v1pb.UserNotification_MemoCommentPayload, error) {
    memoComment := message.GetMemoComment()
    
    // 1. 获取关联的备忘录
    commentMemo := memosByID[memoComment.MemoId]
    relatedMemo := memosByID[memoComment.RelatedMemoId]
    
    // 2. 权限检查（确保接收者能看到这些备忘录）
    if !canViewerAccessMemo(viewer, commentMemo) || !canViewerAccessMemo(viewer, relatedMemo) {
        return nil, nil  // 无权限则返回空载荷
    }
    
    // 3. 生成内容摘要（用于前端展示）
    memoSnippet, err := s.memoNotificationSnippet(commentMemo)  // 截取前 64 字符
    relatedMemoSnippet, err := s.memoNotificationSnippet(relatedMemo)
    
    // 4. 构建返回载荷
    return &v1pb.UserNotification_MemoCommentPayload{
        Memo:                BuildMemoName(commentMemo.UID),           // 资源名
        MemoSnippet:         memoSnippet,                                // 摘要
        RelatedMemo:         BuildMemoName(relatedMemo.UID),           // 原备忘录
        RelatedMemoSnippet:  relatedMemoSnippet,                        // 原备忘录摘要
    }, nil
}
```

### 4.4 更新通知状态

**代码路径**: `server/router/api/v1/user_service.go:1597-1669`

```go
func (s *APIV1Service) UpdateUserNotification(ctx context.Context, request *v1pb.UpdateUserNotificationRequest) (*v1pb.UserNotification, error) {
    // 1. 解析通知名称 "users/xxx/notifications/123"
    user, notificationID, err := s.resolveUserAndNotificationIDFromName(ctx, request.Notification.Name)
    
    // 2. 权限验证
    currentUser, err := s.fetchCurrentUser(ctx)
    if currentUser.ID != user.ID {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
    
    // 3. 验证所有权（确保通知属于当前用户）
    inboxes, err := s.Store.ListInboxes(ctx, &store.FindInbox{ID: &notificationID})
    inbox := inboxes[0]
    if inbox.ReceiverID != currentUser.ID {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
    
    // 4. 构建更新请求
    update := &store.UpdateInbox{
        ID: notificationID,
    }
    
    for _, path := range request.UpdateMask.Paths {
        switch path {
        case "status":
            var inboxStatus store.InboxStatus
            switch request.Notification.Status {
            case v1pb.UserNotification_UNREAD:
                inboxStatus = store.UNREAD
            case v1pb.UserNotification_ARCHIVED:
                inboxStatus = store.ARCHIVED
            default:
                return nil, status.Errorf(codes.InvalidArgument, "invalid status")
            }
            update.Status = inboxStatus
        // 目前只支持更新 status
        }
    }
    
    // 5. 执行更新
    updatedInbox, err := s.Store.UpdateInbox(ctx, update)
    
    // 6. 转换并返回
    notification, err := s.convertInboxToUserNotification(ctx, updatedInbox, currentUser)
    return notification, nil
}
```

### 4.5 ListUserNotifications 完整时序图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        ListUserNotifications 完整时序图                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  前端组件                    Connect RPC                 后端服务                         │
│  ────────                    ───────────                 ────────                         │
│                                                                                          │
│  ┌──────────┐                                                                           │
│  │ Inboxes  │                                                                           │
│  │ 页面     │                                                                           │
│  └────┬─────┘                                                                           │
│       │                                                                                 │
│       │  1. useNotifications()                                                          │
│       │  ─────────────────────────────────────────────────────────────────────────▶    │
│       │                                                                                 │
│       ▼                                                                                 │
│  ┌──────────────────┐                                                                   │
│  │ useUserQueries.ts│                                                                   │
│  │ useNotifications │                                                                   │
│  └─────────┬────────┘                                                                   │
│            │                                                                            │
│            │  2. React Query 检查缓存                                                   │
│            │     - 有缓存且 staleTime(30s) 未过期：直接返回                            │
│            │     - 无缓存或已过期：调用 API                                            │
│            │                                                                             │
│            ▼                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                    Connect RPC 调用                                               │  │
│  │  userServiceClient.listUserNotifications({ parent: "users/xxx" })              │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│            │                                                                             │
│            ▼                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                         后端服务处理                                              │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 阶段 1：权限验证                                                         │   │  │
│  │  │ - 从 "users/xxx" 解析目标用户 ID                                        │   │  │
│  │  │ - 验证当前用户是否为目标用户本人                                         │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                    │                                              │  │
│  │                                    ▼                                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 阶段 2：数据查询                                                         │   │  │
│  │  │ - Store.ListInboxes(ctx, &FindInbox{ReceiverID: &userID})             │   │  │
│  │  │   └─▶ SELECT * FROM inbox WHERE receiver_id = ? ORDER BY created_ts DESC│   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                    │                                              │  │
│  │                                    ▼                                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 阶段 3：批量获取关联数据（优化 N+1）                                     │   │  │
│  │  │ - 收集所有 sender_id, receiver_id → 批量查询用户                        │   │  │
│  │  │ - 收集所有 memo_id, related_memo_id → 批量查询备忘录                    │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                    │                                              │  │
│  │                                    ▼                                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 阶段 4：转换为 API 结构                                                  │   │  │
│  │  │ - convertInboxToUserNotificationWithUsersAndMemos()                    │   │  │
│  │  │   ├─▶ 构建资源名："users/xxx/notifications/123"                        │   │  │
│  │  │   ├─▶ 转换状态：UNREAD/ARCHIVED                                         │   │  │
│  │  │   ├─▶ 转换类型：MEMO_COMMENT/MEMO_MENTION                               │   │  │
│  │  │   ├─▶ 构建载荷（含内容摘要）                                             │   │  │
│  │  │   └─▶ 权限检查：跳过接收者无权限访问的通知                               │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                    │                                              │  │
│  │                                    ▼                                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 阶段 5：返回响应                                                         │   │  │
│  │  │ - ListUserNotificationsResponse{ Notifications: [...] }                 │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                   │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│            │                                                                             │
│            ▼                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                         前端数据处理                                              │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                   │  │
│  │  1. React Query 缓存响应数据                                                     │  │
│  │     queryKey: ["users", "notifications"]                                        │  │
│  │     staleTime: 30 秒                                                             │  │
│  │                                                                                   │  │
│  │  2. Inboxes.tsx 处理数据                                                         │  │
│  │     - 按 created_ts 倒序排序                                                     │  │
│  │     - 过滤显示：all / unread / archived                                         │  │
│  │     - 统计未读数量：unreadCount                                                  │  │
│  │                                                                                   │  │
│  │  3. 渲染组件                                                                      │  │
│  │     - MEMO_COMMENT → MemoCommentMessage.tsx                                     │  │
│  │     - MEMO_MENTION → MemoMentionMessage.tsx                                     │  │
│  │                                                                                   │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 旧 Activity 与现有 Inbox/统计 对照表

| 维度 | 旧版 Activity 表 (≤ 0.26) | 现有 Inbox 表 (≥ 0.27) | 现有活动统计 (动态聚合) |
|------|---------------------------|------------------------|------------------------|
| **数据来源** | 独立的 `activity` 表 | `inbox` 表 | 从 `memo` 表动态聚合 |
| **存储方式** | 独立表，Inbox 通过 `activityId` 引用 | 自包含，载荷直接内嵌在 `message` 字段 | 无持久化，按需计算 |
| **触发场景** | 所有用户活动 | 仅评论、提及等需用户注意的事件 | 备忘录创建/更新 |
| **数据结构** | 通用结构：`type`, `level`, `payload` | 专用结构：`MEMO_COMMENT`, `MEMO_MENTION` | 时间戳数组、标签计数等 |
| **前端展示** | - | Inbox 通知页面、导航栏角标 | 热力图、标签云、统计面板 |
| **查询方式** | 直接查询 `activity` 表 | 查询 `inbox` 表（按 receiver_id 过滤） | 查询 `memo` 表后聚合 |
| **权限控制** | - | 仅接收者本人可查看 | 根据备忘录 visibility 过滤 |
| **状态管理** | - | UNREAD / ARCHIVED | 无状态概念 |
| **迁移状态** | 已删除 (`DROP TABLE`) | 当前使用中 | 当前使用中 |

### 5.1 数据迁移映射表

| 旧版结构 | 新版结构 | 迁移方式 |
|----------|----------|----------|
| `inbox.message.activityId` | 已移除 | 迁移时提取数据后删除此字段 |
| `activity.payload.memoComment.memoId` | `inbox.message.memoComment.memoId` | 从 activity 表提取后内嵌 |
| `activity.payload.memoComment.relatedMemoId` | `inbox.message.memoComment.relatedMemoId` | 从 activity 表提取后内嵌 |
| `activity` 表本身 | 已删除 | 迁移完成后 `DROP TABLE` |

---

## 6. 后端事件生成流程

### 6.1 触发场景

系统中有两个主要场景会生成 Inbox 消息：

#### 场景一：创建评论 (Memo Comment)

当用户在备忘录下创建评论时，系统会通知原备忘录的作者。

**代码路径**: `server/router/api/v1/memo_service.go:680-713`

```go
// 当评论不是私有且评论者不是原作者时
if memoComment.Visibility != v1pb.Visibility_PRIVATE && creatorID != relatedMemo.CreatorID {
    if _, err := s.createInboxWithEmailNotification(ctx, &store.Inbox{
        SenderID:   creatorID,           // 评论者
        ReceiverID: relatedMemo.CreatorID, // 原备忘录作者
        Status:     store.UNREAD,
        Message: &storepb.InboxMessage{
            Type: storepb.InboxMessage_MEMO_COMMENT,
            Payload: &storepb.InboxMessage_MemoComment{
                MemoComment: &storepb.InboxMessage_MemoCommentPayload{
                    MemoId:        memo.ID,        // 评论备忘录 ID
                    RelatedMemoId: relatedMemo.ID, // 原备忘录 ID
                },
            },
        },
    }); err != nil {
        return nil, status.Errorf(codes.Internal, "failed to create inbox")
    }
}
```

#### 场景二：用户被提及 (Memo Mention)

当用户在备忘录或评论中使用 `@username` 提及其他用户时，被提及的用户会收到通知。

**代码路径**: `server/router/api/v1/memo_mention_helpers.go:91-140`

```go
func (s *APIV1Service) dispatchMemoMentionNotifications(ctx context.Context, memo *store.Memo, relatedMemo *store.Memo, previousContent string) error {
    // 1. 解析内容中的 @username 提及
    currentTargets, err := s.resolveMentionTargets(ctx, memo.Content)
    
    // 2. 对比之前的内容，找出新增的提及
    previousTargets, err := s.resolveMentionTargets(ctx, previousContent)
    
    for userID, target := range currentTargets {
        if _, exists := previousTargets[userID]; exists {
            continue // 跳过已存在的提及
        }
        if shouldSkipMentionInbox(target, memo, relatedMemo) {
            continue // 跳过不需要通知的情况
        }
        
        // 3. 创建 Inbox 消息
        if _, err := s.createInboxWithEmailNotification(ctx, &store.Inbox{
            SenderID:   memo.CreatorID,
            ReceiverID: target.ID,
            Status:     store.UNREAD,
            Message: &storepb.InboxMessage{
                Type: storepb.InboxMessage_MEMO_MENTION,
                Payload: &storepb.InboxMessage_MemoMention{
                    MemoMention: &storepb.InboxMessage_MemoMentionPayload{
                        MemoId:        memo.ID,
                        RelatedMemoId: relatedMemo.ID, // 可选，评论场景
                    },
                },
            },
        }); err != nil {
            return errors.Wrap(err, "failed to create mention inbox")
        }
    }
    return nil
}
```

### 6.2 提及过滤逻辑

系统会智能过滤不需要发送通知的情况：

**代码路径**: `server/router/api/v1/memo_mention_helpers.go:74-89`

```go
func shouldSkipMentionInbox(target *store.User, memo *store.Memo, relatedMemo *store.Memo) bool {
    // 1. 跳过提及自己
    if target.ID == memo.CreatorID {
        return true
    }
    
    // 2. 评论场景中，原作者已经收到评论通知，跳过重复的提及通知
    if relatedMemo != nil && target.ID == relatedMemo.CreatorID && 
       memo.Visibility != store.Private && memo.CreatorID != relatedMemo.CreatorID {
        return true
    }
    
    // 3. 检查目标用户是否有权限访问该备忘录
    return !canUserAccessMentionContext(target, memo, relatedMemo)
}
```

---

## 7. 数据存储层

### 7.1 数据结构定义

**Proto 定义**: `proto/store/inbox.proto`

```protobuf
message InboxMessage {
  // 评论通知载荷
  message MemoCommentPayload {
    int32 memo_id = 1;           // 评论备忘录 ID
    int32 related_memo_id = 2;   // 原备忘录 ID
  }

  // 提及通知载荷
  message MemoMentionPayload {
    int32 memo_id = 1;           // 提及所在的备忘录 ID
    int32 related_memo_id = 2;   // 关联的原备忘录 ID（评论场景）
  }

  Type type = 1;           // 消息类型
  oneof payload {          // 载荷
    MemoCommentPayload memo_comment = 2;
    MemoMentionPayload memo_mention = 3;
  }

  enum Type {
    TYPE_UNSPECIFIED = 0;
    MEMO_COMMENT = 1;     // 评论通知
    MEMO_MENTION = 2;     // 提及通知
  }
}
```

**Go 结构体**: `store/inbox.go`

```go
type Inbox struct {
    ID         int32
    CreatedTs  int64                  // 创建时间戳
    SenderID   int32                  // 发送者用户 ID
    ReceiverID int32                  // 接收者用户 ID
    Status     InboxStatus            // 状态: UNREAD / ARCHIVED
    Message    *storepb.InboxMessage  // 消息内容（Protobuf 序列化存储）
}
```

### 7.2 数据库表结构

**迁移脚本**: `store/migration/sqlite/0.17/00__inbox.sql`

```sql
CREATE TABLE IF NOT EXISTS `inbox` (
  `id` INTEGER PRIMARY KEY AUTOINCREMENT,
  `created_ts` BIGINT NOT NULL DEFAULT (strftime('%s', 'now')),
  `sender_id` INTEGER NOT NULL,
  `receiver_id` INTEGER NOT NULL,
  `status` TEXT NOT NULL DEFAULT 'UNREAD',
  `message` TEXT NOT NULL DEFAULT '{}'  -- JSON 格式存储 Protobuf
);

-- 索引优化查询
CREATE INDEX IF NOT EXISTS `idx_inbox_receiver_id` ON `inbox`(`receiver_id`);
CREATE INDEX IF NOT EXISTS `idx_inbox_status` ON `inbox`(`status`);
```

### 7.3 存储操作

**SQLite 实现**: `store/db/sqlite/inbox.go`

| 操作 | 方法 | 说明 |
|------|------|------|
| 创建 | `CreateInbox()` | 插入新记录，返回带 ID 和时间戳的对象 |
| 查询 | `ListInboxes()` | 支持按 ReceiverID、Status、MessageType 过滤 |
| 更新 | `UpdateInbox()` | 目前仅支持更新状态（标记已读/归档） |
| 删除 | `DeleteInbox()` | 物理删除记录 |

**消息序列化**：使用 `protojson.Marshal()` 将 Protobuf 消息序列化为 JSON 存储。

---

## 8. 多通道通知分发

当 Inbox 消息创建时，系统会通过多个通道进行通知分发：

```
                    ┌─────────────────────────────────────────┐
                    │         createInboxWithEmailNotification │
                    └────────────────────┬────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
    ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
    │   Inbox 存储    │        │   邮件通知      │        │  (无直接关联)    │
    │   (持久化)      │        │  (可选发送)     │        │  SSE/Webhook    │
    └─────────────────┘        └─────────────────┘        └─────────────────┘
```

### 8.1 核心分发入口

**代码路径**: `server/router/api/v1/notification_email.go:11-28`

```go
func (s *APIV1Service) createInboxWithEmailNotification(ctx context.Context, inbox *store.Inbox) (*store.Inbox, error) {
    // 1. 首先持久化到数据库
    createdInbox, err := s.Store.CreateInbox(ctx, inbox)
    if err != nil {
        return nil, err
    }
    
    // 2. 尽力发送邮件通知（失败不影响主流程）
    s.dispatchInboxEmailNotificationBestEffort(ctx, createdInbox)
    return createdInbox, nil
}
```

### 8.2 邮件通知

**代码路径**: `server/notification/email.go:40-97`

```go
func (d *EmailDispatcher) DispatchInboxEmail(ctx context.Context, inbox *store.Inbox) error {
    // 1. 检查实例是否启用了邮件通知
    setting, err := d.store.GetInstanceNotificationSetting(ctx)
    emailSetting := setting.GetEmail()
    if emailSetting == nil || !emailSetting.Enabled {
        return nil  // 未启用，直接返回
    }
    
    // 2. 检查接收者是否有邮箱地址
    receiver, err := d.store.GetUser(ctx, &store.FindUser{ID: &inbox.ReceiverID})
    if receiver == nil || strings.TrimSpace(receiver.Email) == "" {
        return nil
    }
    
    // 3. 构建邮件内容
    message, err := d.buildInboxEmailMessage(inbox, receiver, sender, memosByID)
    if message == nil {
        return nil
    }
    
    // 4. 异步发送邮件
    d.sender(config, message)  // 默认使用 email.SendAsync
    return nil
}
```

**邮件模板示例**（评论通知）：

```
Hi [接收者昵称],

[发送者昵称] commented on your memo.

Open in Memos:
https://your-memos-instance.com/memos/xxx#yyy

You are receiving this because you own this memo.
```

### 8.3 SSE 实时事件（独立通道）

**注意**: SSE 事件与 Inbox 消息是**独立的两个系统**，但有部分重叠场景。

#### SSE 事件类型

**代码路径**: `server/router/api/v1/sse_hub.go:11-21`

```go
const (
    SSEEventMemoCreated        SSEEventType = "memo.created"
    SSEEventMemoUpdated        SSEEventType = "memo.updated"
    SSEEventMemoDeleted        SSEEventType = "memo.deleted"
    SSEEventMemoCommentCreated SSEEventType = "memo.comment.created"  // 与 Inbox 重叠
    SSEEventReactionUpserted   SSEEventType = "reaction.upserted"
    SSEEventReactionDeleted    SSEEventType = "reaction.deleted"
)
```

#### SSE 事件结构

```go
type SSEEvent struct {
    Type       SSEEventType     `json:"type"`       // 事件类型
    Name       string           `json:"name"`       // 资源名，如 "memos/xxxx"
    Parent     string           `json:"parent,omitempty"` // 父资源（评论场景）
    Visibility store.Visibility `json:"-"`          // 服务端过滤用，不序列化
    CreatorID  int32            `json:"-"`          // 服务端过滤用，不序列化
}
```

#### SSE Hub 广播机制

**代码路径**: `server/router/api/v1/sse_hub.go:56-144`

```go
type SSEHub struct {
    mu      sync.RWMutex
    clients map[*SSEClient]struct{}  // 所有连接的客户端
    closed  bool
}

// 广播事件给所有符合条件的客户端
func (h *SSEHub) Broadcast(event *SSEEvent) {
    data := event.JSON()
    if len(data) == 0 {
        return
    }
    
    h.mu.RLock()
    defer h.mu.RUnlock()
    
    for c := range h.clients {
        if !c.canReceive(event) {
            continue  // 权限检查
        }
        select {
        case c.events <- data:
        default:
            // 慢客户端丢弃事件，避免阻塞
        }
    }
}

// 权限检查：私有备忘录只有作者和管理员能收到
func (c *SSEClient) canReceive(event *SSEEvent) bool {
    switch event.Visibility {
    case store.Private:
        return c.userID == event.CreatorID || c.role == store.RoleAdmin
    case store.Public, store.Protected, "":
        return true
    default:
        return false
    }
}
```

#### 评论创建时的完整事件流

**代码路径**: `server/router/api/v1/memo_service.go:680-713`

```go
// 1. 创建 Inbox 消息 + 邮件通知
if memoComment.Visibility != v1pb.Visibility_PRIVATE && creatorID != relatedMemo.CreatorID {
    s.createInboxWithEmailNotification(ctx, &store.Inbox{...})
}

// 2. 触发 Webhook（通知外部系统）
s.DispatchMemoCommentCreatedWebhook(ctx, memoComment, relatedMemo.CreatorID)

// 3. 触发提及通知（评论内容中可能有 @mention）
s.dispatchMemoMentionNotificationsBestEffort(ctx, memo, relatedMemo, "")

// 4. 广播 SSE 事件（实时更新前端）
s.SSEHub.Broadcast(&SSEEvent{
    Type:       SSEEventMemoCommentCreated,
    Name:       request.Name,
    Visibility: relatedMemo.Visibility,
    CreatorID:  relatedMemo.CreatorID,
})
```

### 8.4 Webhook 通知（独立通道）

Webhook 用于将事件推送到外部系统，与 Inbox 消息也是独立的。

**代码路径**: `internal/webhook/webhook.go:126-140`

```go
// 异步队列，容量 128
var asyncPostQueue = make(chan *WebhookRequestPayload, 128)

func init() {
    // 启动 4 个 worker 处理队列
    for range 4 {
        go func() {
            for payload := range asyncPostQueue {
                if err := Post(payload); err != nil {
                    slog.Warn("Failed to dispatch webhook asynchronously", ...)
                }
            }
        }()
    }
}

func PostAsync(requestPayload *WebhookRequestPayload) {
    select {
    case asyncPostQueue <- requestPayload:
    default:
        // 队列满时丢弃
        slog.Warn("Dropped webhook dispatch because the async queue is full", ...)
    }
}
```

**Webhook 载荷结构**:

```go
type WebhookRequestPayload struct {
    URL          string       `json:"url"`           // 目标 URL
    ActivityType string       `json:"activityType"`  // 活动类型
    Creator      string       `json:"creator"`       // 创建者资源名
    Memo         *v1pb.Memo   `json:"memo"`          // 关联的备忘录
}
```

---

## 9. 前端数据流

### 9.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端数据流架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐     ┌──────────────────────┐     ┌──────────────────┐   │
│  │ SSE 连接     │────▶│ React Query 缓存    │────▶│ UI 组件渲染      │   │
│  │ (实时事件)   │     │ (状态管理)           │     │ (通知展示)       │   │
│  └──────────────┘     └──────────────────────┘     └──────────────────┘   │
│         │                      ▲                      ▲                      │
│         │                      │                      │                      │
│         ▼                      │                      │                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     Connect RPC 客户端 (API 调用)                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 SSE 实时连接管理

**代码路径**: `web/src/hooks/useLiveMemoRefresh.ts`

这是一个单例 Hook，管理全局 SSE 连接：

```typescript
// 重连参数
const INITIAL_RETRY_DELAY_MS = 1000;
const MAX_RETRY_DELAY_MS = 30000;
const RETRY_BACKOFF_MULTIPLIER = 2;

// 支持的 SSE 事件类型
const SSE_EVENT_TYPES = {
  memoCreated: "memo.created",
  memoUpdated: "memo.updated",
  memoDeleted: "memo.deleted",
  memoCommentCreated: "memo.comment.created",
  reactionUpserted: "reaction.upserted",
  reactionDeleted: "reaction.deleted",
} as const;
```

#### 连接状态管理（单例模式）

```typescript
// 全局状态存储
let _status: SSEConnectionStatus = "disconnected";
const _listeners = new Set<Listener>();

function setSSEStatus(s: SSEConnectionStatus) {
  if (_status !== s) {
    _status = s;
    _listeners.forEach((l) => l());  // 通知所有订阅者
  }
}

// React Hook 读取连接状态
export function useSSEConnectionStatus(): SSEConnectionStatus {
  return useSyncExternalStore(subscribeSSEStatus, getSSEStatus, getSSEStatus);
}
```

#### 事件处理与缓存失效

```typescript
function handleSSEEvent(event: SSEChangeEvent, queryClient: QueryClient) {
  switch (event.type) {
    case SSE_EVENT_TYPES.memoCreated:
      // 使备忘录列表缓存失效
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      queryClient.invalidateQueries({ queryKey: userKeys.stats() });
      break;

    case SSE_EVENT_TYPES.memoUpdated:
      // 使单个备忘录缓存失效
      queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      if (event.parent) {
        queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.parent) });
      }
      break;

    case SSE_EVENT_TYPES.memoCommentCreated:
      // ⚠️ 注意：这里只刷新了评论列表，没有刷新通知列表！
      queryClient.invalidateQueries({ queryKey: memoKeys.comments(event.name) });
      queryClient.invalidateQueries({ queryKey: memoKeys.detail(event.name) });
      break;
    // ... 其他事件
  }
}
```

**重要发现**: `memo.comment.created` 事件目前不会触发通知列表的刷新。这意味着新的 Inbox 消息到达时，前端可能不会立即感知，需要依赖 `useNotifications` 的 `staleTime` 过期后重新拉取。

### 9.3 通知数据获取

**代码路径**: `web/src/hooks/useUserQueries.ts:61-76`

```typescript
export function useNotifications() {
  const currentUser = useCurrentUser();

  return useQuery({
    queryKey: userKeys.notifications(),  // ["users", "notifications"]
    queryFn: async () => {
      if (!currentUser?.name) {
        return [];
      }
      // 调用 Connect RPC API
      const { notifications } = await userServiceClient.listUserNotifications({ 
        parent: currentUser.name 
      });
      return notifications;
    },
    enabled: !!currentUser?.name,
    staleTime: 1000 * 30,  // 30 秒后缓存过期
  });
}
```

### 9.4 UI 展示层

#### 导航栏通知入口

**代码路径**: `web/src/components/Navigation.tsx:49-64`

```typescript
const Navigation = () => {
  const currentUser = useCurrentUser();
  const { data: notifications = [] } = useNotifications();

  // 计算未读数量
  const unreadCount = notifications.filter(
    (n) => n.status === UserNotification_Status.UNREAD
  ).length;

  const inboxNavLink: NavLinkItem = {
    id: "header-inbox",
    path: Routes.INBOX,
    title: t("common.inbox"),
    icon: (
      <div className="relative">
        <BellIcon className="w-6 h-auto shrink-0" />
        {/* 未读数量角标 */}
        {unreadCount > 0 && (
          <span className="absolute -top-1 -right-1 ...">
            {unreadCount > 99 ? "99+" : unreadCount}
          </span>
        )}
      </div>
    ),
  };
  // ...
};
```

#### Inbox 页面

**代码路径**: `web/src/pages/Inboxes.tsx`

```typescript
const Inboxes = () => {
  const [filter, setFilter] = useState<"all" | "unread" | "archived">("all");
  
  // 获取通知数据
  const { data: fetchedNotifications = [] } = useNotifications();

  // 按时间倒序排序
  const allNotifications = sortBy(fetchedNotifications, (notification) => {
    return -((notification.createTime ? timestampDate(notification.createTime) : undefined)?.getTime() || 0);
  });

  // 过滤显示
  const notifications = allNotifications.filter((notification) => {
    if (filter === "unread") return notification.status === UserNotification_Status.UNREAD;
    if (filter === "archived") return notification.status === UserNotification_Status.ARCHIVED;
    return true;
  });

  // 统计
  const unreadCount = allNotifications.filter((n) => n.status === UserNotification_Status.UNREAD).length;
  const archivedCount = allNotifications.filter((n) => n.status === UserNotification_Status.ARCHIVED).length;

  return (
    // ... 渲染 UI
    // 根据通知类型渲染不同组件
    {notifications.map((notification) => {
      if (notification.type === UserNotification_Type.MEMO_COMMENT) {
        return <MemoCommentMessage key={notification.name} notification={notification} />;
      }
      if (notification.type === UserNotification_Type.MEMO_MENTION) {
        return <MemoMentionMessage key={notification.name} notification={notification} />;
      }
      return null;
    })}
  );
};
```

#### 消息组件示例（提及通知）

**代码路径**: `web/src/components/Inbox/MemoMentionMessage.tsx`

```typescript
function MemoMentionMessage({ notification }: Props) {
  const mentionPayload = notification.payload?.case === "memoMention" 
    ? notification.payload.value 
    : undefined;
  const sender = notification.senderUser;
  const isUnread = notification.status === UserNotification_Status.UNREAD;

  const handleNavigate = async () => {
    navigateTo(`/${targetName}`);
    // 点击后自动标记为已读（归档）
    if (isUnread) {
      await handleArchiveMessage(true);
    }
  };

  const handleArchiveMessage = async (silence = false) => {
    await userServiceClient.updateUserNotification({
      notification: {
        name: notification.name,
        status: UserNotification_Status.ARCHIVED,
      },
      updateMask: create(FieldMaskSchema, { paths: ["status"] }),
    });
    // 更新后 React Query 会自动重新获取
  };

  return (
    // 渲染 UI，包含：
    // - 发送者头像
    // - 提及图标 (@)
    // - 消息内容预览
    // - 未读标识（左侧蓝色条）
    // - 操作按钮（标记已读/删除）
  );
}
```

---

## 10. 完整数据流时序图

### 10.1 创建评论触发通知

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐     ┌──────────┐
│  用户A  │     │ MemoService  │     │ Inbox Store │     │  SSE Hub    │     │  Webhook │
└────┬────┘     └──────┬───────┘     └──────┬──────┘     └──────┬──────┘     └────┬─────┘
     │                  │                     │                     │                  │
     │  创建评论请求     │                     │                     │                  │
     │─────────────────>│                     │                     │                  │
     │                  │                     │                     │                  │
     │                  │ 1. 保存评论数据      │                     │                  │
     │                  │                     │                     │                  │
     │                  │ 2. 检查是否需要通知   │                     │                  │
     │                  │ (非私有 + 非作者)    │                     │                  │
     │                  │                     │                     │                  │
     │                  │ 3. 创建 Inbox       │                     │                  │
     │                  │────────────────────>│                     │                  │
     │                  │                     │ INSERT INTO inbox   │                  │
     │                  │                     │                     │                  │
     │                  │ 4. 发送邮件通知      │                     │                  │
     │                  │ (异步，失败不影响)   │                     │                  │
     │                  │                     │                     │                  │
     │                  │ 5. 触发 Webhook     │                     │                  │
     │                  │────────────────────────────────────────────>│                  │
     │                  │                     │                     │ PostAsync()      │
     │                  │                     │                     │                  │
     │                  │ 6. 处理评论中提及     │                     │                  │
     │                  │ (dispatchMention)   │                     │                  │
     │                  │                     │                     │                  │
     │                  │ 7. 广播 SSE 事件    │                     │                  │
     │                  │───────────────────────────────────────────>│                  │
     │                  │                     │                     │ Broadcast()      │
     │                  │                     │                     │                  │
     │                  │ 返回评论数据          │                     │                  │
     │<─────────────────│                     │                     │                  │
     │                  │                     │                     │                  │
```

### 10.2 前端实时更新流程

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────┐     ┌─────────────┐
│ SSE 连接     │     │ useLiveMemoRefresh │    │ React Query   │     │ UI 组件     │
└──────┬───────┘     └─────────┬────────┘     └───────┬───────┘     └──────┬──────┘
       │                        │                      │                    │
       │  memo.comment.created  │                      │                    │
       │───────────────────────>│                      │                    │
       │                        │                      │                    │
       │                        │ 1. 解析事件类型        │                    │
       │                        │                      │                    │
       │                        │ 2. 使相关缓存失效      │                    │
       │                        │ invalidateQueries()  │                    │
       │                        │─────────────────────>│                    │
       │                        │                      │                    │
       │                        │                      │ ⚠️ 注意：            │                    │
       │                        │                      │ 只刷新了 memo 相关    │                    │
       │                        │                      │ 没有刷新 notifications │                    │
       │                        │                      │                    │
       │                        │                      │ 3. 等待 staleTime    │                    │
       │                        │                      │ (30秒) 后自动重取    │                    │
       │                        │                      │────────────────────>│
       │                        │                      │                    │
       │                        │                      │ 4. 重新获取通知数据   │                    │
       │                        │                      │ listUserNotifications │
       │                        │                      │<────────────────────│
       │                        │                      │                    │
       │                        │                      │ 5. 通知更新          │                    │
       │                        │                      │────────────────────>