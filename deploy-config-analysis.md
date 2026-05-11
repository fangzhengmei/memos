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

## 二、配置加载流程

### 2.1 Viper 初始化流程 (`cmd/memos/main.go:105-153`)

```
init() 函数执行顺序:
├── 1. 设置默认值 (viper.SetDefault)
├── 2. 定义 Cobra 命令行标志
├── 3. 绑定标志到 Viper (viper.BindPFlag)
├── 4. 设置环境变量前缀 (MEMOS_)
├── 5. 设置键名替换规则 (- → _)
└── 6. 启用自动环境变量读取 (viper.AutomaticEnv)
```

### 2.2 配置合并机制

Viper 的配置优先级（从高到低）：
1. 显式调用 `Set()` 设置的值
2. 命令行参数
3. 环境变量
4. 配置文件（本项目未使用）
5. 默认值

**关键代码** (`cmd/memos/main.go:148-150`):
```go
viper.SetEnvPrefix("memos")
viper.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))
viper.AutomaticEnv()
```

这意味着：
- 环境变量 `MEMOS_INSTANCE_URL` 会映射到 `instance-url` 配置键
- 命令行 `--instance-url` 的优先级高于环境变量

### 2.3 配置验证与后处理

配置加载后，`Profile.Validate()` (`internal/profile/profile.go:59-108`) 执行：

1. **数据目录处理**: 设置默认值、创建目录、转换为绝对路径
2. **DSN 生成**: SQLite 驱动自动生成 DSN
3. **路径规范化**: 移除路径尾部的斜杠

---

## 三、模块初始化顺序

### 3.1 启动流程总览 (`cmd/memos/main.go:28-94`)

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

### 3.2 Server 内部初始化顺序 (`server/server.go:44-96`)

```
NewServer() 内部初始化:
│
├── 1. 创建 Echo 实例，注册 Recover 中间件
│
├── 2. 获取/创建实例基础设置 (含 SecretKey)
│       └── getOrUpsertInstanceBasicSetting()
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

### 3.3 数据库迁移流程 (`store/migrator.go:100-136`)

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

### 3.4 后台任务启动 (`server/server.go:150-171`)

`Start()` 方法在 HTTP 服务启动后启动后台任务：

```
startBackgroundRunners():
│
└── S3 Presign Runner
    ├── RunOnce(ctx) - 立即执行一次
    └── Run(s3Context) - 持续运行的 goroutine
```

---

## 四、配置依赖关系

### 4.1 启动时配置依赖

```
Profile 配置
│
├── 数据库驱动选择 ──→ driver (sqlite/mysql/postgres)
│       └── 影响 db.NewDBDriver() 的分支
│
├── 数据库连接 ──→ dsn + data
│       └── SQLite: data 目录决定 dsn 默认值
│
├── 网络监听 ──→ addr + port + unix-sock
│       └── unix-sock 存在时覆盖 addr/port
│
├── CORS 控制 ──→ instance-url
│       └── 用于 Connect RPC 的跨域验证
│
└── Webhook 安全 ──→ allow-private-webhooks
        └── 控制 webhook 是否可访问私有 IP
```

### 4.2 运行时配置（数据库存储）

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
```

---

## 五、配置覆盖示例

### 5.1 Docker Compose 配置示例

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

### 5.2 命令行覆盖示例

```bash
# 环境变量设置默认端口
export MEMOS_PORT=8080

# 命令行参数覆盖环境变量
memos --port 9000  # 实际使用 9000
```

### 5.3 优先级验证

```bash
# 场景：同时设置默认值、环境变量、命令行
# 默认: port=8081
export MEMOS_PORT=8080
memos --port 9000
# 结果：使用 9000（命令行优先级最高）
```

---

## 六、关键代码位置汇总

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
| API V1 服务 | `server/router/api/v1/v1.go` | `NewAPIV1Service()`: 50-65, `RegisterGateway()`: 68-168 |

---

## 七、注意事项

1. **配置文件不支持**：本项目未使用 Viper 的配置文件功能（如 config.yaml），仅支持环境变量和命令行参数

2. **Unix Socket 优先级**：设置 `unix-sock` 后，`addr` 和 `port` 会被忽略

3. **DSN 格式差异**：
   - SQLite: 文件路径（自动生成）
   - MySQL: `user:pass@tcp(host:port)/dbname`
   - PostgreSQL: `postgresql://user:pass@host:port/dbname`

4. **SecretKey 持久化**：首次启动生成的 SecretKey 存储在数据库中，后续启动会复用

5. **升级限制**：从 v0.22 之前的版本升级，必须先升级到 v0.25.3 作为中间版本

6. **数据目录权限**：自动创建的数据目录权限为 `0770`，确保进程有读写权限
