# Memos API Schema 与前端类型同步机制分析报告

## 1. 概述

Memos 项目采用 **Protocol Buffers (Protobuf)** 作为接口契约定义语言，通过 **Buf Build** 工具链实现从后端 API schema 到前端类型定义的代码生成。

**重要前提**：代码生成是**开发者本地手动触发**的，CI 仅负责 proto 文件的语法和格式检查，**不会自动生成代码**，也**不会验证生成代码与 proto 的同步性**。这是整个同步机制中最关键的边界和风险点。

## 2. 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           接口契约层 (Source of Truth)                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  proto/api/v1/*.proto                                                    │ │
│  │  - memo_service.proto    (备忘录服务定义)                                 │ │
│  │  - user_service.proto    (用户服务定义)                                   │ │
│  │  - auth_service.proto    (认证服务定义)                                   │ │
│  │  - common.proto          (公共类型定义)                                   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                        │
│                                        ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                    代码生成（开发者本地手动执行）                           │ │
│  │  命令: cd proto && buf generate                                          │ │
│  │  配置: buf.yaml + buf.gen.yaml                                            │ │
│  │  ⚠️  CI 不会执行此步骤 ⚠️                                                  │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                        │
│                    ┌───────────────────┼───────────────────┐                  │
│                    ▼                   ▼                   ▼                  │
┌─────────────────────────────┐ ┌─────────────────────┐ ┌─────────────────────────┐
│       后端生成代码           │ │   OpenAPI 文档      │ │      前端生成代码         │
│  proto/gen/api/v1/          │ │ proto/gen/openapi   │ │ web/src/types/proto/    │
│  - *.pb.go        (Protobuf)│ │      .yaml          │ │  - *_pb.ts    (类型定义) │
│  - *_grpc.pb.go   (gRPC)    │ │                     │ │  - 服务描述符             │
│  - apiv1connect/            │ │                     │ │                         │
│  - *.pb.gw.go     (Gateway) │ │                     │ │                         │
└─────────────────────────────┘ └─────────────────────┘ └─────────────────────────┘
│                                        │                                        │
▼                                        ▼                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    运行时集成层                                         │
│  ┌─────────────────────────────────────┐     ┌─────────────────────────────┐        │
│  │      后端服务实现（双层架构）         │     │      前端客户端调用          │        │
│  │                                     │     │                             │        │
│  │  ┌─────────────────────────────┐   │     │  web/src/connect.ts         │        │
│  │  │ APIV1Service                │   │     │  - 创建 Connect RPC 客户端   │        │
│  │  │ - 嵌入 v1pb.Unimplemented*  │   │     │  - 认证拦截器                │        │
│  │  │ - 实现 gRPC 业务逻辑         │   │     │                             │        │
│  │  │ - 例: CreateMemo()          │   │     │  web/src/hooks/*.ts         │        │
│  │  └─────────────────────────────┘   │     │  - useMemoQueries.ts         │        │
│  │              ▲                      │     │  - 使用生成的类型进行类型安全 │        │
│  │              │ 委托调用             │     │                             │        │
│  │  ┌─────────────────────────────┐   │     └─────────────────────────────┘        │
│  │  │ ConnectServiceHandler       │   │                                            │
│  │  │ - 包装 APIV1Service         │   │                                            │
│  │  │ - 实现 Connect handler 接口  │   │                                            │
│  │  │ - 转换 gRPC <-> Connect 错误 │   │                                            │
│  │  │ - 例: CreateMemo() 调用      │   │                                            │
│  │  │   s.APIV1Service.CreateMemo()│   │                                            │
│  │  └─────────────────────────────┘   │                                            │
│  │              │                      │                                            │
│  │              ▼                      │                                            │
│  │  server/router/api/v1/v1.go        │                                            │
│  │  - RegisterGateway()               │                                            │
│  │  - 注册 gRPC-Gateway + Connect     │                                            │
│  └─────────────────────────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 3. 核心文件分析

### 3.1 Protobuf Schema 定义 (`proto/api/v1/`)

#### 3.1.1 服务定义示例 (`memo_service.proto`)

```protobuf
syntax = "proto3";

package memos.api.v1;

import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "google/api/resource.proto";

option go_package = "gen/api/v1";

// MemoService 定义了备忘录相关的 API 接口
service MemoService {
  // CreateMemo 创建一个新备忘录
  rpc CreateMemo(CreateMemoRequest) returns (Memo) {
    option (google.api.http) = {
      post: "/api/v1/memos"
      body: "memo"
    };
  }
  // ... 更多方法
}
```

#### 3.1.2 关键设计特点

| 特性 | 说明 |
|------|------|
| **HTTP 路由映射** | 使用 `google.api.http` 注解定义 RESTful HTTP 端点，由 gRPC-Gateway 自动转换 |
| **字段行为** | `google.api.field_behavior` 标记字段为 `REQUIRED`、`OPTIONAL`、`OUTPUT_ONLY` 等 |
| **资源命名** | `google.api.resource` 定义资源的命名模式（如 `memos/{memo}`），符合 AIP 规范 |

### 3.2 Buf 构建配置

#### 3.2.1 `proto/buf.yaml` - 项目配置

```yaml
version: v2
deps:
  - buf.build/googleapis/googleapis
lint:
  use:
    - BASIC
  except:
    - ENUM_VALUE_PREFIX
    - FIELD_NOT_REQUIRED
    - PACKAGE_DIRECTORY_MATCH
    - PACKAGE_NO_IMPORT_CYCLE
    - PACKAGE_VERSION_SUFFIX
```

#### 3.2.2 `proto/buf.gen.yaml` - 代码生成配置（核心）

```yaml
version: v2
plugins:
  # 1. Go Protobuf 基础代码生成
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative

  # 2. Go gRPC 服务端/客户端代码
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative

  # 3. Go Connect RPC 代码 (Connect 协议)
  - remote: buf.build/connectrpc/go
    out: gen
    opt: paths=source_relative

  # 4. Go gRPC-Gateway 代码 (HTTP 反向代理)
  - remote: buf.build/grpc-ecosystem/gateway
    out: gen
    opt: paths=source_relative

  # 5. OpenAPI 文档生成
  - remote: buf.build/community/google-gnostic-openapi
    out: gen
    opt: enum_type=string

  # 6. TypeScript 代码生成（前端类型同步核心）
  - remote: buf.build/bufbuild/es
    out: ../web/src/types/proto  # 输出到前端源码目录
    opt:
      - target=ts
    include_imports: true
```

#### 3.2.3 插件输出对照表

| 插件 | 输出目录 | 生成内容 | 用途 |
|------|----------|----------|------|
| `protocolbuffers/go` | `proto/gen/api/v1/*.pb.go` | Protobuf 消息的 Go 类型定义 | 后端业务逻辑中的类型使用 |
| `grpc/go` | `proto/gen/api/v1/*_grpc.pb.go` | gRPC 服务端接口和客户端存根 | 定义 gRPC 服务接口（`Unimplemented*Server`） |
| `connectrpc/go` | `proto/gen/api/v1/apiv1connect/*.connect.go` | Connect RPC handler 注册函数 | 注册 Connect HTTP 处理器 |
| `grpc-ecosystem/gateway` | `proto/gen/api/v1/*.pb.gw.go` | gRPC-Gateway 反向代理代码 | HTTP 到 gRPC 的转换 |
| `bufbuild/es` | `web/src/types/proto/api/v1/*_pb.ts` | TypeScript 类型 + 服务描述符 | 前端类型安全、Connect 客户端 |

## 4. 前端类型同步机制

### 4.1 生成的 TypeScript 文件结构

```
web/src/types/proto/
├── api/
│   └── v1/
│       ├── memo_service_pb.ts       # Memo 服务的类型和描述符
│       ├── user_service_pb.ts       # User 服务的类型和描述符
│       ├── auth_service_pb.ts       # Auth 服务的类型和描述符
│       ├── common_pb.ts             # 公共类型
│       └── ...
└── google/
    └── api/
        └── ...
```

### 4.2 前端 Connect RPC 客户端配置

**文件**: `web/src/connect.ts`

```typescript
import { createClient, createConnectTransport } from "@connectrpc/connect-web";

// 导入生成的服务描述符（来自 proto 定义）
import { MemoService } from "./types/proto/api/v1/memo_service_pb";
import { UserService } from "./types/proto/api/v1/user_service_pb";
import { AuthService } from "./types/proto/api/v1/auth_service_pb";

// Transport 配置
const transport = createConnectTransport({
  baseUrl: window.location.origin,
  useBinaryFormat: true,  // 使用 Protobuf 二进制格式
  fetch: fetchWithCredentials,
  interceptors: [authInterceptor],
});

// 创建服务客户端 - 使用生成的服务描述符
export const memoServiceClient = createClient(MemoService, transport);
export const userServiceClient = createClient(UserService, transport);
export const authServiceClient = createClient(AuthService, transport);
```

### 4.3 React Query Hooks 中的类型使用

**文件**: `web/src/hooks/useMemoQueries.ts`

```typescript
import { create } from "@bufbuild/protobuf";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

// 导入生成的客户端
import { memoServiceClient } from "@/connect";

// 导入生成的类型定义（类型安全的核心）
import type { ListMemosRequest, Memo } from "@/types/proto/api/v1/memo_service_pb";
import { ListMemosRequestSchema, MemoSchema } from "@/types/proto/api/v1/memo_service_pb";

/**
 * 获取备忘录列表
 * @param request - 筛选条件，类型为 Partial<ListMemoRequest>（由 proto 生成）
 */
