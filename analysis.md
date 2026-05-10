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

### 2.1 两种注册路径对比

Memos 支持两种创建用户账户的方式：

| 路径 | 触发方式 | 创建用户方式 | 用户名来源 |
|-----|---------|-------------|-----------|
| **密码注册** | 用户主动访问 `/auth/signup` | `CreateUser` API | 用户手动输入 |
| **SSO 首次登录** | 用户通过 OAuth2 登录 | `resolveSSOUser` 内部创建 | 自动生成 UUID 格式 |

---

### 2.2 密码注册路径完整分析

#### 2.2.1 前端入口控制（`web/src/pages/SignUp.tsx`）

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

**前端显示条件：**
- `disallowUserRegistration = false`（允许用户注册）
- `disallowPasswordAuth = false`（允许密码认证）

#### 2.2.2 服务端强制验证（`server/router/api/v1/user_service.go:145-238`）

```go
// CreateUser API 中的验证逻辑
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

**服务端验证条件：**
- 第一个用户创建为 Admin，**不受限制**
- 普通用户注册时必须同时满足：
  - `DisallowUserRegistration = false`
  - `DisallowPasswordAuth = false`

#### 2.2.3 密码注册受约束的全局开关

| 开关名称 | 类型 | 作用 | 密码注册时的影响 |
|---------|------|------|----------------|
| `DisallowUserRegistration` | 实例设置 | 是否允许用户注册 | 为 `true` 时完全禁止创建非 Admin 用户 |
| `DisallowPasswordAuth` | 实例设置 | 是否允许密码认证 | 为 `true` 时禁止使用密码注册和登录 |

---

### 2.3 SSO 首次登录创建账号路径完整分析

#### 2.3.1 前端登录入口（`web/src/pages/SignIn.tsx`）

```typescript
// 关键代码位置：web/src/pages/SignIn.tsx:27-33
useEffect(() => {
  const fetchIdentityProviderList = async () => {
    const { identityProviders } = await identityProviderServiceClient.listIdentityProviders({});
    setIdentityProviderList(identityProviders);
  };
  fetchIdentityProviderList();
}, []);

// 关键代码位置：web/src/pages/SignIn.tsx:80-92
{!instanceGeneralSetting.disallowPasswordAuth ? (
  <PasswordSignInForm redirectPath={redirectTarget} />
) : (
  identityProviderList.length === 0 && <p className="w-full text-2xl mt-2 text-muted-foreground">Password auth is not allowed.</p>
)}

