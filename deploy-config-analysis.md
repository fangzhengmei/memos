# 自托管部署配置分析报告

## 一、配置来源与优先级

Memos 应用采用多源配置机制，通过 Viper 库实现配置的加载与合并。

### 1.1 配置来源优先级（从高到低）

1. **命令行参数 (CLI Flags)**
   - 通过 Cobra 定义的 `PersistentFlags`
   - 例如：`--port 8081`、`--data ./data`

2. **环境变量 (Environment Variables)**
   - 前缀：`MEMOS_`
   - 键名转换：`-` → `_`，自动大写
   - 例如：`MEMOS_PORT=8081`、`MEMOS_INSTANCE_URL=https://memos.example.com`

3. **默认值 (Defaults)**
   - 在 `init()` 函数中通过 `viper.SetDefault()` 设置
   - 例如：`driver: sqlite`、`port: 8081`

### 1.2 支持的配置项

| 配置项 | 命令行参数 | 环境变量 | 默认值 | 说明 |
|--------|-----------|---------|--------|------|
| demo | `--demo` | `MEMOS_DEMO` | `false` | 演示模式 |
| addr | `--addr` | `MEMOS_ADDR` | `""` | 绑定地址 |
| port | `--port` | `MEMOS_PORT` | `8081` | 监听端口 |
| unix-sock | `--unix-sock` | `MEMOS_UNIX_SOCK` | `""` | Unix Socket 路径 |
| data | `--data` | `MEMOS_DATA` | 见下方 | 数据目录 |
| driver | `--driver` | `MEMOS_DRIVER` | `sqlite` | 数据库驱动 (sqlite/mysql/postgres) |
| dsn | `--dsn` | `MEMOS_DSN` | `""` | 数据库连接字符串 |
| instance-url | `--instance-url` | `MEMOS_INSTANCE_URL` | `""` | 实例访问 URL |
| allow-private-webhooks | `--allow-private-webhooks` | `MEMOS_ALLOW_PRIVATE_WEBHOOKS` | `false` | 允许私有 IP Webhook |

### 1.3 数据目录默认值逻辑

`internal/profile/profile.go:59-108` 中的 `Validate()` 方法处理数据目录：

- **Windows**: `%ProgramData%\memos`
- **Linux/macOS**:
  1. 检查 `/var/opt/memos` 是否存在且可写（Docker 场景）
  2. 否则使用当前目录 `.`
- 若目录不存在，自动创建（权限 `0770`）

### 1.4 SQLite DSN 自动生成

当 `driver=sqlite` 且未指定 `dsn` 时：
- 生产模式：`{data_dir}/memos_prod.db`
- 演示模式：`{data_dir}/memos_demo.db`

---

## 二、文件配置链路分析

### 2.1 配置文件不存在结论

**关键发现**：项目完全不使用 Viper 的配置文件功能，不存在任何文件配置链路。

**证据**：
- 代码中未调用 `viper.SetConfigFile()`
- 代码中未调用 `viper.AddConfigPath()`
- 代码中未调用 `viper.ReadInConfig()`
- 代码中未调用 `viper.ConfigFileUsed()`
- 项目根目录不存在任何 `config.yaml`、`config.json`、`config.toml` 等配置文件模板

**搜索结果验证**：
```bash
# Viper 配置文件相关函数调用：无匹配
viper.SetConfigFile | viper.AddConfigPath | viper.ReadInConfig | viper.ConfigFileUsed
```

### 2.2 实际合并边界

项目实际生效的配置合并优先级（从高到低）：

```
优先级  来源          代码位置
 1      命令行参数    cmd/memos/main.go:110-118 (Cobra PersistentFlags)
 2      环境变量      cmd/memos/main.go:148-150 (viper.AutomaticEnv)
 3      默认值        cmd/memos/main.go:106-108 (viper.SetDefault)
```

**Viper 实际配置链合并规则**：

| 配置键 | 合并边界说明 |
|--------|-------------|
| `port` | 命令行 `--port` > 环境变量 `MEMOS_PORT` > 默认值 `8081` |
| `driver` | 命令行 `--driver` > 环境变量 `MEMOS_DRIVER` > 默认值 `sqlite` |
| `data` | 命令行 `--data` > 环境变量 `MEMOS_DATA` > Validate() 自动推断 |
| `unix-sock` | 命令行 `--unix-sock` > 环境变量 `MEMOS_UNIX_SOCK` > 空（使用 TCP） |
| `instance-url` | 命令行 `--instance-url` > 环境变量 `MEMOS_INSTANCE_URL` > 空（功能受限） |