export function useMemos(request: Partial<ListMemosRequest> = {}) {
  return useQuery({
    queryKey: memoKeys.list(request),
    queryFn: async () => {
      // 使用 create() 函数创建类型安全的请求对象
      const response = await memoServiceClient.listMemos(
        create(ListMemosRequestSchema, request as Record<string, unknown>)
      );
      return response;
    },
  });
}

/**
 * 创建备忘录
 */
export function useCreateMemo() {
  const queryClient = useQueryClient();

  return useMutation({
    // 参数类型严格限定为 Memo（由 proto 生成）
    mutationFn: async (memoToCreate: Memo) => {
      const memo = await memoServiceClient.createMemo({ memo: memoToCreate });
      return memo;
    },
    onSuccess: (newMemo) => {
      // 类型推断：newMemo 自动为 Memo 类型
      queryClient.setQueryData(memoKeys.detail(newMemo.name), newMemo);
    },
  });
}
```

## 5. 后端服务实现（双层架构）

### 5.1 架构概览

后端采用 **双层架构** 同时支持 gRPC 和 Connect 两种协议：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           服务注册入口 (v1.go)                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  RegisterGateway() 方法                                                   │ │
│  │  ├── 注册 gRPC-Gateway (处理 /api/v1/*)                                  │ │
│  │  └── 注册 Connect handlers (处理 /memos.api.v1.*)                        │ │
│  │      └── connectHandler.RegisterConnectHandlers(connectMux, ...)         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   gRPC-Gateway      │   │   Connect 协议      │   │   (纯 gRPC 可选)     │
│   HTTP → gRPC       │   │   HTTP/1.1 + HTTP/2 │   │   HTTP/2 only        │
└──────────┬──────────┘   └──────────┬──────────┘   └──────────┬──────────┘
           │                         │                         │
           │                         ▼                         │
           │              ┌─────────────────────┐              │
           │              │ ConnectServiceHandler│              │
           │              │ - 实现 Connect      │              │
           │              │   handler 接口      │              │
           │              │ - 委托调用           │              │
           │              │   APIV1Service      │              │
           │              │ - 错误转换           │              │
           │              └──────────┬──────────┘              │
           │                         │                         │
           ▼                         ▼                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           APIV1Service (业务逻辑层)                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  嵌入生成的 Unimplemented 接口（来自 *_grpc.pb.go）                       │ │
│  │  - v1pb.UnimplementedMemoServiceServer                                   │ │
│  │  - v1pb.UnimplementedUserServiceServer                                   │ │
│  │  - ...                                                                   │ │
│  │                                                                         │ │
│  │  实现业务逻辑方法：                                                       │ │
│  │  func (s *APIV1Service) CreateMemo(                                      │ │
│  │    ctx context.Context,                                                  │ │
│  │    request *v1pb.CreateMemoRequest,  // 使用生成的类型                   │ │
│  │  ) (*v1pb.Memo, error)  // 返回生成的类型                                │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 第一层：APIV1Service（gRPC 接口实现）

**文件**: `server/router/api/v1/v1.go`

```go
package v1