// 关键代码位置：web/src/pages/SignIn.tsx:85-92
{!instanceGeneralSetting.disallowUserRegistration && !instanceGeneralSetting.disallowPasswordAuth && (
  <p className="w-full mt-4 text-sm">
    <span className="text-muted-foreground">{t("auth.sign-up-tip")}</span>
    <Link to={signUpPath} className="cursor-pointer ml-2 text-primary hover:underline" viewTransition>
      {t("common.sign-up")}
    </Link>
  </p>
)}
```

**前端显示特点：**
- SSO 登录按钮始终显示（只要配置了 IDP）
- 密码登录表单受 `disallowPasswordAuth` 控制
- 注册链接受 `disallowUserRegistration` 和 `disallowPasswordAuth` 双重控制

#### 2.3.2 服务端 SSO 处理流程（`server/router/api/v1/auth_service.go`）

整个流程分为两步：

**第一步：身份验证（`auth_service.go:91-101`）**
```go
} else if ssoCredentials := request.GetSsoCredentials(); ssoCredentials != nil {
    // 1. 解析 OAuth2 回调，获取用户信息
    identityProvider, userInfo, err := s.resolveSSOIdentity(ctx, ssoCredentials.IdpName, ssoCredentials.Code, ssoCredentials.RedirectUri, ssoCredentials.CodeVerifier)
    if err != nil {
        return nil, err
    }
    // 2. 查找或创建本地用户
    user, err := s.resolveSSOUser(ctx, nil, identityProvider, userInfo)
    if err != nil {
        return nil, err
    }
    existingUser = user
}
```

**第二步：首次登录创建用户（`auth_service.go:134-217`）**

```go
func (s *APIV1Service) resolveSSOUser(ctx context.Context, currentUser *store.User, identityProvider *storepb.IdentityProvider, userInfo *idp.IdentityProviderUserInfo) (*store.User, error) {
    // 1. 检查是否已有本地用户与该 SSO 账号关联
    user, err := s.getLinkedSSOUser(ctx, provider, externUID)
    if err != nil {
        return nil, err
    }
    if user != nil {
        // 已存在关联，直接返回
        return user, nil
    }

    // 2. 首次登录：强制检查注册开关
    instanceGeneralSetting, err := s.Store.GetInstanceGeneralSetting(ctx)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get instance general setting, error: %v", err)
    }
    if instanceGeneralSetting.DisallowUserRegistration {
        return nil, status.Errorf(codes.PermissionDenied, "user registration is not allowed")
    }

    // 3. 自动创建本地用户
    password, err := util.RandomString(20)  // 生成随机密码
    passwordHash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    username, err := deriveSSOUsername()    // 生成 UUID 格式用户名
    user, err = s.Store.CreateUser(ctx, &store.User{
        Username:     username,             // UUID，如 "a1b2c3d4"
        Role:         store.RoleUser,
        Nickname:     userInfo.DisplayName, // 来自 SSO
        Email:        userInfo.Email,       // 来自 SSO
        AvatarURL:    userInfo.AvatarURL,   // 来自 SSO
        PasswordHash: string(passwordHash), // 随机密码，用户不知道
    })

    // 4. 创建用户身份关联记录
    if _, err := s.Store.CreateUserIdentity(ctx, &store.UserIdentity{
        UserID:    user.ID,
        Provider:  provider,
        ExternUID: externUID,
    }); err != nil {
        // 并发处理：如果另一个请求先创建了，加载获胜的用户
        // ...
    }
    return user, nil
}
```

#### 2.3.3 SSO 首次登录受约束的全局开关

| 开关名称 | 类型 | 作用 | SSO 首次登录时的影响 |
|---------|------|------|---------------------|
| `DisallowUserRegistration` | 实例设置 | 是否允许用户注册 | **为 `true` 时禁止** |
| `DisallowPasswordAuth` | 实例设置 | 是否允许密码认证 | **不影响** |

**关键点：**
- SSO 首次登录**不受 `DisallowPasswordAuth` 影响**
- SSO 首次登录**受 `DisallowUserRegistration` 影响**
- SSO 创建的用户有随机密码，但用户无法通过密码登录（除非后来启用了密码认证且用户设置了密码）

#### 2.3.4 密码登录的额外限制（`auth_service.go:87-89`）

```go
// 关键代码位置：server/router/api/v1/auth_service.go:87-89
if instanceGeneralSetting.DisallowPasswordAuth && user.Role == store.RoleUser {
    return nil, status.Errorf(codes.PermissionDenied, "password signin is not allowed")
}
```

**注意：**
- 密码登录时，**Admin 用户不受 `DisallowPasswordAuth` 限制**
- 只有普通用户（`RoleUser`）被禁止使用密码登录

---

### 2.4 两种路径对比总结

| 维度 | 密码注册 | SSO 首次登录创建 |
|-----|---------|-----------------|
| **触发入口** | `/auth/signup` 页面 | `/auth/signin` 页面点击 SSO 按钮 |
| **主动/被动** | 用户主动注册 | 用户被动创建（首次登录时） |
| **用户名** | 用户输入 | 自动生成 UUID |
| **密码** | 用户设置 | 系统随机生成（用户未知） |
| **受 `DisallowUserRegistration`** | ✅ 是 | ✅ 是 |
| **受 `DisallowPasswordAuth`** | ✅ 是 | ❌ 否 |
| **Admin 豁免** | ✅ 是（第一个用户） | ❌ 否（SSO 用户总是普通用户） |
| **服务端验证位置** | `user_service.go:195-207` | `auth_service.go:158-160` |
| **创建用户方式** | `Store.CreateUser()` 直接调用 | `Store.CreateUser()` 在 `resolveSSOUser` 中调用 |
| **并发安全** | 依赖数据库唯一约束（用户名） | 有完整的并发处理逻辑（检查冲突 → 加载获胜者） |

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

### 5.1 设置更新流程

#### 5.1.1 实例设置更新（`useInstanceSettingUpdater.ts`）

```typescript
// 关键代码位置：web/src/components/Settings/useInstanceSettingUpdater.ts:16-34
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

**Context 状态同步（`InstanceContext.tsx:192-198`）：**

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

---

### 5.2 设置变更后的状态差异分析

设置变更后，不同场景下的状态表现不同：

#### 5.2.1 已打开页面的状态

**同标签页内：**

| 场景 | 行为 | 原因 |
|-----|------|------|
| 修改设置的页面 | 即时更新 | React Context 响应式更新 |
| 已渲染但未重新挂载的组件 | 可能滞后 | 依赖 useMemo/useEffect 的依赖数组 |

**关键代码（编辑器组件的可见性设置更新）：**

```typescript
// 关键代码位置：web/src/components/MemoEditor/index.tsx:279-280
useEffect(() => {
  if (!memoName && defaultVisibility) {
    dispatch(actions.setMetadata({ visibility: defaultVisibility }));
  }
}, [dispatch, defaultVisibility, memoName]);
```

**注意：** 这个 effect 只在 `defaultVisibility` 变化且 `memoName` 为空时触发。如果用户已经手动修改了可见性，设置变更**不会覆盖**用户的手动选择。

**编辑器中的行为总结：**
- **新建备忘录（未开始编辑）**：设置变更后，默认可见性会更新
- **新建备忘录（已开始编辑）**：如果用户已手动选择可见性，设置变更不会覆盖
- **编辑已有备忘录**：使用备忘录自身的可见性，不受默认可见性设置影响

---

#### 5.2.2 草稿缓存的状态

**草稿缓存的存储机制（`cacheService.ts`）：**