**关键代码** (`cmd/memos/main.go:148-150`):
```go
viper.SetEnvPrefix("memos")
viper.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))
viper.AutomaticEnv()
```

### 2.3 运行时配置（数据库存储）与启动配置的边界

项目存在两套配置体系，边界清晰：

```
┌─────────────────────────────────────────────────────────────┐
│                    启动配置 (不可热重载)                       │
│  Profile 结构体                                              │
│  ├── addr, port, unix-sock     (网络监听)                   │
│  ├── data, driver, dsn         (数据库连接)                  │
│  ├── instance-url              (CORS / 外部 URL)            │
│  └── demo, allow-private-webhooks                            │
│                                                              │
│  来源: 命令行 > 环境变量 > 默认值 > Validate() 推断          │
│  加载时机: main() 启动时一次性构建                            │
│  修改方式: 重启服务                                           │
├─────────────────────────────────────────────────────────────┤
│                    运行时配置 (可热重载)                       │
│  数据库存储                                                   │
│  ├── instance_setting 表                                      │
│  │   ├── SecretKey              (JWT 密钥)                   │
│  │   ├── SchemaVersion          (数据库版本)                 │
│  │   ├── EmailConfig            (SMTP 邮件配置)              │
│  │   └── StorageSetting         (S3/本地存储配置)            │
│  └── idp 表                     (OAuth2 身份提供商)           │
│                                                              │
│  来源: 首次启动自动生成 + 用户通过 API/Web UI 修改           │
│  加载时机: 每次请求按需读取 + 内存缓存                        │
│  修改方式: API 调用实时生效                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、配置加载流程

### 3.1 Viper 初始化流程 (`cmd/memos/main.go:105-153`)

```
init() 函数执行顺序:
├── 1. 设置默认值 (viper.SetDefault)
│       ├── demo: false
│       ├── driver: sqlite
│       └── port: 8081
│
├── 2. 定义 Cobra 命令行标志
│       ├── --demo, --addr, --port
│       ├── --unix-sock, --data, --driver, --dsn
│       ├── --instance-url, --allow-private-webhooks
│
├── 3. 绑定标志到 Viper (viper.BindPFlag)
│       └── 共 9 个标志绑定到对应的配置键
│
├── 4. 设置环境变量前缀 (MEMOS_)
│       └── viper.SetEnvPrefix("memos")
│
├── 5. 设置键名替换规则 (- → _)
│       └── viper.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))
│
└── 6. 启用自动环境变量读取 (viper.AutomaticEnv)
        └── 环境变量自动映射到配置键
```

### 3.2 配置合并机制

**实际生效的合并优先级**（从高到低）：
1. 显式调用 `Set()` 设置的值（代码中未使用）
2. 命令行参数
3. 环境变量
4. 默认值

**环境变量映射规则**：
- 前缀 `MEMOS_` + 配置键（大写）+ `-` 替换为 `_`
- 例：`instance-url` → `MEMOS_INSTANCE_URL`
- 例：`unix-sock` → `MEMOS_UNIX_SOCK`

### 3.3 配置验证与后处理

配置加载后，`Profile.Validate()` (`internal/profile/profile.go:59-108`) 执行：

1. **数据目录处理**: 设置默认值、创建目录、转换为绝对路径
2. **DSN 生成**: SQLite 驱动自动生成 DSN
3. **路径规范化**: 移除路径尾部的斜杠

**Validate() 是配置合并的最终边界**：
- 在此之前：Viper 完成命令行 + 环境变量 + 默认值的合并
- 在此之后：Profile 对象中的值是最终生效的配置

---

## 四、模块初始化顺序

### 4.1 启动流程总览 (`cmd/memos/main.go:28-94`)

```
main 执行顺序:
│
├── 1. 从 Viper 读取配置，构建 Profile 对象
│       └── cmd/memos/main.go:29-41
│
├── 2. Profile.Validate() - 配置验证与后处理
│       └── internal/profile/profile.go:59-108
│
├── 3. 创建数据库驱动 (db.NewDBDriver)
│       └── store/db/db.go:14-31
│
├── 4. 创建 Store 实例 (store.New)
│       └── store/store.go:28-47
│
├── 5. 数据库迁移 (store.Migrate)
│       └── store/migrator.go:100-136
│
├── 6. 创建 Server 实例 (server.NewServer)
│       └── server/server.go:44-96
│
├── 7. 启动 Server (server.Start)
│       └── server/server.go:98-129
│
└── 8. 等待信号，优雅关闭
        └── cmd/memos/main.go:70-93