import (
	v1pb "github.com/usememos/memos/proto/gen/api/v1"
)

// APIV1Service 实现了所有 gRPC 服务接口
// 通过嵌入生成的 Unimplemented*Server 结构体来"实现"接口
type APIV1Service struct {
	// 嵌入生成的 Unimplemented 接口（来自 *_grpc.pb.go）
	// 这些结构体包含所有方法的默认实现（返回 Unimplemented 错误）
	v1pb.UnimplementedInstanceServiceServer
	v1pb.UnimplementedAuthServiceServer
	v1pb.UnimplementedUserServiceServer
	v1pb.UnimplementedMemoServiceServer
	v1pb.UnimplementedAttachmentServiceServer
	v1pb.UnimplementedAIServiceServer
	v1pb.UnimplementedShortcutServiceServer
	v1pb.UnimplementedIdentityProviderServiceServer

	// 业务依赖
	Secret          string
	Store           *store.Store
	MarkdownService markdown.Service
	// ...
}
```

**文件**: `server/router/api/v1/memo_service.go`（业务逻辑实现）

```go
package v1

import (
	"context"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"

	// 导入生成的 API 类型
	v1pb "github.com/usememos/memos/proto/gen/api/v1"
	"github.com/usememos/memos/store"
)

// CreateMemo 实现了 v1pb.MemoServiceServer 接口
// 注意：参数和返回值都是生成的 Protobuf 类型
func (s *APIV1Service) CreateMemo(
	ctx context.Context,
	request *v1pb.CreateMemoRequest,  // 生成的请求类型
) (*v1pb.Memo, error) {  // 生成的响应类型
	user, err := s.fetchCurrentUser(ctx)
	if err != nil {
		return nil, status.Errorf(codes.Internal, "failed to get user")
	}

	// 使用生成的类型字段
	memoUID, err := ValidateAndGenerateUID(request.MemoId)
	if err != nil {
		return nil, err
	}

	// 转换 API 类型到内部 Store 类型
	create := &store.Memo{
		UID:        memoUID,
		CreatorID:  user.ID,
		Content:    request.Memo.Content,    // 使用生成的 request.Memo.Content
		Visibility: convertVisibilityToStore(request.Memo.Visibility),
	}

	// ... 业务逻辑

	memo, err := s.Store.CreateMemo(ctx, create)
	if err != nil {
		return nil, err
	}

	// 转换 Store 类型回生成的 API 类型
	return convertMemoFromStore(memo), nil
}
```

### 5.3 第二层：ConnectServiceHandler（Connect 协议适配）

**文件**: `server/router/api/v1/connect_handler.go`

```go
package v1

