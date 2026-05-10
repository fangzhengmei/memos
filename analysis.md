# Memos 全局设置分析报告

## 一、全局设置概述

Memos 使用了两层设置系统：
- **实例设置 (Instance Setting)**: 全局系统级设置，由管理员配置
- **用户设置 (User Setting)**: 个人偏好设置，由用户配置

### 1.1 实例设置类型

根据 `store/instance_setting.go`，实例设置分为以下几类：

| 设置类型 | Key | 用途 |
|---------|-----|------|
| 基础设置 | BASIC | 未使用 |
| 通用设置 | GENERAL | 注册控制、密码认证、自定义配置等 |
| 存储设置 | STORAGE | 文件存储配置（本地/S3） |
| 备忘录相关 | MEMO_RELATED | 内容长度限制、双击编辑、表情反应 |
| 标签设置 | TAGS | 标签颜色和模糊内容 |
| 通知设置 | NOTIFICATION | 邮件通知配置 |
| AI 设置 | AI | AI 提供商和语音转写配置 |

### 1.2 设置存储机制

- **服务端缓存**: `store/instance_setting.go:62` 使用 `instanceSettingCache` (TTL 10分钟，最大 1000 条)
- **数据库存储**: 以 JSON 格式存储在数据库中
- **默认值处理**: 未设置时返回默认值，例如：
  - 内容长度限制默认 8KB
  - 默认表情反应列表

---

## 二、注册入口控制

### 2.1 双层控制机制

#### 服务端强制验证（`server/router/api/v1/user_service.go:195-207`）

```go
// 关键代码位置：server/router/api/v1/user_service.go:195-207
if roleToAssign != store.RoleAdmin {
    instanceGeneralSetting, err := s.Store.GetInstanceGeneralSetting(ctx)
    if instanceGeneralSetting.DisallowUserRegistration {
        return nil, status.Errorf(codes.PermissionDenied, "user registration is not allowed")
    }
    if instanceGeneralSetting.DisallowPasswordAuth {
        return nil, status.Errorf(codes.PermissionDenied, "password signup is not allowed")
    }
}
```

**验证逻辑：**
1. 第一个用户创建为 Admin，不受限制
2. 普通用户注册时必须检查：
   - `DisallowUserRegistration`: 是否允许用户注册
   - `DisallowPasswordAuth`: 是否允许密码注册

#### 前端隐藏控制（`web/src/pages/SignUp.tsx:33, 97-145`）

```typescript
// 关键代码位置：web/src/pages/SignUp.tsx:33
const canUsePasswordSignUp = !instanceGeneralSetting.disallowUserRegistration && !instanceGeneralSetting.disallowPasswordAuth;

// 条件渲染：web/src/pages/SignUp.tsx:97-145
{canUsePasswordSignUp ? (
  // 显示注册表单
) : instanceGeneralSetting.disallowPasswordAuth ? (
  <p>Password sign up is not allowed.</p>
) : (
  <p>Sign up is not allowed.</p>
)}
```

### 2.2 双层边界分析

**第一层：服务端强制（安全边界）**
- 即使前端绕过，服务端 API 也会拒绝
- 返回 gRPC 错误码 `PermissionDenied`
- 适用于所有 API 调用（包括直接 API 访问）

**第二层：前端隐藏（用户体验边界）**
- 根据设置动态显示/隐藏注册入口
- 提供友好的提示信息
- 减少无效请求

---

## 三、默认可见性控制

### 3.1 设置位置

默认可见性是**用户级设置**，存储在用户设置中：

```go
// 关键代码位置：server/router/api/v1/user_service.go:431-437
func getDefaultUserGeneralSetting() *v1pb.UserSetting_GeneralSetting {
    return &v1pb.UserSetting_GeneralSetting{
        Locale:         "en",
        MemoVisibility: "PRIVATE",  // 默认私有
        Theme:          "",
    }
}
```

### 3.2 前端使用

```typescript
// 关键代码位置：web/src/components/MemoEditor/index.tsx:67
const defaultVisibility = userGeneralSetting?.memoVisibility 
  ? convertVisibilityFromString(userGeneralSetting.memoVisibility) 
  : undefined;

// 初始化编辑器时使用：web/src/components/MemoEditor/index.tsx:69-77
const { isInitialized } = useMemoInit({
  editorRef,
  memo,
  cacheKey,
  username: currentUser?.name ?? "",
  autoFocus,
  defaultVisibility,  // 传入默认可见性
  defaultCreateTime,
});
```