```

### 4.2 启动阶段顺序保障机制

#### 4.2.1 同步初始化保障（阻塞式）

**所有关键初始化步骤均为同步阻塞执行**，确保顺序性：

```
阶段 1: Profile 构建与验证 (同步阻塞)
  └── 失败则打印错误并 return，进程退出 (code=0)

阶段 2: 数据库驱动创建 (同步阻塞)
  └── 失败则 cancel() 上下文，打印错误并 return

阶段 3: Store 创建 (同步阻塞)
  └── 纯内存操作，无失败点

阶段 4: 数据库迁移 (同步阻塞)
  ├── preMigrate() - 检查是否需要初始化
  ├── 检查版本兼容性（< v0.22 拒绝启动）
  ├── 检查降级（禁止）
  ├── 应用增量迁移（事务保障原子性）
  └── 失败则 cancel() 上下文，打印错误并 return

阶段 5: Server 实例创建 (同步阻塞)
  ├── 创建 Echo 实例
  ├── 获取/创建 SecretKey（数据库读写）
  ├── 注册所有路由（前端、API、文件服务等）
  └── 失败则 cancel() 上下文，打印错误并 return

阶段 6: Server 启动
  ├── 创建 Listener (同步阻塞)
  ├── 设置 Unix Socket 权限 (同步阻塞)
  ├── HTTP Serve (异步 goroutine，但 Start() 本身同步返回)
  └── 后台任务启动 (见下方)
```

#### 4.2.2 失败即退出的保障策略

**任何同步阶段失败都会立即终止启动流程**：

```go
// cmd/memos/main.go:43-46
if err := instanceProfile.Validate(); err != nil {
    slog.Error("failed to validate profile", "error", err)
    return  // 直接返回，不继续
}

// cmd/memos/main.go:49-54
dbDriver, err := db.NewDBDriver(instanceProfile)
if err != nil {
    cancel()
    slog.Error("failed to create db driver", "error", err)
    return
}

// cmd/memos/main.go:57-61
if err := storeInstance.Migrate(ctx); err != nil {
    cancel()
    slog.Error("failed to migrate", "error", err)
    return
}
```

#### 4.2.3 数据库事务保障

数据库迁移使用事务确保原子性：

```go
// store/migrator.go:147-152
tx, err := s.driver.GetDB().Begin()  // 开始事务
if err != nil {
    return errors.Wrap(err, "failed to start transaction")
}
defer tx.Rollback()  // 任何错误自动回滚

// ... 执行所有迁移 ...