import (
	"net/http"
	"connectrpc.com/connect"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"

	// 导入生成的 Connect 注册函数
	"github.com/usememos/memos/proto/gen/api/v1/apiv1connect"
)

// ConnectServiceHandler 包装 APIV1Service，实现 Connect handler 接口
// 它的职责是：
// 1. 适配 Connect 的请求/响应包装类型（connect.Request[T] / connect.Response[T]）
// 2. 转换 gRPC 错误到 Connect 错误
// 3. 委托实际业务逻辑给 APIV1Service
type ConnectServiceHandler struct {
	*APIV1Service  // 嵌入 APIV1Service，可直接访问其方法
}

// NewConnectServiceHandler 创建一个新的 Connect 服务处理器
func NewConnectServiceHandler(svc *APIV1Service) *ConnectServiceHandler {
	return &ConnectServiceHandler{APIV1Service: svc}
}

// RegisterConnectHandlers 使用生成的 apiv1connect 包注册所有服务
// 这是连接层注册的核心方法
func (s *ConnectServiceHandler) RegisterConnectHandlers(
	mux *http.ServeMux,
	opts ...connect.HandlerOption,
) {
	// 使用生成的注册函数注册每个服务
	// 注意：包名是 apiv1connect（由 buf.gen.yaml 中的 go_package_prefix 决定）
	handlers := []struct {
		path    string
		handler http.Handler
	}{
		// 生成的函数签名：
		// func NewMemoServiceHandler(handler MemoServiceHandler, opts ...connect.HandlerOption) (string, http.Handler)
		wrap(apiv1connect.NewInstanceServiceHandler(s, opts...)),
		wrap(apiv1connect.NewAuthServiceHandler(s, opts...)),
		wrap(apiv1connect.NewUserServiceHandler(s, opts...)),
		wrap(apiv1connect.NewMemoServiceHandler(s, opts...)),      // ⬅️ 关键：s 必须实现 MemoServiceHandler 接口
		wrap(apiv1connect.NewAttachmentServiceHandler(s, opts...)),
		wrap(apiv1connect.NewAIServiceHandler(s, opts...)),
		wrap(apiv1connect.NewShortcutServiceHandler(s, opts...)),
		wrap(apiv1connect.NewIdentityProviderServiceHandler(s, opts...)),
	}

	// 注册到 HTTP mux
	for _, h := range handlers {
		mux.Handle(h.path, h.handler)
	}
}