```typescript
// 关键代码位置：web/src/components/MemoEditor/services/cacheService.ts:32-76
export const cacheService = {
  // 只保存 content，不保存 visibility
  save: (key: string, content: string) => {
    // ...
  },
  load(key: string): string {
    const raw = localStorage.getItem(key);
    return raw ? deserializeContent(raw) : "";  // 只返回 content
  },
  // ...
};
```

**草稿加载时的行为（`useMemoInit.ts:40-46`）：**

```typescript
const cachedContent = cacheService.load(key);
if (cachedContent) {
  dispatch(actions.updateContent(cachedContent));  // 只恢复内容
}
if (defaultVisibility !== undefined) {
  dispatch(actions.setMetadata({ visibility: defaultVisibility }));  // 使用当前默认可见性
}
```

**草稿缓存与设置变更的关系：**

| 维度 | 行为 |
|-----|------|
| 草稿内容 | 保存在 localStorage，不受设置变更影响 |
| 草稿可见性 | **不保存**，每次重新加载时使用当前默认设置 |
| 已保存的备忘录 | 可见性已落库，不受设置变更影响 |

**场景示例：**

```
场景：默认可见性从 PRIVATE 改为 PUBLIC

时间线：
1. T=0: 默认可见性 = PRIVATE
2. T=1: 用户开始写草稿，保存为草稿（只保存内容）
3. T=2: 用户修改默认可见性为 PUBLIC
4. T=3: 用户重新打开编辑器

结果：
- 草稿内容：恢复 T=1 时的内容
- 草稿可见性：使用 T=3 的默认值 PUBLIC（不是 T=1 的 PRIVATE）
```

---

#### 5.2.3 跨标签页的状态差异

**Token 同步机制（`auth-state.ts`）：**

```typescript
// 关键代码位置：web/src/auth-state.ts:9-46
const TOKEN_CHANNEL_NAME = "memos_token_sync";

let tokenChannel: BroadcastChannel | null = null;

function getTokenChannel(): BroadcastChannel | null {
  if (tokenChannel) return tokenChannel;
  try {
    tokenChannel = new BroadcastChannel(TOKEN_CHANNEL_NAME);
    tokenChannel.onmessage = (event: MessageEvent<TokenBroadcastMessage>) => {
      const { token, expiresAt } = event.data ?? {};
      if (token && expiresAt) {
        accessToken = token;
        tokenExpiresAt = new Date(expiresAt);
      }
    };
  } catch {
    tokenChannel = null;
  }
  return tokenChannel;
}
```

**关键发现：**
- `BroadcastChannel` 只用于**token 同步**，**不用于设置同步**
- 设置变更不会广播到其他标签页

**跨标签页的状态表现：**

| 标签页 | 状态 | 原因 |
|-------|------|------|
| 修改设置的标签页 | 即时更新 | Context 响应式更新 |
| 其他已打开的标签页 | 保持旧值 | 没有 SSE/Broadcast 推送 |
| 新打开的标签页 | 使用新值 | 初始化时从服务器获取 |

**跨标签页同步机制对比：**

| 同步对象 | 机制 | 是否跨标签页 |
|---------|------|-------------|
| Access Token | BroadcastChannel | ✅ 是 |
| 实例设置 | React Context | ❌ 否 |
| 用户设置 | React Context | ❌ 否 |
| 备忘录数据 | SSE (useLiveMemoRefresh) | ✅ 是 |

---

### 5.3 实时动态更新的设置

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

**实时响应的设置：**
- `additionalStyle`：立即注入 `<style>` 标签
- `additionalScript`：立即注入 `<script>` 标签
- `customProfile.title`：立即更新 `document.title`
- `customProfile.logoUrl`：立即更新 favicon

---

### 5.4 SSE 实时刷新机制

Memos 使用 SSE (Server-Sent Events) 实现备忘录的实时更新，但**不用于实例设置的实时同步**：

```typescript
// 关键代码位置：web/src/hooks/useLiveMemoRefresh.ts:64-104
// 只处理备忘录变更事件，不处理设置变更
```

---

### 5.5 敏感设置的服务端保护

即使前端绕过，服务端也会保护敏感设置（`server/router/api/v1/instance_service.go:128-165`）：

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

### 5.6 设置变更后状态差异总结

| 场景 | 已打开页面 | 草稿缓存 | 跨标签页 |
|-----|-----------|---------|---------|
| **同标签页内** | ✅ 即时更新 | 不影响（只存内容） | N/A |
| **用户手动修改过** | ❌ 不覆盖 | 不保存 | N/A |
| **草稿重新打开** | N/A | 使用新默认值 | N/A |
| **其他标签页** | N/A | N/A | ❌ 保持旧值 |
| **新标签页** | ✅ 使用新值 | ✅ 使用新值 | ✅ 使用新值 |

**刷新时机：**
- 页面刷新：重新初始化，获取最新设置
- 重新登录：重新初始化用户设置
- 导航切换：如果组件重新挂载，会使用最新 Context 值

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