### 3.3 服务端验证（`server/router/api/v1/memo_service.go:58-70`）

即使前端设置了可见性，服务端仍会在读取时验证权限：

```go
func (s *APIV1Service) checkMemoReadAccess(ctx context.Context, memo *store.Memo) error {
    if memo.Visibility != store.Public {
        user, err := s.fetchCurrentUser(ctx)
        if err != nil {
            return status.Errorf(codes.Internal, "failed to get user")
        }
        if user == nil {
            return status.Errorf(codes.Unauthenticated, "user not authenticated")
        }
        if memo.Visibility == store.Private && memo.CreatorID != user.ID {
            return status.Errorf(codes.PermissionDenied, "permission denied")
        }
    }
    return nil
}
```

---

## 四、前端功能显示控制

### 4.1 基于实例设置的功能控制

#### 密码认证开关（`web/src/components/Settings/InstanceSection.tsx:117-121`）

```typescript
<Switch
  disabled={profile.demo || (identityProviderList.length === 0 && !instanceGeneralSetting.disallowPasswordAuth)}
  checked={instanceGeneralSetting.disallowPasswordAuth}
  onCheckedChange={(checked) => updatePartialSetting({ disallowPasswordAuth: checked })}
/>
```

**禁用逻辑：**
- Demo 模式下禁用
- 如果没有配置身份提供商且当前已允许密码认证，则不允许禁用（防止锁定自己）

#### 备忘录相关功能（`web/src/components/Settings/MemoRelatedSettings.tsx`）

- `enableDoubleClickEdit`: 双击编辑功能开关
- `contentLengthLimit`: 内容长度限制
- `reactions`: 可用的表情反应列表

#### AI 功能显示（`web/src/components/MemoEditor/index.tsx:59-64`）

```typescript
const canTranscribe = useMemo(() => {
  const providerId = aiSetting.transcription?.providerId ?? "";
  if (!providerId) return false;
  const provider = aiSetting.providers.find((p) => p.id === providerId);
  return Boolean(provider?.apiKeySet);
}, [aiSetting.providers, aiSetting.transcription?.providerId]);
```

AI 转录功能只有在：
1. 配置了转写 provider ID
2. 该 provider 已设置 API Key
时才会显示

### 4.2 用户资料编辑限制

```go
// 关键代码位置：server/router/api/v1/user_service.go:282-294
case "username":
    if instanceGeneralSetting.DisallowChangeUsername {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied: disallow change username")
    }
case "display_name":
    if instanceGeneralSetting.DisallowChangeNickname {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied: disallow change nickname")
    }
```

---

## 五、设置变更后旧页面状态处理

### 5.1 实例设置更新流程

#### 1. 更新钩子（`web/src/components/Settings/useInstanceSettingUpdater.ts:16-34`）

```typescript
const useInstanceSettingUpdater = () => {
  const t = useTranslate();
  const { updateSetting, fetchSetting } = useInstance();

  return useCallback(
    async ({ key, setting, errorContext }: SaveInstanceSettingOptions) => {
      try {
        await updateSetting(setting);      // 1. 发送更新请求
        await fetchSetting(key);           // 2. 重新获取最新设置
        toast.success(t("message.update-succeed"));
        return true;
      } catch (error: unknown) {
        await handleError(error, toast.error, { context: errorContext });
        return false;
      }
    },
    [fetchSetting, t, updateSetting],
  );
};
```

#### 2. Context 状态同步（`web/src/contexts/InstanceContext.tsx:192-198`）

```typescript
const updateSetting = useCallback(async (setting: InstanceSetting) => {
  const updatedSetting = await instanceServiceClient.updateInstanceSetting({ setting });
  setState((prev) => ({
    ...prev,
    settings: [...prev.settings.filter((s) => s.name !== updatedSetting.name), updatedSetting],
  }));
}, []);
```

**更新机制：**
1. 调用服务端 API 更新设置
2. 用新设置替换旧设置（通过 `setState`）
3. 所有使用 `useInstance()` 的组件会自动响应式更新

### 5.2 实时动态更新的设置

#### App 级别的实时响应（`web/src/App.tsx:31-57`）