// wrap 辅助函数：将 (path, handler) 转换为结构体
func wrap(path string, handler http.Handler) struct {
	path    string
	handler http.Handler
} {
	return struct {
		path    string
		handler http.Handler
	}{path, handler}
}

// convertGRPCError 将 gRPC status 错误转换为 Connect 错误
// 这是必需的，因为 APIV1Service 返回的是 gRPC 风格的错误
// 而 Connect 协议需要 connect.Error 类型
func convertGRPCError(err error) error {
	if err == nil {
		return nil
	}
	if st, ok := status.FromError(err); ok {
		// gRPC 和 Connect 使用相同的错误码语义，直接转换
		return connect.NewError(grpcCodeToConnectCode(st.Code()), err)
	}
	return connect.NewError(connect.CodeInternal, err)
}

func grpcCodeToConnectCode(code codes.Code) connect.Code {
	return connect.Code(code)
}
```

### 5.4 ConnectServiceHandler 方法实现（委托模式）

**文件**: `server/router/api/v1/connect_services.go`

```go
package v1

import (
	"context"
	"net/http"
	"connectrpc.com/connect"
	"google.golang.org/protobuf/types/known/emptypb"

	// 导入生成的 API 类型
	v1pb "github.com/usememos/memos/proto/gen/api/v1"
)

// 此文件包含所有 Connect handler 方法的实现
// 每个方法都遵循相同的模式：
// 1. 接收 connect.Request[T] 类型的参数
// 2. 提取内部的 Msg（即生成的 Protobuf 类型）
// 3. 委托调用 APIV1Service 的对应方法
// 4. 用 connect.NewResponse() 包装返回值
// 5. 转换错误类型

// ============================================================================
// MemoService 方法示例
// ============================================================================

// CreateMemo 实现了 apiv1connect.MemoServiceHandler 接口
//
// 注意参数类型差异：
// - gRPC 接口: CreateMemo(ctx, *v1pb.CreateMemoRequest) (*v1pb.Memo, error)
// - Connect 接口: CreateMemo(ctx, *connect.Request[v1pb.CreateMemoRequest]) (*connect.Response[v1pb.Memo], error)
func (s *ConnectServiceHandler) CreateMemo(
	ctx context.Context,
	req *connect.Request[v1pb.CreateMemoRequest],  // Connect 的请求包装类型
) (*connect.Response[v1pb.Memo], error) {  // Connect 的响应包装类型
	// 委托调用 APIV1Service 的 gRPC 方法
	// req.Msg 是内部的实际请求对象（*v1pb.CreateMemoRequest）
	resp, err := s.APIV1Service.CreateMemo(ctx, req.Msg)
	if err != nil {
		// 转换 gRPC 错误到 Connect 错误
		return nil, convertGRPCError(err)
	}
	// 用 connect.NewResponse() 包装返回值
	return connect.NewResponse(resp), nil
}