if err := tx.Commit(); err != nil {  // 全部成功才提交
    return errors.Wrap(err, "failed to commit migration transaction")
}
```

### 4.3 后台异步任务分界

#### 4.3.1 同步与异步的明确分界点

**分界点**: `server.Start()` 方法 (`server/server.go:98-129`)

```
┌──────────────────────────────────────────────────────────┐
│              同步启动阶段 (Start() 方法内)                 │
│                                                          │
│  98: func (s *Server) Start(ctx context.Context) error { │
│                                                          │
│  [同步] 决定监听方式 (TCP vs Unix Socket)                 │
│  ├── 100-106: if len(s.Profile.UNIXSock) == 0 { ... }   │
│  └── 选择 address 和 network                              │
│                                                          │
│  [同步阻塞] 创建 Listener                                 │
│  └── 107: listener, err := net.Listen(network, address)  │
│      └── 失败则返回 error，Start() 终止                   │
│                                                          │
│  [同步] Unix Socket 权限设置                              │
│  └── 112-116: if network == "unix" { os.Chmod(...) }    │
│      └── 失败则关闭 listener 并返回 error                 │
│                                                          │
│  [异步] HTTP 服务启动 (goroutine)                         │
│  └── 121-125: go func() { s.httpServer.Serve(...) }()   │
│      └── Serve() 内部是阻塞的，但在 goroutine 中执行      │
│      └── 错误仅通过 slog 记录，不终止进程                 │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  [异步分界点] startBackgroundRunners() 调用       │    │
│  │  └── 126: s.startBackgroundRunners(ctx)         │    │
│  │                                                  │    │
│  │  此方法内部启动后台 goroutine，但方法本身同步返回  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  [同步] Start() 返回 nil（成功）                          │
│  └── 128: return nil                                    │
└──────────────────────────────────────────────────────────┘
```

#### 4.3.2 后台任务启动机制

```go
// server/server.go:150-171
func (s *Server) startBackgroundRunners(ctx context.Context) {
    // 为每个后台 runner 创建独立的 context
    s3Context, s3Cancel := context.WithCancel(ctx)
    s.backgroundRunnerCancels = append(s.backgroundRunnerCancels, s3Cancel)

    // 创建 S3 presign runner
    s3presignRunner := s3presign.NewRunner(s.Store)
    
    // [同步执行一次] RunOnce 在当前 goroutine 中执行
    s3presignRunner.RunOnce(ctx)  // ← 同步执行，可能阻塞
    
    // [异步持续运行] 在独立 goroutine 中启动 ticker
    s.backgroundRunnerWG.Add(1)
    go func() {
        defer s.backgroundRunnerWG.Done()
        s3presignRunner.Run(s3Context)  // ← 异步循环
        slog.Info("s3presign runner stopped")
    }()

    slog.Info("background runners started")
}
```

#### 4.3.3 S3 Presign Runner 内部机制

```
S3 Presign Runner 生命周期:

RunOnce(ctx) - 同步执行 (startBackgroundRunners 调用时)
  └── CheckAndPresign(ctx)
      ├── 读取数据库中的存储配置
      ├── 分批查询 S3 附件 (每次 100 条)
      ├── 对需要更新的附件重新生成 presigned URL
      └── 更新数据库

Run(ctx) - 异步循环 (goroutine 中)
  ├── time.NewTicker(12 * time.Hour)
  └── for {
          select {
          case <-ticker.C:
              RunOnce(ctx)  // 每 12 小时执行一次
          case <-ctx.Done():
              return  // 收到取消信号退出
          }
      }
```

#### 4.3.4 同步与异步的关键区别

| 特性 | 同步启动阶段 | 后台异步任务 |
|------|------------|-------------|
| 执行时机 | `main()` → `Start()` 返回前 | `Start()` 内启动 goroutine |
| 错误处理 | 失败即终止启动 | 失败仅记录日志，不影响主进程 |
| 上下文管理 | 共享父 context | 独立子 context，可单独取消 |
| 生命周期 | 一次性执行 | 持续运行直到 Shutdown |
| 对可用性影响 | 必须全部成功 | 失败不影响核心服务 |

### 4.4 Server 内部初始化顺序 (`server/server.go:44-96`)

```
NewServer() 内部初始化:
│
├── 1. 创建 Echo 实例，注册 Recover 中间件
│
├── 2. 获取/创建实例基础设置 (含 SecretKey)
│       └── getOrUpsertInstanceBasicSetting()
│       │
│       ├── 从数据库读取 instance_setting
│       ├── 如果 SecretKey 为空，生成 UUID 并保存
│       └── 演示模式使用固定值 "usememos"
│
├── 3. 注册健康检查端点 /healthz
│
├── 4. 前端静态文件服务
│       └── frontend.NewFrontendService().Serve()
│
├── 5. 创建 API V1 服务
│       └── apiv1.NewAPIV1Service()
│       │
│       ├── 创建 Markdown 服务
│       ├── 创建 SSEHub
│       └── 初始化信号量 (thumbnail/image processing)
│
├── 6. 文件服务器路由 (在 gRPC Gateway 之前注册)
│       └── fileserver.NewFileServerService().RegisterRoutes()
│
├── 7. RSS 服务路由
│       └── rss.NewRSSService().RegisterRoutes()
│
├── 8. gRPC Gateway 注册
│       └── apiV1Service.RegisterGateway()
│       │
│       ├── 注册 gRPC-Gateway mux (含 auth middleware)
│       ├── 注册 SSE 端点
│       └── 注册 Connect handlers
│
└── 9. MCP 服务路由
        └── mcprouter.NewMCPService().RegisterRoutes()
```

### 4.5 数据库迁移流程 (`store/migrator.go:100-136`)

```
Migrate() 执行顺序:
│
├── 1. preMigrate() - 预迁移检查
│       │
│       ├── 检查数据库是否初始化 (IsInitialized)
│       │   └── 未初始化 → 应用 LATEST.sql 完整 schema
│       │
│       └── 检查最低升级版本
│           └── < v0.22.0 → 拒绝直接升级，要求先升级到 v0.25.3
│
├── 2. 获取当前 schema 版本
│
├── 3. 版本检查
│       │
│       ├── 检查降级 (禁止降级)
│       └── 检查是否需要升级
│
├── 4. 应用增量迁移 (applyMigrations)
│       │
│       ├── 按版本号排序迁移文件
│       ├── 在事务中执行所有迁移
│       └── 更新 schema 版本
│
└── 5. 演示模式 - 种子数据 (seed)
        └── 仅 SQLite 支持
```

---

## 五、关键冲突场景分析

### 5.1 unix-sock 与 addr/port 冲突

#### 5.1.1 冲突规则

**Unix Socket 完全覆盖 TCP 配置**：

```go
// server/server.go:100-106
if len(s.Profile.UNIXSock) == 0 {
    // 使用 TCP
    address = fmt.Sprintf("%s:%d", s.Profile.Addr, s.Profile.Port)
    network = "tcp"
} else {
    // 使用 Unix Socket，addr 和 port 被完全忽略
    address = s.Profile.UNIXSock
    network = "unix"
}
```

**Profile 结构体注释确认** (`internal/profile/profile.go:22`):
```go
// UNIXSock is the IPC binding path. Overrides Addr and Port
UNIXSock string
```

#### 5.1.2 冲突场景示例

| 场景 | unix-sock | addr | port | 实际监听方式 |
|------|-----------|------|------|-------------|
| 场景 1 | 空 | 0.0.0.0 | 8081 | TCP: 0.0.0.0:8081 |
| 场景 2 | /tmp/memos.sock | 0.0.0.0 | 8081 | Unix Socket: /tmp/memos.sock |
| 场景 3 | /tmp/memos.sock | （未设置）| （未设置）| Unix Socket: /tmp/memos.sock |
| 场景 4 | （未设置）| 127.0.0.1 | 9000 | TCP: 127.0.0.1:9000 |

#### 5.1.3 Unix Socket 权限处理

```go
// server/server.go:112-117
if network == "unix" {
    // 设置 Socket 文件权限为 0660 (rw-rw----)
    if err := os.Chmod(address, 0660); err != nil {
        _ = listener.Close()
        return errors.Wrap(err, "failed to chmod socket")
    }
}
```

**注意**：权限设置失败会导致启动失败，这是一个硬约束。

### 5.2 instance-url 约束场景

#### 5.2.1 instance-url 的功能影响

`instance-url` 不为空时启用以下功能，为空时这些功能受限或禁用：

| 功能模块 | instance-url 已设置 | instance-url 为空 | 代码位置 |
|---------|---------------------|------------------|---------|
| Connect RPC CORS | 允许跨域（匹配 scheme + host）| 拒绝非同源请求 | `server/router/api/v1/v1.go:180-187` |
| MCP 服务 CORS | 允许跨域（匹配 scheme + host）| 拒绝非同源请求 | `server/router/mcp/access.go:99-108` |
| robots.txt | 返回正确的 Host 和 Sitemap | 返回 404 | `server/router/frontend/frontend.go:130-142` |
| sitemap.xml | 生成完整 sitemap | 返回 404 | `server/router/frontend/frontend.go:145-148` |
| 邮件通知 | 生成完整的 memo 链接 | 跳过邮件发送 | `server/router/api/v1/test/user_notification_test.go:240` |

#### 5.2.2 CORS 验证逻辑

**Connect RPC CORS 验证** (`server/router/api/v1/v1.go:170-188`):

```go
func (s *APIV1Service) isAllowedConnectOrigin(c *echo.Context, origin string) bool {
    originURL, err := url.Parse(origin)
    if err != nil || originURL.Scheme == "" || originURL.Host == "" {
        return false
    }

    // 同源请求总是允许
    if strings.EqualFold(originURL.Host, c.Request().Host) {
        return true
    }

    // 非同源请求需要 instance-url 配置
    if s.Profile == nil || s.Profile.InstanceURL == "" {
        return false  // ← instance-url 为空，拒绝
    }

    instanceURL, err := url.Parse(s.Profile.InstanceURL)
    if err != nil || instanceURL.Scheme == "" || instanceURL.Host == "" {
        return false
    }

    // 匹配 scheme 和 host
    return strings.EqualFold(originURL.Scheme, instanceURL.Scheme) &&
           strings.EqualFold(originURL.Host, instanceURL.Host)
}
```

**MCP 服务 CORS 验证** (`server/router/mcp/access.go:84-109`):

```go
func (s *MCPService) isAllowedOrigin(r *http.Request) bool {
    origin := r.Header.Get("Origin")
    if origin == "" {
        return true  // 无 Origin 头允许
    }

    // ... 解析 origin ...

    // 同源允许
    if sameOriginHost(originURL.Host, r.Host) {
        return true
    }

    // 非同源需要 instance-url
    if s.profile.InstanceURL == "" {
        return false  // ← instance-url 为空，拒绝
    }

    // 匹配 scheme 和 host
    instanceURL, err := url.Parse(s.profile.InstanceURL)
    // ...
    return strings.EqualFold(originURL.Scheme, instanceURL.Scheme) &&
           sameOriginHost(originURL.Host, instanceURL.Host)
}
```

#### 5.2.3 robots.txt 和 sitemap.xml 约束

```go
// server/router/frontend/frontend.go:170-176
func normalizeInstanceURL(instanceURL string) (string, error) {
    instanceURL = strings.TrimRight(instanceURL, "/")
    if instanceURL == "" {
        // 返回 404
        return "", echo.NewHTTPError(http.StatusNotFound, "instance URL is not configured")
    }
    return instanceURL, nil
}
```

**单元测试验证** (`server/router/frontend/frontend_test.go:219`):
```go
func TestFrontendService_SitemapRoutesRequireInstanceURL(t *testing.T) {
    // ...
    ts.Profile.InstanceURL = ""  // 为空
    // ... 预期 404
}
```

#### 5.2.4 instance-url 冲突场景

| 场景 | instance-url 值 | 外部域请求来源 | Connect RPC | MCP | robots.txt |
|------|----------------|---------------|-------------|-----|-----------|
| 场景 1 | 空 | https://external.com | 拒绝 | 拒绝 | 404 |
| 场景 2 | https://memos.example.com | https://memos.example.com | 允许 | 允许 | 正常 |
| 场景 3 | https://memos.example.com | https://other.com | 拒绝 | 拒绝 | 正常 |
| 场景 4 | https://memos.example.com | http://memos.example.com | 拒绝 (scheme 不匹配) | 拒绝 (scheme 不匹配) | 正常 |
| 场景 5 | https://memos.example.com:8443 | https://memos.example.com | 拒绝 (host 不匹配) | 拒绝 (host 不匹配) | 正常 |

### 5.3 data 目录与 dsn 冲突

#### 5.3.1 冲突规则

**SQLite 模式下，dsn 优先级高于自动生成**：

```go
// internal/profile/profile.go:99-106
if p.Driver == "sqlite" && p.DSN == "" {
    // 仅当 dsn 为空时才自动生成
    mode := "prod"
    if p.Demo {
        mode = "demo"
    }
    dbFile := fmt.Sprintf("memos_%s.db", mode)
    p.DSN = filepath.Join(dataDir, dbFile)
}
```

**MySQL/PostgreSQL 模式下，data 目录对 dsn 无影响**：
- dsn 必须完整指定（如 `postgresql://user:pass@host/db`）
- data 目录仅用于附件存储等其他用途

#### 5.3.2 冲突场景

| 场景 | driver | data | dsn | 最终数据库路径 |
|------|--------|------|-----|---------------|
| 场景 1 | sqlite | ./data | （空）| ./data/memos_prod.db |
| 场景 2 | sqlite | ./data | /custom/memos.db | /custom/memos.db |
| 场景 3 | sqlite | （空）| （空）| {默认data}/memos_prod.db |
| 场景 4 | postgres | ./data | postgresql://... | 使用 dsn，data 仅用于附件 |

### 5.4 其他配置约束

#### 5.4.1 driver 有效值约束

**仅支持三种驱动**：
- `sqlite`
- `mysql`
- `postgres`

**无效驱动导致启动失败** (`store/db/db.go:25-27`):
```go
default:
    return nil, errors.New("unknown db driver")
```

#### 5.4.2 demo 模式的影响

**demo=true 时的行为差异**：
1. **SecretKey**: 使用固定值 `"usememos"` 而非数据库中的值
2. **数据库迁移**: 应用 seed 数据（仅 SQLite）
3. **Connect RPC**: 启用完整的 stacktrace 日志

```go
// server/server.go:59-62
secret := "usememos"
if !profile.Demo {
    secret = instanceBasicSetting.SecretKey
}

// server/router/api/v1/v1.go:141-147
logStacktraces := s.Profile.Demo
connectInterceptors := connect.WithInterceptors(
    NewMetadataInterceptor(),
    NewLoggingInterceptor(logStacktraces),  // demo 模式记录完整 stacktrace
    // ...
)
```

---

## 六、配置依赖关系

### 6.1 启动时配置依赖

```
Profile 配置
│
├── 数据库驱动选择 ──→ driver (sqlite/mysql/postgres)
│       └── 影响 db.NewDBDriver() 的分支
│
├── 数据库连接 ──→ dsn + data
│       └── SQLite: data 目录决定 dsn 默认值
│       └── MySQL/PostgreSQL: dsn 必须完整指定
│
├── 网络监听 ──→ addr + port + unix-sock
│       └── unix-sock 存在时完全覆盖 addr/port
│
├── CORS 控制 ──→ instance-url
│       └── Connect RPC 和 MCP 服务的跨域验证
│       └── robots.txt 和 sitemap.xml 的可用性
│
└── Webhook 安全 ──→ allow-private-webhooks
        └── 控制 webhook 是否可访问私有/保留 IP
```

### 6.2 运行时配置（数据库存储）

除了启动配置，部分配置存储在数据库中：

| 配置类型 | 存储位置 | 说明 |
|---------|---------|------|
| SecretKey | `instance_setting` | 首次启动时自动生成 UUID |
| SchemaVersion | `instance_setting` | 数据库 schema 版本 |
| 邮件配置 | `instance_setting` | SMTP 服务器配置 |
| 存储配置 | `instance_setting` | S3/本地存储配置 |
| IDP 配置 | `idp` 表 | OAuth2 身份提供商配置 |

**启动时 SecretKey 处理** (`server/server.go:219-240`):
```go
// 获取或创建实例基础设置
// 如果 SecretKey 为空，自动生成 UUID 并保存
// demo 模式跳过，使用固定值 "usememos"
```

---

## 七、配置覆盖示例

### 7.1 Docker Compose 配置示例

```yaml
services:
  memos:
    image: neosmemo/memos:latest
    ports:
      - "5230:5230"
    environment:
      - MEMOS_DRIVER=postgres
      - MEMOS_DSN=postgresql://user:pass@db:5432/memos
      - MEMOS_INSTANCE_URL=https://memos.example.com
    volumes:
      - memos-data:/var/opt/memos
```

### 7.2 Unix Socket 部署示例

```bash
# 使用 Unix Socket 替代 TCP
export MEMOS_UNIX_SOCK=/run/memos/memos.sock
export MEMOS_INSTANCE_URL=https://memos.example.com

# 注意：addr 和 port 会被忽略
export MEMOS_ADDR=0.0.0.0  # 无效
export MEMOS_PORT=8081     # 无效

memos
```

### 7.3 命令行覆盖示例

```bash
# 环境变量设置默认端口
export MEMOS_PORT=8080

# 命令行参数覆盖环境变量
memos --port 9000  # 实际使用 9000
```

### 7.4 优先级验证

```bash
# 场景：同时设置默认值、环境变量、命令行
# 默认: port=8081
export MEMOS_PORT=8080
memos --port 9000
# 结果：使用 9000（命令行优先级最高）
```

### 7.5 instance-url 必要场景

```bash
# 场景：需要从外部域名访问 Connect RPC/MCP
# 必须设置 instance-url
export MEMOS_INSTANCE_URL=https://memos.example.com

# 如果不设置：
# - 浏览器从 memos.example.com 访问时，Connect RPC 请求会被 CORS 拒绝
# - robots.txt 和 sitemap.xml 返回 404
```

---

## 八、关键代码位置汇总

| 功能 | 文件路径 | 关键函数/行号 |
|------|---------|--------------|
| 配置加载入口 | `cmd/memos/main.go` | `init()`: 105-153, `Run()`: 28-94 |
| Profile 定义 | `internal/profile/profile.go` | `Profile` 结构体: 15-37, `Validate()`: 59-108 |
| Viper 绑定 | `cmd/memos/main.go` | 120-150 |
| 数据库驱动创建 | `store/db/db.go` | `NewDBDriver()`: 14-31 |
| SQLite 初始化 | `store/db/sqlite/sqlite.go` | `NewDB()`: 24-57 |
| 数据库迁移 | `store/migrator.go` | `Migrate()`: 100-136 |
| Server 初始化 | `server/server.go` | `NewServer()`: 44-96 |
| Server 启动 | `server/server.go` | `Start()`: 98-129 |
| 后台任务启动 | `server/server.go` | `startBackgroundRunners()`: 150-171 |
| S3 Runner | `server/runner/s3presign/runner.go` | `Run()`: 29-41, `RunOnce()`: 43-45 |
| Connect RPC CORS | `server/router/api/v1/v1.go` | `isAllowedConnectOrigin()`: 170-188 |
| MCP CORS | `server/router/mcp/access.go` | `isAllowedOrigin()`: 84-109 |
| Unix Socket 覆盖 | `server/server.go` | 100-106 |
| robots/sitemap 约束 | `server/router/frontend/frontend.go` | `normalizeInstanceURL()`: 170-176 |
| API V1 服务 | `server/router/api/v1/v1.go` | `NewAPIV1Service()`: 50-65, `RegisterGateway()`: 68-168 |

---

## 九、注意事项

### 9.1 配置系统

1. **无配置文件支持**：项目完全不使用 Viper 的配置文件功能，仅支持环境变量和命令行参数
2. **配置合并边界**：命令行 > 环境变量 > 默认值，无文件配置层
3. **两套配置体系**：启动配置（Profile）和运行时配置（数据库）完全分离

### 9.2 启动顺序保障

4. **同步阻塞初始化**：所有关键步骤（验证、数据库、迁移、Server 创建）均为同步阻塞执行
5. **失败即退出**：任何同步阶段失败都会立即终止启动流程，不会部分启动
6. **事务保障**：数据库迁移使用事务确保原子性，部分失败自动回滚
7. **异步分界点**：`startBackgroundRunners()` 是同步与异步的明确分界

### 9.3 后台任务

8. **RunOnce 同步执行**：S3 Presign 的首次执行是同步的，可能阻塞启动
9. **后台错误不影响主进程**：后台任务失败仅记录日志，不会终止服务
10. **独立上下文管理**：每个后台 runner 有独立的 context，可单独取消

### 9.4 关键冲突场景

11. **Unix Socket 完全覆盖**：设置 `unix-sock` 后，`addr` 和 `port` 被完全忽略
12. **Unix Socket 权限硬约束**：`chmod 0660` 失败会导致启动失败
13. **instance-url 为空的功能限制**：
    - Connect RPC 和 MCP 拒绝非同源跨域请求
    - robots.txt 和 sitemap.xml 返回 404
    - 邮件通知可能跳过
14. **instance-url CORS 严格匹配**：scheme 和 host 必须完全匹配（端口不匹配也会拒绝）
15. **SQLite dsn 优先级**：显式设置的 dsn 优先级高于 data 目录自动生成的路径
16. **demo 模式 SecretKey 固定**：demo=true 时使用 `"usememos"` 而非数据库中的值

### 9.5 其他

17. **DSN 格式差异**：
    - SQLite: 文件路径（可自动生成）
    - MySQL: `user:pass@tcp(host:port)/dbname`
    - PostgreSQL: `postgresql://user:pass@host:port/dbname`
18. **SecretKey 持久化**：首次启动生成的 SecretKey 存储在数据库中，后续启动会复用
19. **升级限制**：从 v0.22 之前的版本升级，必须先升级到 v0.25.3 作为中间版本
20. **数据目录权限**：自动创建的数据目录权限为 `0770`，确保进程有读写权限