```typescript
// 额外样式实时应用
useEffect(() => {
  if (instanceGeneralSetting.additionalStyle) {
    const styleEl = document.createElement("style");
    styleEl.innerHTML = instanceGeneralSetting.additionalStyle;
    styleEl.setAttribute("type", "text/css");
    document.body.insertAdjacentElement("beforeend", styleEl);
  }
}, [instanceGeneralSetting.additionalStyle]);

// 额外脚本实时应用
useEffect(() => {
  if (instanceGeneralSetting.additionalScript) {
    const scriptEl = document.createElement("script");
    scriptEl.innerHTML = instanceGeneralSetting.additionalScript;
    document.head.appendChild(scriptEl);
  }
}, [instanceGeneralSetting.additionalScript]);

// 页面标题和图标实时更新
useEffect(() => {
  if (!instanceGeneralSetting.customProfile) {
    return;
  }
  document.title = instanceGeneralSetting.customProfile.title;
  const link = document.querySelector("link[rel~='icon']") as HTMLLinkElement;
  link.href = instanceGeneralSetting.customProfile.logoUrl || "/logo.webp";
}, [instanceGeneralSetting.customProfile]);
```

### 5.3 SSE 实时刷新机制

Memos 使用 SSE (Server-Sent Events) 实现备忘录的实时更新，但**不用于实例设置的实时同步**：

```typescript
// 关键代码位置：web/src/hooks/useLiveMemoRefresh.ts:64-104
// 只处理备忘录变更事件，不处理设置变更
```

### 5.4 旧页面状态处理的限制

**当前实现的特点：**

1. **单页面内即时响应**：
   - 设置页面内修改后，同一页面的其他组件会立即响应
   - 基于 React Context 的响应式更新

2. **跨标签页/跨会话不实时同步**：
   - 没有 SSE 事件推送设置变更
   - 其他标签页需要刷新或重新初始化才能看到新设置

3. **初始化时加载**：
   - 页面加载时调用 `InstanceContext.initialize()` 获取最新设置
   - 登录后调用 `AuthContext.initialize()` 获取用户设置

4. **敏感设置的服务端保护**（`server/router/api/v1/instance_service.go:128-165`）：

```go
// Storage 和 Notification 设置包含凭据，只允许管理员访问
if instanceSetting.Key == storepb.InstanceSettingKey_STORAGE ||
    instanceSetting.Key == storepb.InstanceSettingKey_NOTIFICATION {
    user, err := caller.currentUser(ctx, s)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get current user: %v", err)
    }
    if user == nil {
        return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
    }
    if user.Role != store.RoleAdmin {
        return nil, status.Errorf(codes.PermissionDenied, "permission denied")
    }
}

// AI 设置对非管理员部分脱敏
if instanceSetting.Key == storepb.InstanceSettingKey_AI && !isAdminCaller {
    if ai := result.GetAiSetting(); ai != nil && ai.Transcription != nil {
        ai.Transcription.Model = ""
        ai.Transcription.Language = ""
        ai.Transcription.Prompt = ""
    }
}
```

---

## 六、架构总结

### 6.1 安全边界层次

| 层次 | 实现位置 | 作用 |
|-----|---------|------|
| 服务端 API 验证 | `server/router/api/v1/*.go` | 最终安全边界，不可绕过 |
| 前端条件渲染 | `web/src/components/*` | 用户体验优化，减少无效请求 |
| 路由守卫 | `web/src/router/guards.tsx` | 页面级访问控制 |
| Context 状态 | `web/src/contexts/*` | 前端状态管理 |

### 6.2 设置数据流

```
用户修改设置
    ↓
前端 Context.updateSetting()
    ↓
API 调用 instanceServiceClient.updateInstanceSetting()
    ↓
服务端 Store.UpsertInstanceSetting()
    ↓
数据库存储 + 缓存更新
    ↓
返回新设置
    ↓
前端 Context.setState()
    ↓
所有使用 useInstance() 的组件响应式更新
```

### 6.3 关键设计决策

1. **双层验证**：前端隐藏 + 服务端强制，确保安全性
2. **默认值处理**：未设置时返回合理默认值，避免空值问题
3. **敏感信息保护**：凭据类设置（密码、API Key）不返回给前端
4. **Context 模式**：使用 React Context 实现全局状态共享和响应式更新
5. **缓存策略**：服务端缓存减少数据库查询，提高性能