// ListMemos - 另一个示例
func (s *ConnectServiceHandler) ListMemos(
	ctx context.Context,
	req *connect.Request[v1pb.ListMemosRequest],
) (*connect.Response[v1pb.ListMemosResponse], error) {
	resp, err := s.APIV1Service.ListMemos(ctx, req.Msg)
	if err != nil {
		return nil, convertGRPCError(err)
	}
	return connect.NewResponse(resp), nil
}

// GetMemo
func (s *ConnectServiceHandler) GetMemo(
	ctx context.Context,
	req *connect.Request[v1pb.GetMemoRequest],
) (*connect.Response[v1pb.Memo], error) {
	resp, err := s.APIV1Service.GetMemo(ctx, req.Msg)
	if err != nil {
		return nil, convertGRPCError(err)
	}
	return connect.NewResponse(resp), nil
}

// UpdateMemo
func (s *ConnectServiceHandler) UpdateMemo(
	ctx context.Context,
	req *connect.Request[v1pb.UpdateMemoRequest],
) (*connect.Response[v1pb.Memo], error) {
	resp, err := s.APIV1Service.UpdateMemo(ctx, req.Msg)
	if err != nil {
		return nil, convertGRPCError(err)
	}
	return connect.NewResponse(resp), nil
}

// DeleteMemo - 返回 emptypb.Empty
func (s *ConnectServiceHandler) DeleteMemo(
	ctx context.Context,
	req *connect.Request[v1pb.DeleteMemoRequest],
) (*connect.Response[emptypb.Empty], error) {
	resp, err := s.APIV1Service.DeleteMemo(ctx, req.Msg)
	if err != nil {
		return nil, convertGRPCError(err)
	}
	return connect.NewResponse(resp), nil
}

// ============================================================================
// AuthService 特殊处理（需要操作 HTTP 响应头）
// ============================================================================

// AuthService 的方法需要特殊处理，因为它们需要设置 Cookie（HTTP 响应头）
// 使用 connectWithHeaderCarrier 辅助函数将 header carrier 注入 context

func (s *ConnectServiceHandler) SignIn(
	ctx context.Context,
	req *connect.Request[v1pb.SignInRequest],
) (*connect.Response[v1pb.SignInResponse], error) {
	// 使用特殊的 header carrier 处理
	return connectWithHeaderCarrier(ctx, func(ctx context.Context) (*v1pb.SignInResponse, error) {
		return s.APIV1Service.SignIn(ctx, req.Msg)
	})
}

func (s *ConnectServiceHandler) SignOut(
	ctx context.Context,
	req *connect.Request[v1pb.SignOutRequest],
) (*connect.Response[emptypb.Empty], error) {
	return connectWithHeaderCarrier(ctx, func(ctx context.Context) (*emptypb.Empty, error) {
		return s.APIV1Service.SignOut(ctx, req.Msg)
	})
}

func (s *ConnectServiceHandler) RefreshToken(
	ctx context.Context,
	req *connect.Request[v1pb.RefreshTokenRequest],
) (*connect.Response[v1pb.RefreshTokenResponse], error) {
	return connectWithHeaderCarrier(ctx, func(ctx context.Context) (*v1pb.RefreshTokenResponse, error) {
		return s.APIV1Service.RefreshToken(ctx, req.Msg)
	})
}

// ============================================================================
// 其他服务方法（UserService, AttachmentService 等）
// 都遵循相同的委托模式
// ============================================================================

