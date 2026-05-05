# Memos Inbox Activity 数据流分析报告

## 1. 概述

本报告详细分析了 Memos 项目中 **Inbox 消息**、**用户活动记录**和**通知入口**的完整数据流串联关系。从后端事件生成到前端通知展示，涵盖了所有关键组件和交互路径。

### 核心概念区分

| 概念 | 定义 | 存储位置 | 触发场景 |
|------|------|----------|----------|
| **Inbox 消息** | 用户收到的通知（评论、提及） | `inbox` 表 | 评论创建、用户被提及 |
| **SSE 实时事件** | 服务器推送到客户端的实时更新 | 内存 (SSEHub) | 备忘录增删改、评论、反应 |
| **Webhook** | 推送到外部系统的事件通知 | 无持久化 | 备忘录增删改、评论创建 |
| **邮件通知** | 发送给用户的邮件提醒 | 无持久化 | Inbox 消息创建（可选） |

---

## 2. 后端事件生成流程

### 2.1 触发场景

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

### 2.2 提及过滤逻辑

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

## 3. 数据存储层

### 3.1 数据结构定义

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

### 3.2 数据库表结构

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

### 3.3 存储操作

**SQLite 实现**: `store/db/sqlite/inbox.go`

| 操作 | 方法 | 说明 |
|------|------|------|
| 创建 | `CreateInbox()` | 插入新记录，返回带 ID 和时间戳的对象 |
| 查询 | `ListInboxes()` | 支持按 ReceiverID、Status、MessageType 过滤 |
| 更新 | `UpdateInbox()` | 目前仅支持更新状态（标记已读/归档） |
| 删除 | `DeleteInbox()` | 物理删除记录 |

**消息序列化**：使用 `protojson.Marshal()` 将 Protobuf 消息序列化为 JSON 存储。

---

## 4. 多通道通知分发

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

### 4.1 核心分发入口

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

### 4.2 邮件通知

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

### 4.3 SSE 实时事件（独立通道）

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

### 4.4 Webhook 通知（独立通道）

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

## 5. 前端数据流

### 5.1 整体架构

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

### 5.2 SSE 实时连接管理

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

### 5.3 通知数据获取

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

### 5.4 UI 展示层

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

## 6. 完整数据流时序图

### 6.1 创建评论触发通知

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

### 6.2 前端实时更新流程

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
       │                        │                      │────────────────────>│
       │                        │                      │                    │ 重新渲染
       │                        │                      │                    │ 未读计数更新
```

---

## 7. 关键数据结构映射关系

### 7.1 后端到前端的转换

| 后端 (Go) | 前端 (TypeScript) | 说明 |
|-----------|-------------------|------|
| `store.Inbox` | `UserNotification` | 通过 API 层转换 |
| `storepb.InboxMessage_MEMO_COMMENT` | `UserNotification_Type.MEMO_COMMENT` | 评论通知类型 |
| `storepb.InboxMessage_MEMO_MENTION` | `UserNotification_Type.MEMO_MENTION` | 提及通知类型 |
| `store.UNREAD` | `UserNotification_Status.UNREAD` | 未读状态 |
| `store.ARCHIVED` | `UserNotification_Status.ARCHIVED` | 已归档状态 |

### 7.2 事件与通知的对应关系

| 事件类型 | 是否生成 Inbox | 是否触发 SSE | 是否触发 Webhook |
|----------|----------------|--------------|------------------|
| 备忘录创建 | ❌ 否 | ✅ 是 | ✅ 是 |
| 备忘录更新 | ❌ 否 | ✅ 是 | ✅ 是 |
| 备忘录删除 | ❌ 否 | ✅ 是 | ✅ 是 |
| 评论创建 | ✅ 是 (通知原作者) | ✅ 是 | ✅ 是 |
| 用户被提及 | ✅ 是 (通知被提及者) | ❌ 否 | ❌ 否 |
| 反应添加/删除 | ❌ 否 | ✅ 是 | ❌ 否 |

---

## 8. 架构总结

### 8.1 系统设计亮点

1. **多通道通知**: Inbox 消息 + 邮件 + SSE + Webhook，满足不同场景需求
2. **尽力而为模式**: 邮件和 Webhook 采用异步发送，失败不影响主流程
3. **权限控制**: SSE 广播时根据备忘录可见性过滤接收者
4. **智能去重**: 评论场景中避免重复发送通知（原作者已收到评论通知，不再发送提及通知）
5. **前端缓存优化**: React Query + SSE 实现高效的实时更新

### 8.2 潜在改进点

1. **SSE 事件与 Inbox 的联动**
   - 目前 `memo.comment.created` 事件不会触发通知列表刷新
   - 建议：当收到评论或提及相关的 SSE 事件时，主动使 `userKeys.notifications()` 缓存失效

2. **通知类型扩展**
   - 目前只支持 `MEMO_COMMENT` 和 `MEMO_MENTION`
   - 可考虑扩展：`REACTION_CREATED`（有人点赞/反应）、`MEMO_SHARED`（备忘录被共享）等

3. **批量操作支持**
   - 前端目前只能单条标记已读/删除
   - 可考虑添加"全部标记已读"功能

---

## 9. 相关文件索引

### 后端

| 文件路径 | 职责 |
|----------|------|
| `store/inbox.go` | Inbox 数据结构定义 |
| `store/db/sqlite/inbox.go` | SQLite 存储实现 |
| `proto/store/inbox.proto` | Protobuf 定义 |
| `server/router/api/v1/memo_service.go` | 评论创建、事件广播 |
| `server/router/api/v1/memo_mention_helpers.go` | 提及解析与通知分发 |
| `server/router/api/v1/notification_email.go` | Inbox + 邮件通知入口 |
| `server/notification/email.go` | 邮件通知构建与发送 |
| `server/router/api/v1/sse_hub.go` | SSE 连接管理与广播 |
| `server/router/api/v1/sse_handler.go` | SSE HTTP 处理器 |
| `internal/webhook/webhook.go` | Webhook 异步发送 |

### 前端

| 文件路径 | 职责 |
|----------|------|
| `web/src/hooks/useLiveMemoRefresh.ts` | SSE 连接管理与事件处理 |
| `web/src/hooks/useUserQueries.ts` | 通知数据获取 Hook |
| `web/src/pages/Inboxes.tsx` | Inbox 页面主组件 |
| `web/src/components/Navigation.tsx` | 导航栏通知入口 |
| `web/src/components/Inbox/MemoMentionMessage.tsx` | 提及通知组件 |
| `web/src/components/Inbox/MemoCommentMessage.tsx` | 评论通知组件 |
