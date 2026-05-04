# OAuth SSO 绑定流程分析

本文档分析 Memos 系统中外部身份提供方（OAuth2）登录后，用户创建/绑定、身份信息保存、登录入口开关控制以及错误回调处理的完整流程。

---

## 目录

1. [整体架构概览](#整体架构概览)
2. [前端登录入口与流程启动](#前端登录入口与流程启动)
3. [OAuth 回调与错误处理](#oauth-回调与错误处理)
4. [后端身份解析与用户绑定](#后端身份解析与用户绑定)
5. [登录入口开关控制](#登录入口开关控制)
6. [身份信息存储结构](#身份信息存储结构)
7. [关键数据流程图](#关键数据流程图)

---

## 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              OAuth SSO 完整流程                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐      ┌──────────────┐      ┌────────────────────────────┐   │
│  │  前端   │ ───> │ OAuth 授权页  │ ───> │  /auth/callback 回调处理  │   │
│  │ SignIn  │      │ (第三方 IdP)  │      │      (AuthCallback.tsx)   │   │
│  └──────────┘      └──────────────┘      └────────────────────────────┘   │
│                                                          │                    │
│                                                          ▼                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         后端 API 层 (auth_service.go)                 │  │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐  │  │
│  │  │ resolveSSOUser  │    │ resolveSSOIdent │    │   doSignIn      │  │  │
│  │  │  用户绑定/创建   │    │   身份信息解析   │    │  生成 Token     │  │  │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                          │                    │
│                                                          ▼                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         存储层 (store)                                  │  │
│  │  ┌─────────────────┐    ┌─────────────────────────────────────────┐  │  │
│  │  │   User 表       │    │         UserIdentity 关联表               │  │  │
│  │  │  (本地用户信息)  │    │  (Provider + ExternUID -> UserID 映射)  │  │  │
│  │  └─────────────────┘    └─────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 前端登录入口与流程启动

### 1. 登录页面组件

**文件位置**: `web/src/pages/SignIn.tsx`

登录页面通过以下步骤启动 OAuth 流程：

```typescript
// 1. 获取已配置的身份提供方列表
useEffect(() => {
  const fetchIdentityProviderList = async () => {
    const { identityProviders } = await identityProviderServiceClient.listIdentityProviders({});
    setIdentityProviderList(identityProviders);
  };
  fetchIdentityProviderList();
}, []);

// 2. 用户点击 SSO 登录按钮时触发
const handleSignInWithIdentityProvider = async (identityProvider: IdentityProvider) => {
  if (identityProvider.type === IdentityProvider_Type.OAUTH2) {
    const redirectUri = absolutifyLink("/auth/callback");
    
    // 生成安全 state 参数 (CSRF 保护) 和 PKCE 参数
    const { state, codeChallenge } = await storeOAuthState(
      identityProvider.name, 
      "signin", 
      redirectTarget
    );

    // 构建 OAuth 授权 URL
    let authUrl = `${oauth2Config.authUrl}?client_id=${...}&state=${state}&...`;
    
    // 添加 PKCE 参数 (如果可用)
    if (codeChallenge) {
      authUrl += `&code_challenge=${codeChallenge}&code_challenge_method=S256`;
    }

    // 跳转到第三方授权页面
    window.location.href = authUrl;
  }
};
```

### 2. OAuth State 管理

**文件位置**: `web/src/utils/oauth.ts`

系统使用 `sessionStorage` 存储 OAuth 状态，实现 CSRF 保护和 PKCE 支持：

```typescript
const STATE_STORAGE_KEY = "oauth_state";
const STATE_EXPIRY_MS = 10 * 60 * 1000; // 10 分钟过期

interface OAuthState {
  state: string;                    // 随机状态值 (CSRF 保护)
  identityProviderName: string;     // 身份提供方标识
  flowMode: OAuthFlowMode;          // "signin" | "link"
  timestamp: number;                // 创建时间戳
  returnUrl?: string;               // 登录后跳转 URL
  linkingUserName?: string;         // 账号绑定时的当前用户名
  codeVerifier?: string;            // PKCE 验证器
}

// 生成安全的随机 state
function generateSecureState(): string {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return Array.from(array, (byte) => byte.toString(16).padStart(2, "0")).join("");
}

// 验证 state 并返回存储的上下文信息
export function validateOAuthState(stateParam: string): { 
  identityProviderName: string; 
  flowMode: OAuthFlowMode; 
  returnUrl?: string; 
  linkingUserName?: string; 
  codeVerifier?: string 
} | null {
  // 1. 检查 state 是否已过期 (10 分钟)
  // 2. 验证 state 参数匹配 (CSRF 保护)
  // 3. 清理存储并返回上下文数据
}
```

### 3. PKCE 支持 (RFC 7636)

```typescript
// 生成 code_verifier (43-128 字符的 URL-safe base64)
function generateCodeVerifier(): string {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

// 生成 code_challenge = SHA256(code_verifier)
async function generateCodeChallenge(codeVerifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(codeVerifier);
  const hash = await crypto.subtle.digest("SHA-256", data);
  return base64UrlEncode(new Uint8Array(hash));
}
```

**注意**: PKCE 仅在 HTTPS 或 localhost 环境下可用（需要 `crypto.subtle`）。

---

## OAuth 回调与错误处理

### 1. 回调页面组件

**文件位置**: `web/src/pages/AuthCallback.tsx`

回调页面负责处理 OAuth 授权服务器返回的响应，包括成功和错误场景：

```typescript
const AuthCallback = () => {
  useEffect(() => {
    // ========== 错误回调处理 ==========
    
    // 1. 检查 OAuth 错误参数 (e.g., user denied access)
    const error = searchParams.get("error");
    const errorDescription = searchParams.get("error_description");
    const errorUri = searchParams.get("error_uri");

    if (error) {
      let errorMessage = `OAuth error: ${error}`;
      if (errorDescription) {
        errorMessage += `\n${decodeURIComponent(errorDescription)}`;
      }
      if (errorUri) {
        errorMessage += `\nMore info: ${errorUri}`;
      }
      setState({ loading: false, errorMessage });
      return;
    }

    // 2. 检查必需参数
    const code = searchParams.get("code");
    const state = searchParams.get("state");

    if (!code || !state) {
      setState({
        loading: false,
        errorMessage: "Failed to authorize. Missing authorization code or state parameter.",
      });
      return;
    }

    // 3. 验证 OAuth state (CSRF 保护)
    const validatedState = validateOAuthState(state);
    if (!validatedState) {
      setState({
        loading: false,
        errorMessage: "Failed to authorize. Invalid or expired state parameter. " +
                      "This may indicate a CSRF attack attempt.",
      });
      return;
    }

    // ========== 成功回调处理 ==========
    
    const { flowMode, identityProviderName, returnUrl, linkingUserName, codeVerifier } = validatedState;
    const redirectUri = absolutifyLink("/auth/callback");

    (async () => {
      try {
        if (flowMode === "link") {
          // 账号绑定模式: 绑定到当前登录用户
          await userServiceClient.createLinkedIdentity({
            parent: currentUser.name,
            idpName: identityProviderName,
            code,
            redirectUri,
            codeVerifier: codeVerifier || "",
          });
        } else {
          // 登录模式: 调用后端 SignIn API
          const response = await authServiceClient.signIn({
            credentials: {
              case: "ssoCredentials",
              value: {
                idpName: identityProviderName,
                code,
                redirectUri,
                codeVerifier: codeVerifier || "",
              },
            },
          });
          
          // 存储 access token
          if (response.accessToken) {
            setAccessToken(response.accessToken, response.accessTokenExpiresAt);
          }
        }
        
        // 初始化用户状态并跳转
        await initialize();
        navigateTo(getSafeRedirectPath(returnUrl) ?? ROUTES.HOME);
        
      } catch (error: unknown) {
        // 处理后端返回的错误
        handleError(error, () => {}, {
          fallbackMessage: "Failed to authenticate.",
          onError: (err) => {
            const message = err instanceof Error ? err.message : "Failed to authenticate.";
            setState({ loading: false, errorMessage: message });
          },
        });
      }
    })();
  }, [...]);
};
```

### 2. 错误回调场景汇总

#### 2.1 通用错误 (登录模式和绑定模式共有)

| 错误类型 | 触发条件 | 错误信息示例 |
|---------|---------|-------------|
| **OAuth 授权错误** | 用户拒绝授权、权限不足等 | `OAuth error: access_denied\nUser denied access` |
| **参数缺失** | 回调 URL 缺少 `code` 或 `state` | `Missing authorization code or state parameter` |
| **State 无效/过期** | state 不匹配或超过 10 分钟 | `Invalid or expired state parameter. This may indicate a CSRF attack attempt` |
| **后端验证失败** | 令牌交换失败、用户信息获取失败 | `Failed to exchange token` / `Failed to get user info` |

#### 2.2 登录模式特有错误

| 错误类型 | 触发条件 | 错误信息示例 |
|---------|---------|-------------|
| **注册被禁用** | 首次登录但 `DisallowUserRegistration=true` | `user registration is not allowed` |
| **身份已绑定他人** | 并发竞争或身份已绑定到其他用户 | `identity provider account is already linked to another user` |

#### 2.3 绑定模式特有错误 (用户状态变化检测)

**文件位置**: `web/src/pages/AuthCallback.tsx:85-91`

| 错误类型 | 触发条件 | 错误信息示例 | 设计目的 |
|---------|---------|-------------|---------|
| **当前用户未登录** | 发起绑定后，在 OAuth 跳转期间用户登录状态丢失 (如 token 过期、手动登出) | `Failed to link account. Please sign in to Memos again and retry.` | 防止绑定操作在无用户上下文时执行 |
| **用户身份已切换** | 发起绑定后，在 OAuth 跳转期间用户切换了账号 (如在另一标签页登出后用其他账号登录) | `The signed-in user changed before the OAuth callback completed. Please retry linking from account settings.` | **关键安全检查**: 防止外部身份被绑定到错误的用户账号 |

#### 2.4 绑定模式错误场景详解

**场景 1: 用户在 OAuth 跳转期间登出**
```
时间线:
1. 用户 A 登录系统，进入设置页面
2. 用户 A 点击 "绑定 GitHub 账号"，系统生成 state 并存入 sessionStorage
   state 中包含: linkingUserName = "users/123" (用户 A 的标识)
3. 页面跳转到 GitHub 授权页面
4. 用户 A 在另一标签页手动登出系统 (或 token 过期)
5. 用户 A 在 GitHub 完成授权，回调到 /auth/callback
6. 回调页面检查: currentUser?.name → undefined (用户未登录)
7. 抛出错误: "Failed to link account. Please sign in to Memos again and retry."
```

**场景 2: 用户在 OAuth 跳转期间切换账号**
```
时间线:
1. 用户 A 登录系统，进入设置页面
2. 用户 A 点击 "绑定 GitHub 账号"，系统生成 state
   state 中包含: linkingUserName = "users/123" (用户 A 的标识)
3. 页面跳转到 GitHub 授权页面
4. 用户 A 在另一标签页登出，然后用用户 B 的账号重新登录
5. 用户 A 在 GitHub 完成授权，回调到 /auth/callback
6. 回调页面检查:
   - currentUser.name = "users/456" (用户 B 的标识)
   - linkingUserName = "users/123" (发起绑定时的用户 A)
   - 检查: linkingUserName && currentUser.name !== linkingUserName → true
7. 抛出错误: "The signed-in user changed before the OAuth callback completed..."
```

**设计意图**:
- OAuth 流程是跨站点的，用户可能在跳转期间在其他标签页操作
- 如果不做此检查，用户 A 的 GitHub 身份可能会被错误地绑定到用户 B
- 这是一个**安全防护机制**，确保绑定操作的用户一致性

---

## 后端身份解析与用户绑定

### 1. SignIn 入口

**文件位置**: `server/router/api/v1/auth_service.go:64-121`

`SignIn` 方法支持两种认证方式：密码认证和 SSO 认证。

```go
func (s *APIV1Service) SignIn(ctx context.Context, request *v1pb.SignInRequest) (*v1pb.SignInResponse, error) {
    var existingUser *store.User

    // 方式 1: 密码认证
    if passwordCredentials := request.GetPasswordCredentials(); passwordCredentials != nil {
        // ... 密码验证逻辑
    } else if ssoCredentials := request.GetSsoCredentials(); ssoCredentials != nil {
        // 方式 2: SSO (OAuth2) 认证
        
        // 步骤 A: 解析外部身份
        identityProvider, userInfo, err := s.resolveSSOIdentity(
            ctx, 
            ssoCredentials.IdpName, 
            ssoCredentials.Code, 
            ssoCredentials.RedirectUri, 
            ssoCredentials.CodeVerifier
        )
        if err != nil {
            return nil, err
        }
        
        // 步骤 B: 绑定/创建本地用户
        user, err := s.resolveSSOUser(ctx, nil, identityProvider, userInfo)
        if err != nil {
            return nil, err
        }
        existingUser = user
    }

    // 生成 Token 并返回
    accessToken, accessExpiresAt, err := s.doSignIn(ctx, existingUser)
    // ...
}
```

### 2. 外部身份解析 (resolveSSOIdentity)

**文件位置**: `server/router/api/v1/auth_service.go:219-263`

```go
func (s *APIV1Service) resolveSSOIdentity(
    ctx context.Context, 
    idpName, code, redirectURI, codeVerifier string,
) (*storepb.IdentityProvider, *idp.IdentityProviderUserInfo, error) {
    
    // 1. 从名称提取 IdP UID 并查找配置
    idpUID, err := ExtractIdentityProviderUIDFromName(idpName)
    identityProvider, err := s.Store.GetIdentityProvider(ctx, &store.FindIdentityProvider{
        UID: &idpUID,
    })
    if identityProvider == nil {
        return nil, nil, status.Errorf(codes.InvalidArgument, "identity provider not found")
    }

    // 2. 处理 OAuth2 类型
    var userInfo *idp.IdentityProviderUserInfo
    if identityProvider.Type == storepb.IdentityProvider_OAUTH2 {
        // 创建 OAuth2 客户端
        oauth2IdentityProvider, err := oauth2.NewIdentityProvider(
            identityProvider.Config.GetOauth2Config()
        )
        
        // 2a. 交换授权码获取 Access Token
        // 支持 PKCE: codeVerifier 用于验证 code_challenge
        token, err := oauth2IdentityProvider.ExchangeToken(
            ctx, redirectURI, code, codeVerifier
        )
        
        // 2b. 使用 Access Token 获取用户信息
        userInfo, err = oauth2IdentityProvider.UserInfo(ctx, token)
    }

    // 3. 应用标识符过滤器 (IdentifierFilter)
    // 允许管理员限制哪些外部用户可以登录
    identifierFilter := identityProvider.IdentifierFilter
    if identifierFilter != "" {
        identifierFilterRegex, err := regexp.Compile(identifierFilter)
        if !identifierFilterRegex.MatchString(userInfo.Identifier) {
            return nil, nil, status.Errorf(
                codes.PermissionDenied, 
                "identifier %s is not allowed", 
                userInfo.Identifier
            )
        }
    }

    return identityProvider, userInfo, nil
}
```

### 3. OAuth2 令牌交换与用户信息获取

**文件位置**: `internal/idp/oauth2/oauth2.go`

#### 3.1 令牌交换 (ExchangeToken)

```go
func (p *IdentityProvider) ExchangeToken(
    ctx context.Context, 
    redirectURL, code, codeVerifier string,
) (string, error) {
    conf := &oauth2.Config{
        ClientID:     p.config.ClientId,
        ClientSecret: p.config.ClientSecret,
        RedirectURL:  redirectURL,
        Scopes:       p.config.Scopes,
        Endpoint: oauth2.Endpoint{
            AuthURL:   p.config.AuthUrl,
            TokenURL:  p.config.TokenUrl,
            AuthStyle: oauth2.AuthStyleInParams,
        },
    }

    // PKCE 支持: 添加 code_verifier
    opts := []oauth2.AuthCodeOption{}
    if codeVerifier != "" {
        opts = append(opts, oauth2.SetAuthURLParam("code_verifier", codeVerifier))
    }

    // 执行令牌交换
    token, err := conf.Exchange(ctx, code, opts...)
    if err != nil {
        return "", errors.Wrap(err, "failed to exchange access token")
    }

    if token.AccessToken == "" {
        return "", errors.New("missing access token from authorization response")
    }

    return token.AccessToken, nil
}
```

#### 3.2 用户信息获取 (UserInfo)

```go
func (p *IdentityProvider) UserInfo(
    ctx context.Context, 
    token string,
) (*idp.IdentityProviderUserInfo, error) {
    // 1. 发送请求到 UserInfo 端点
    client := &http.Client{Timeout: userInfoRequestTimeout}
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, p.config.UserInfoUrl, nil)
    req.Header.Set("Authorization", fmt.Sprintf("Bearer %s", token))
    resp, err := client.Do(req)
    
    // 2. 错误响应处理
    if resp.StatusCode < http.StatusOK || resp.StatusCode >= http.StatusMultipleChoices {
        body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
        return nil, errors.Errorf(
            "userinfo request failed with status %d: %s", 
            resp.StatusCode, string(body)
        )
    }

    // 3. 解析 JSON 响应
    body, _ := io.ReadAll(resp.Body)
    var claims map[string]any
    json.Unmarshal(body, &claims)

    // 4. 字段映射 (根据配置的 FieldMapping)
    userInfo := &idp.IdentityProviderUserInfo{}
    
    // 必需字段: Identifier
    if v, ok := claims[p.config.FieldMapping.Identifier].(string); ok {
        userInfo.Identifier = v
    }
    if userInfo.Identifier == "" {
        return nil, errors.Errorf(
            "the field %q is not found in claims or has empty value", 
            p.config.FieldMapping.Identifier
        )
    }

    // 可选字段: DisplayName, Email, AvatarURL
    if p.config.FieldMapping.DisplayName != "" {
        if v, ok := claims[p.config.FieldMapping.DisplayName].(string); ok {
            userInfo.DisplayName = v
        }
    }
    if userInfo.DisplayName == "" {
        userInfo.DisplayName = userInfo.Identifier // 兜底使用 Identifier
    }
    // ... Email, AvatarURL 类似处理

    return userInfo, nil
}
```

### 4. 用户绑定/创建 (resolveSSOUser)

**文件位置**: `server/router/api/v1/auth_service.go:123-217`

这是整个流程的核心函数，处理用户查找、创建和绑定逻辑。

```go
func (s *APIV1Service) resolveSSOUser(
    ctx context.Context, 
    currentUser *store.User,           // 绑定时传入的当前用户，登录时为 nil
    identityProvider *storepb.IdentityProvider, 
    userInfo *idp.IdentityProviderUserInfo,
) (*store.User, error) {
    
    provider := identityProvider.Uid
    externUID := userInfo.Identifier

    // ========== 阶段 1: 查找已绑定的用户 ==========
    user, err := s.getLinkedSSOUser(ctx, provider, externUID)
    if err != nil {
        return nil, err
    }
    
    // 如果已存在绑定
    if user != nil {
        // 绑定模式下: 检查是否绑定到同一用户
        if currentUser != nil && currentUser.ID != user.ID {
            return nil, status.Errorf(
                codes.AlreadyExists, 
                "identity provider account is already linked to another user"
            )
        }
        return user, nil
    }

    // ========== 阶段 2: 绑定到现有用户 (绑定模式) ==========
    if currentUser != nil {
        return s.bindSSOIdentityToUser(ctx, currentUser, provider, externUID)
    }

    // ========== 阶段 3: 创建新用户 (首次登录) ==========
    
    // 3.1 检查注册是否被禁用
    instanceGeneralSetting, err := s.Store.GetInstanceGeneralSetting(ctx)
    if instanceGeneralSetting.DisallowUserRegistration {
        return nil, status.Errorf(
            codes.PermissionDenied, 
            "user registration is not allowed"
        )
    }

    // 3.2 生成随机密码 (SSO 用户不需要密码登录)
    password, _ := util.RandomString(20)
    passwordHash, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    
    // 3.3 生成 UUID 格式的用户名
    username, err := deriveSSOUsername() // 生成如 "a1b2c3d4-..." 的用户名
    
    // 3.4 创建本地用户记录
    user, err = s.Store.CreateUser(ctx, &store.User{
        Username:     username,
        Role:         store.RoleUser,
        Nickname:     userInfo.DisplayName,  // 从外部身份获取
        Email:        userInfo.Email,        // 从外部身份获取
        AvatarURL:    userInfo.AvatarURL,    // 从外部身份获取
        PasswordHash: string(passwordHash),
    })

    // 3.5 创建身份绑定记录 (UserIdentity)
    if _, err := s.Store.CreateUserIdentity(ctx, &store.UserIdentity{
        UserID:    user.ID,
        Provider:  provider,
        ExternUID: externUID,
    }); err != nil {
        // 并发竞争处理: 如果 UNIQUE 约束冲突
        if isUniqueConstraintViolation(err) {
            // 查找并发创建的绑定记录
            winner, _ := s.Store.GetUserIdentity(ctx, &store.FindUserIdentity{
                Provider:  &provider,
                ExternUID: &externUID,
            })
            // 返回获胜者绑定的用户
            winnerUser, _ := s.Store.GetUser(ctx, &store.FindUser{ID: &winner.UserID})
            return winnerUser, nil
        }
        
        // 其他错误: 回滚用户创建
        _ = s.Store.DeleteUser(ctx, &store.DeleteUser{ID: user.ID})
        return nil, status.Errorf(
            codes.Internal, 
            "failed to create user identity, error: %v", err
        )
    }

    return user, nil
}
```

### 5. 绑定到现有用户 (bindSSOIdentityToUser)

**文件位置**: `server/router/api/v1/auth_service.go:286-337`

用于已登录用户绑定外部身份的场景：

```go
func (s *APIV1Service) bindSSOIdentityToUser(
    ctx context.Context, 
    currentUser *store.User, 
    provider, externUID string,
) (*store.User, error) {
    
    // 1. 检查该用户是否已绑定同一 Provider 的其他身份
    existingForProvider, err := s.Store.GetUserIdentity(ctx, &store.FindUserIdentity{
        UserID:   &currentUser.ID,
        Provider: &provider,
    })
    if existingForProvider != nil {
        // 同一身份: 幂等返回
        if existingForProvider.ExternUID == externUID {
            return currentUser, nil
        }
        // 不同身份: 报错
        return nil, status.Errorf(
            codes.AlreadyExists, 
            "identity provider is already linked to another external account for this user"
        )
    }

    // 2. 创建绑定记录
    if _, err := s.Store.CreateUserIdentity(ctx, &store.UserIdentity{
        UserID:    currentUser.ID,
        Provider:  provider,
        ExternUID: externUID,
    }); err != nil {
        // 并发竞争处理
        if isUniqueConstraintViolation(err) {
            // 检查是否绑定到了当前用户
            winner, getErr := s.getLinkedSSOUser(ctx, provider, externUID)
            if winner != nil {
                if winner.ID != currentUser.ID {
                    return nil, status.Errorf(
                        codes.AlreadyExists, 
                        "identity provider account is already linked to another user"
                    )
                }
                return currentUser, nil
            }
            // ... 更多检查
        }
        return nil, err
    }

    return currentUser, nil
}
```

### 6. 并发竞争处理

**文件位置**: `server/router/api/v1/auth_service.go:339-353`

```go
func isUniqueConstraintViolation(err error) bool {
    if err == nil {
        return false
    }
    msg := err.Error()
    // 检查各数据库驱动的唯一约束错误信息
    return strings.Contains(msg, "UNIQUE constraint failed") ||  // SQLite
           strings.Contains(msg, "duplicate key") ||             // PostgreSQL
           strings.Contains(msg, "Duplicate entry")              // MySQL
}
```

**并发场景处理策略**:
1. **首次登录并发**: 两个请求同时用同一外部身份首次登录
   - 请求 A: 创建 User → 创建 UserIdentity (成功)
   - 请求 B: 创建 User → 创建 UserIdentity (UNIQUE 冲突)
   - 请求 B: 检测到冲突 → 查找请求 A 创建的 UserIdentity → 返回绑定的 User
   - 可选: 请求 B 回滚自己创建的临时 User

2. **绑定并发**: 两个用户同时尝试绑定同一外部身份
   - 先成功的用户获得绑定
   - 后尝试的用户收到 `identity provider account is already linked to another user`

---

## 登录入口开关控制

### 1. 实例级别配置

系统通过 `InstanceGeneralSetting` 控制登录入口：

```protobuf
// 来自 proto/gen/store/instance_setting.pb.go
message InstanceGeneralSetting {
    bool disallow_user_registration = 2;  // 禁用用户注册
    bool disallow_password_auth = 3;       // 禁用密码登录
    // ...
}
```

### 2. 前端开关逻辑

**文件位置**: `web/src/pages/SignIn.tsx:80-116`

```typescript
return (
  <div>
    {/* 密码登录表单 */}
    {!instanceGeneralSetting.disallowPasswordAuth ? (
      <PasswordSignInForm redirectPath={redirectTarget} />
    ) : (
      identityProviderList.length === 0 && 
      <p>Password auth is not allowed.</p>
    )}

    {/* 注册提示 */}
    {!instanceGeneralSetting.disallowUserRegistration && 
     !instanceGeneralSetting.disallowPasswordAuth && (
      <p>
        <span>Don't have an account?</span>
        <Link to={signUpPath}>Sign up</Link>
      </p>
    )}

    {/* SSO 登录按钮 (始终显示，只要有配置的 IdP) */}
    {identityProviderList.length > 0 && (
      <>
        {!instanceGeneralSetting.disallowPasswordAuth && <Separator />}
        {identityProviderList.map((identityProvider) => (
          <Button onClick={() => handleSignInWithIdentityProvider(identityProvider)}>
            Sign in with {identityProvider.title}
          </Button>
        ))}
      </>
    )}
  </div>
);
```

### 3. 后端开关逻辑

#### 3.1 密码登录检查 (管理员例外机制)

**文件位置**: `server/router/api/v1/auth_service.go:82-89`

```go
// 密码登录时检查
instanceGeneralSetting, err := s.Store.GetInstanceGeneralSetting(ctx)
if instanceGeneralSetting.DisallowPasswordAuth && user.Role == store.RoleUser {
    // 仅普通用户被禁止使用密码登录
    // 管理员 (RoleAdmin) 不受此限制，可以继续使用密码登录
    return nil, status.Errorf(
        codes.PermissionDenied, 
        "password signin is not allowed"
    )
}
```

**用户角色定义** (`store/user.go:11-16`):
```go
const (
    // RoleAdmin is the ADMIN role.
    RoleAdmin Role = "ADMIN"
    // RoleUser is the USER role.
    RoleUser Role = "USER"
)
```

**关键设计说明**:
- `DisallowPasswordAuth` 采用**条件限制**而非完全禁用
- 限制条件：`DisallowPasswordAuth && user.Role == RoleUser`
- 这意味着：
  - **普通用户 (USER)**: 当 `DisallowPasswordAuth=true` 时，无法使用密码登录
  - **管理员 (ADMIN)**: 无论 `DisallowPasswordAuth` 如何设置，始终可以使用密码登录
- **设计目的**: 防止管理员被锁定在系统外。即使配置了强制 SSO，管理员仍可通过密码登录进行紧急维护和配置调整。

**前端行为补充**:
- 当 `DisallowPasswordAuth=true` 时，前端登录页面会隐藏密码登录表单
- 但这只是 UI 层面的隐藏，后端仍保留管理员密码登录能力
- 管理员可以直接调用 API 或通过其他方式使用密码登录

#### 3.2 SSO 首次登录检查

**文件位置**: `server/router/api/v1/auth_service.go:153-160`

```go
// SSO 首次登录时检查注册是否允许
if instanceGeneralSetting.DisallowUserRegistration {
    return nil, status.Errorf(
        codes.PermissionDenied, 
        "user registration is not allowed"
    )
}
```

#### 3.3 开关组合场景 (统一结论)

| 配置组合 | 普通用户可用登录方式 | 管理员可用登录方式 | 统一结论说明 |
|---------|---------------------|-------------------|-------------|
| `DisallowPasswordAuth=false`<br>`DisallowUserRegistration=false` | 密码登录 + SSO 登录 + 用户注册 | 密码登录 + SSO 登录 + 用户注册 | **默认配置**<br>所有用户可自由选择登录方式 |
| `DisallowPasswordAuth=true`<br>`DisallowUserRegistration=false` | SSO 登录 + 新用户自动注册 | 密码登录 + SSO 登录 + 新用户自动注册 | **强制 SSO 模式**<br>普通用户：只能用 SSO，新用户首次登录自动创建账号<br>**管理员例外入口**：仍可使用密码登录，防止被锁定在系统外 |
| `DisallowPasswordAuth=false`<br>`DisallowUserRegistration=true` | 密码登录 (已有用户) + SSO 登录 (仅绑定用户) | 密码登录 (已有用户) + SSO 登录 (仅绑定用户) | **封闭注册模式**<br>禁止新用户注册<br>SSO 仅允许已绑定身份的用户登录<br>管理员和普通用户规则相同 |
| `DisallowPasswordAuth=true`<br>`DisallowUserRegistration=true` | SSO 登录 (仅绑定用户) | 密码登录 + SSO 登录 (仅绑定用户) | **最严格模式**<br>普通用户：仅已绑定 SSO 身份的用户可登录<br>**管理员例外入口**：仍可使用密码登录，作为紧急维护入口 |

---

## 身份信息存储结构

### 1. 核心数据模型

#### 1.1 UserIdentity (身份绑定表)

**文件位置**: `store/user_identity.go`

```go
type UserIdentity struct {
    ID        int32   // 内部主键
    UserID    int32   // 关联的本地用户 ID
    Provider  string  // 身份提供方 UID (如 "github", "google")
    ExternUID string  // 外部用户唯一标识 (来自 IdP 的 Identifier)
    CreatedTs int64   // 创建时间戳
    UpdatedTs int64   // 更新时间戳
}
```

**数据库约束**: `UNIQUE(Provider, ExternUID)` - 一个外部身份只能绑定到一个本地用户。

#### 1.2 IdentityProvider (身份提供方配置表)

**文件位置**: `store/idp.go`

```go
type IdentityProvider struct {
    ID               int32                              // 内部主键
    UID              string                             // 唯一标识 (用于 URL 路径)
    Name             string                             // 显示名称
    Type             storepb.IdentityProvider_Type     // 类型 (目前仅 OAUTH2)
    IdentifierFilter string                            // 标识符过滤正则
    Config           string                             // JSON 序列化的配置
}
```

#### 1.3 OAuth2 配置结构

```protobuf
message OAuth2Config {
    string client_id = 1;
    string client_secret = 2;  // 写保护，永不返回给前端
    string auth_url = 3;
    string token_url = 4;
    string user_info_url = 5;
    repeated string scopes = 6;
    FieldMapping field_mapping = 7;
}

message FieldMapping {
    string identifier = 1;   // 必需: 唯一标识字段名 (如 "sub", "id")
    string display_name = 2; // 可选: 显示名称字段
    string email = 3;        // 可选: 邮箱字段
    string avatar_url = 4;   // 可选: 头像 URL 字段
}
```

### 2. 数据关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           数据存储关系                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────┐        ┌──────────────────────────────┐      │
│  │   identity_provider  │        │      user_identity           │      │
│  ├──────────────────────┤        ├──────────────────────────────┤      │
│  │ id          (PK)     │        │ id               (PK)        │      │
│  │ uid         (UNIQUE) │<───────│ provider         (FK part1)  │      │
│  │ name                 │   │    │ extern_uid       (FK part2)  │      │
│  │ type                 │   │    │ user_id          (FK)────┐   │      │
│  │ identifier_filter    │   │    │ created_ts           │    │   │      │
│  │ config (JSON)        │   │    │ updated_ts           │    │   │      │
│  └──────────────────────┘   │    └────────────────────────┼────┘      │
│                              │                              │            │
│                              │    UNIQUE(provider, extern_uid)          │
│                              │                              │            │
│                              │    ┌─────────────────────┐   │            │
│                              │    │       user          │   │            │
│                              │    ├─────────────────────┤   │            │
│                              └───>│ id          (PK)   │<──┘            │
│                                   │ username  (UNIQUE) │                 │
│                                   │ role               │                 │
│                                   │ nickname           │                 │
│                                   │ email              │                 │
│                                   │ avatar_url         │                 │
│                                   │ password_hash      │                 │
│                                   └─────────────────────┘                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3. 关键查询模式

#### 3.1 查找绑定用户

```go
// 根据外部身份查找本地用户
func (s *APIV1Service) getLinkedSSOUser(
    ctx context.Context, 
    provider, externUID string,
) (*store.User, error) {
    // 1. 查询 user_identity 表
    identity, err := s.Store.GetUserIdentity(ctx, &store.FindUserIdentity{
        Provider:  &provider,
        ExternUID: &externUID,
    })
    if identity == nil {
        return nil, nil // 未找到绑定
    }
    
    // 2. 通过 user_id 关联查询 user 表
    user, err := s.Store.GetUser(ctx, &store.FindUser{ID: &identity.UserID})
    return user, err
}
```

#### 3.2 列出用户绑定的所有身份

```go
// 用于设置页面显示已绑定的身份
identities, err := s.Store.ListUserIdentities(ctx, &store.FindUserIdentity{
    UserID: &currentUser.ID,
})
```

---

## 关键数据流程图

### 1. SSO 首次登录流程

```
┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────────────┐
│  前端    │   │  第三方  │   │  /callback   │   │  后端 SignIn    │
│  SignIn  │   │   IdP    │   │   (回调)     │   │    (API)        │
└────┬─────┘   └────┬─────┘   └──────┬───────┘   └────────┬─────────┘
     │               │                  │                      │
     │  1. 点击 SSO  │                  │                      │
     │  登录按钮     │                  │                      │
     │──────────────>│                  │                      │
     │               │                  │                      │
     │               │ 2. 用户授权      │                      │
     │               │ (或拒绝)         │                      │
     │               │                  │                      │
     │<──────────────│ 3. 重定向到回调 │                      │
     │               │  带 code+state   │                      │
     │               │  或 error 参数   │                      │
     │               │                  │                      │
     │               │                  │ 4. 解析回调参数      │
     │               │                  │    - 检查 error      │
     │               │                  │    - 验证 state      │
     │               │                  │    - 提取 code       │
     │               │                  │                      │
     │               │                  │ 5. 调用 SignIn API   │
     │               │                  │ (SSO 凭证)           │
     │               │                  │─────────────────────>│
     │               │                  │                      │
     │               │                  │                      │ 6. 交换 Token
     │               │                  │                      │    (OAuth2)
     │               │                  │                      │
     │               │                  │                      │ 7. 获取 UserInfo
     │               │                  │                      │    (字段映射)
     │               │                  │                      │
     │               │                  │                      │ 8. 查找绑定
     │               │                  │                      │    (无)
     │               │                  │                      │
     │               │                  │                      │ 9. 检查注册开关
     │               │                  │                      │    DisallowUserRegistration?
     │               │                  │                      │
     │               │                  │                      │ 10. 创建用户
     │               │                  │                      │     - UUID 用户名
     │               │                  │                      │     - 随机密码
     │               │                  │                      │     - 昵称/邮箱/头像来自 IdP
     │               │                  │                      │
     │               │                  │                      │ 11. 创建 UserIdentity
     │               │                  │                      │     (Provider+ExternUID -> UserID)
     │               │                  │                      │
     │               │                  │                      │ 12. 生成 Token
     │               │                  │                      │     (Access + Refresh)
     │               │                  │                      │
     │               │                  │<─────────────────────│
     │               │                  │                      │
     │ 13. 存储 Token │                  │                      │
     │     跳转首页    │                  │                      │
     │<──────────────│                  │                      │
     │               │                  │                      │
```

### 2. SSO 登录绑定检查流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     resolveSSOUser 决策流程                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  输入: provider, externUID, currentUser (可选)                          │
│                                                                          │
│                              ┌─────────────────┐                          │
│                              │  开始           │                          │
│                              └────────┬────────┘                          │
│                                       │                                     │
│                                       ▼                                     │
│                         ┌───────────────────────┐                          │
│                         │ 查询 UserIdentity 表  │                          │
│                         │ (provider, externUID) │                          │
│                         └───────────┬───────────┘                          │
│                                     │                                         │
│                    ┌────────────────┴────────────────┐                      │
│                    │                                   │                      │
│              找到绑定                           未找到绑定                     │
│                    │                                   │                      │
│                    ▼                                   ▼                      │
│         ┌──────────────────┐          ┌──────────────────────────┐        │
│         │ currentUser 非空? │          │   currentUser 非空?      │        │
│         └────────┬─────────┘          └────────────┬─────────────┘        │
│                  │                                   │                         │
│           ┌──────┴──────┐                     ┌─────┴─────┐               │
│           │             │                     │           │               │
│          是            否                    是          否                │
│           │             │                     │           │               │
│           ▼             ▼                     ▼           ▼               │
│  ┌──────────────┐  ┌─────────┐    ┌────────────────┐  ┌──────────────┐  │
│  │ 检查是否绑定 │  │ 返回    │    │ 绑定到当前用户  │  │ 检查注册开关  │  │
│  │ 到同一用户   │  │ 已绑定  │    │                │  │              │  │
│  └──────┬───────┘  │ 用户   │    └────────┬───────┘  └──────┬───────┘  │
│         │           └─────────┘             │                  │          │
│    ┌────┴────┐                               │                  │          │
│    │         │                               │                  │          │
│   是        否                               │                  │          │
│    │         │                               │                  │          │
│    ▼         ▼                               │                  │          │
│ ┌──────┐  ┌─────────────┐                    │                  │          │
│ │返回  │  │返回错误:    │                    │                  │          │
│ │用户  │  │"已绑定到    │                    │                  │          │
│ └──────┘  │ 其他用户"   │                    │                  │          │
│           └─────────────┘                    │                  │          │
│                                                │                  │          │
│                                                ▼                  ▼          │
│                                         ┌──────────┐    ┌──────────────┐  │
│                                         │ 绑定成功 │    │注册被禁用?    │  │
│                                         │ 返回用户 │    └──────┬───────┘  │
│                                         └──────────┘           │          │
│                                                           ┌─────┴────┐     │
│                                                           │          │     │
│                                                          是         否      │
│                                                           │          │     │
│                                                           ▼          ▼     │
│                                                    ┌──────────┐ ┌────────┐ │
│                                                    │返回错误: │ │创建用户│ │
│                                                    │"注册禁用"│ │创建绑定│ │
│                                                    └──────────┘ │返回用户│ │
│                                                                 └────────┘ │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 附录: 关键代码位置速查

| 功能模块 | 文件路径 | 关键函数/组件 |
|---------|---------|-------------|
| **前端 SSO 启动** | `web/src/pages/SignIn.tsx` | `handleSignInWithIdentityProvider` |
| **OAuth State 管理** | `web/src/utils/oauth.ts` | `storeOAuthState`, `validateOAuthState` |
| **回调处理** | `web/src/pages/AuthCallback.tsx` | `AuthCallback` 组件 |
| **后端 SSO 登录入口** | `server/router/api/v1/auth_service.go:64` | `SignIn` |
| **身份解析** | `server/router/api/v1/auth_service.go:219` | `resolveSSOIdentity` |
| **用户绑定/创建** | `server/router/api/v1/auth_service.go:123` | `resolveSSOUser` |
| **绑定到现有用户** | `server/router/api/v1/auth_service.go:286` | `bindSSOIdentityToUser` |
| **查找绑定用户** | `server/router/api/v1/auth_service.go:265` | `getLinkedSSOUser` |
| **OAuth2 客户端** | `internal/idp/oauth2/oauth2.go` | `ExchangeToken`, `UserInfo` |
| **身份提供方配置存储** | `store/idp.go` | `IdentityProvider` 结构体 |
| **用户身份绑定存储** | `store/user_identity.go` | `UserIdentity` 结构体 |
| **登录入口开关** | `proto/gen/store/instance_setting.pb.go` | `InstanceGeneralSetting` |

---

## 总结

### 核心设计亮点

1. **分离的身份标识**: 外部身份的 `Identifier` 从不直接用作本地用户名，而是通过 `UserIdentity` 表关联，避免外部身份变更影响本地用户。

2. **并发安全**: 通过数据库 `UNIQUE` 约束和错误检测处理并发登录/绑定场景，确保数据一致性。

3. **灵活的开关控制**:
   - `DisallowPasswordAuth`: 可强制使用 SSO，但保留管理员密码登录能力
   - `DisallowUserRegistration`: 可禁止新用户注册，但允许已绑定用户继续登录

4. **完整的错误处理**:
   - 前端处理 OAuth 协议级错误 (`error`, `error_description`)
   - 状态验证防止 CSRF 攻击
   - 后端返回详细的业务错误信息

5. **PKCE 支持**: 增强 OAuth2 安全性，适配公共客户端场景。

6. **字段映射机制**: 通过 `FieldMapping` 配置支持不同 OAuth 提供方的用户信息字段差异。

7. **标识符过滤**: 通过 `IdentifierFilter` 正则限制哪些外部用户可以登录，适用于企业内部 SSO 场景。