// UserService 示例
func (s *ConnectServiceHandler) ListUsers(
	ctx context.Context,
	req *connect.Request[v1pb.ListUsersRequest],
) (*connect.Response[v1pb.ListUsersResponse], error) {
	resp, err := s.APIV1Service.ListUsers(ctx, req.Msg)
	if err != nil {
		return nil, convertGRPCError(err)
	}
	return connect.NewResponse(resp), nil
}

func (s *ConnectServiceHandler) GetUser(
	ctx context.Context,
	req *connect.Request[v1pb.GetUserRequest],
) (*connect.Response[v1pb.User], error) {
	resp, err := s.APIV1Service.GetUser(ctx, req.Msg)
	if err != nil {
		return nil, convertGRPCError(err)
	}
	return connect.NewResponse(resp), nil
}

// ... 其他服务的方法实现遵循相同模式
```

### 5.5 服务注册入口

**文件**: `server/router/api/v1/v1.go`（RegisterGateway 方法节选）

```go
// RegisterGateway 注册 gRPC-Gateway 和 Connect handlers
func (s *APIV1Service) RegisterGateway(ctx context.Context, echoServer *echo.Echo) error {
	// =========================================================================
	// 1. 注册 gRPC-Gateway（处理 /api/v1/* 路径）
	// =========================================================================
	gwMux := runtime.NewServeMux(
		runtime.WithMiddlewares(gatewayAuthMiddleware),
	)
	// 使用生成的 Register*HandlerServer 函数注册 gRPC-Gateway
	if err := v1pb.RegisterMemoServiceHandlerServer(ctx, gwMux, s); err != nil {
		return err
	}
	if err := v1pb.RegisterUserServiceHandlerServer(ctx, gwMux, s); err != nil {
		return err
	}
	// ... 其他服务注册

	// 注册到 Echo
	gwGroup := echoServer.Group("")
	gwGroup.Any("/api/v1/*", echo.WrapHandler(gwMux))

	// =========================================================================
	// 2. 注册 Connect handlers（处理 /memos.api.v1.* 路径）
	// =========================================================================

	// 创建拦截器
	connectInterceptors := connect.WithInterceptors(
		NewMetadataInterceptor(),
		NewLoggingInterceptor(logStacktraces),
		NewRecoveryInterceptor(logStacktraces),
		NewAuthInterceptor(s.Store, s.Secret),
	)

	// 创建 HTTP mux
	connectMux := http.NewServeMux()

	// ⬅️ 关键：创建 ConnectServiceHandler 并注册
	connectHandler := NewConnectServiceHandler(s)
	connectHandler.RegisterConnectHandlers(
		connectMux,
		connectInterceptors,
		connect.WithReadMaxBytes(maxAPIRequestBytes),
	)

	// 注册到 Echo（路径格式：/memos.api.v1.MemoService/CreateMemo）
	connectGroup := echoServer.Group("", corsHandler)
	connectGroup.Any("/memos.api.v1.*", echo.WrapHandler(connectMux))

	return nil
}
```

### 5.6 两层架构总结

| 层级 | 结构体 | 职责 | 接口来源 | 关键特征 |
|------|--------|------|----------|----------|
| **第一层** | `APIV1Service` | 实现业务逻辑 | `v1pb.Unimplemented*Server`（来自 `*_grpc.pb.go`） | 参数/返回值为原生 Protobuf 类型（`*v1pb.CreateMemoRequest`） |
| **第二层** | `ConnectServiceHandler` | 适配 Connect 协议 | `apiv1connect.*Handler`（来自 `*.connect.go`） | 参数/返回值为 Connect 包装类型（`*connect.Request[T]` / `*connect.Response[T]`） |
| **注册** | `RegisterGateway` | 统一注册入口 | - | 同时注册 gRPC-Gateway 和 Connect |

**关键设计决策**：
1. **业务逻辑只写一次**：在 `APIV1Service` 中实现，被两种协议共享
2. **错误转换统一处理**：`convertGRPCError` 确保错误码语义一致
3. **双协议支持**：gRPC-Gateway 提供 RESTful HTTP，Connect 提供浏览器友好的 RPC
4. **生成代码驱动**：所有接口和注册函数都由 `buf generate` 生成

## 6. CI/CD 边界