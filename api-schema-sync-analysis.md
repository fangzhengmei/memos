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

## 6. CI/CD 边界与职责划分

### 6.1 关键边界声明

**重要区分**：

| 职责 | 本地开发者 | CI 流水线 |
|------|-----------|-----------|
| 修改 proto 文件 | ✅ | ❌ |
| 运行 `buf generate` | ✅ **必需** | ❌ **不会执行** |
| 提交生成的代码 | ✅ **必需** | ❌ |
| 检查 proto 语法（buf lint） | 建议执行 | ✅ 强制执行 |
| 检查 proto 格式（buf format） | 建议执行 | ✅ 强制执行 |
| 检查 Go 代码编译 | 建议执行 | ✅ 强制执行 |
| 检查 TypeScript 类型 | 建议执行 | ✅ 强制执行 |
| **验证生成代码与 proto 同步** | 人工检查 | ❌ **无检查** |

### 6.1.1 本地手动生成 vs CI 校验：能发现什么，漏掉什么

这是整个同步机制中最关键的边界。以下是清晰的能力对比：

#### 本地手动生成（开发者执行 `buf generate`）

| 检查项 | 能发现 | 漏掉 |
|--------|--------|------|
| **Proto 语法错误** | ❌ 不检查（需单独运行 `buf lint`） | - |
| **Proto 格式问题** | ❌ 不检查（需单独运行 `buf format`） | - |
| **生成代码与 proto 同步** | ✅ 确保同步（直接覆盖生成） | - |
| **Go 类型编译** | ✅ 生成合法的 Go 代码 | ❌ 不检查业务逻辑是否正确使用新类型 |
| **TypeScript 类型编译** | ✅ 生成合法的 TS 代码 | ❌ 不检查前端代码是否正确使用新类型 |

**本地生成的核心作用**：将 proto 定义转换为可编译的代码。但它**不验证**：
- 业务逻辑是否正确使用新字段
- 破坏性变更的影响范围
- 前后端使用一致性

#### CI 校验（现有实现）

| 检查项 | 能发现 | 漏掉 |
|--------|--------|------|
| **Proto 语法错误** | ✅ `buf lint` | - |
| **Proto 格式问题** | ✅ `buf format` 检查 | - |
| **Go 代码编译** | ✅ `go build` | ❌ 不检查生成代码是否与 proto 同步 |
| **TypeScript 类型** | ✅ `tsc --noEmit` | ❌ 不检查生成代码是否与 proto 同步 |
| **生成代码与 proto 同步** | ❌ **完全漏掉** | - |
| **破坏性变更** | ❌ **完全漏掉** | - |

**最危险的遗漏**：CI 无法发现以下场景：

```
场景：开发者修改 proto 添加新字段 Location，
      但忘记运行 buf generate，直接提交 PR。

CI 检查结果：
├── ✅ buf lint 通过（proto 语法正确）
├── ✅ buf format 通过（proto 格式正确）
├── ✅ go build 通过（使用仓库中旧的生成代码编译）
├── ✅ pnpm lint 通过（使用仓库中旧的生成代码编译）
└── ❌ 完全没发现：proto 与生成代码不同步！

后果：
- 后端运行时：proto 定义有 Location，但生成代码没有
- 前端运行时：完全不知道有 Location 字段
- 运行时行为不一致，但 CI 全部通过
```

#### 对比总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  本地生成（buf generate）能保证什么？                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  ✅ 保证 proto 到生成代码的转换是完整的                                       │
│  ✅ 保证生成的代码语法正确（可编译）                                           │
│  ❌ 不保证 proto 本身语法正确（需 buf lint）                                  │
│  ❌ 不保证业务逻辑正确使用新类型                                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  CI 校验（现有）能保证什么？                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ✅ 保证 proto 语法正确                                                       │
│  ✅ 保证 proto 格式规范                                                       │
│  ✅ 保证代码能编译通过                                                         │
│  ❌ **完全不能保证** proto 与生成代码的同步性！                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 CI 工作流分析

#### 6.2.1 proto-linter.yml - 仅检查 proto 源文件

```yaml
name: Proto Linter

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths:
      - "proto/**"

jobs:
  lint:
    steps:
      # 1. 运行 buf lint - 检查 proto 语法和风格
      - name: Run buf lint
        uses: bufbuild/buf-lint-action@v1
        with:
          input: proto

      # 2. 检查 buf format - 检查 proto 文件格式
      - name: Check buf format
        run: |
          if [[ $(buf format -d) ]]; then
            echo "❌ Proto files are not formatted."
            exit 1
          fi
```

**这个 CI 检查的范围**：
- ✅ 检查 `.proto` 文件的语法正确性
- ✅ 检查 `.proto` 文件的格式规范性
- ❌ **不检查** 生成的代码（`proto/gen/`, `web/src/types/proto/`）
- ❌ **不运行** `buf generate`
- ❌ **不验证** 生成代码与 proto 的一致性

#### 6.2.2 backend-tests.yml - 检查 Go 代码

```yaml
name: Backend Tests

on:
  paths:
    - "go.mod"
    - "go.sum"
    - "**.go"  # 包括生成的 .pb.go 文件

jobs:
  static-checks:
    steps:
      - name: Run golangci-lint
        uses: golangci-lint-action@v9

  tests:
    steps:
      - name: Run tests
        run: go test ./...
```

**这个 CI 检查的范围**：
- ✅ 检查 Go 代码能否编译通过
- ✅ 运行 Go 测试
- ❌ **不检查** 生成的 Go 代码是否与 proto 同步
- ❌ 即使 proto 修改了但生成代码没更新，只要 Go 代码能编译就通过

#### 6.2.3 frontend-tests.yml - 检查 TypeScript 代码

```yaml
name: Frontend Tests

on:
  paths:
    - "web/**"  # 包括生成的 *_pb.ts 文件

jobs:
  lint:
    steps:
      - name: Run lint
        working-directory: web
        run: pnpm lint  # tsc --noEmit + biome check
```

**这个 CI 检查的范围**：
- ✅ 检查 TypeScript 类型能否通过编译
- ❌ **不检查** 生成的 TypeScript 代码是否与 proto 同步
- ❌ 即使 proto 修改了但生成代码没更新，只要 TS 代码能编译就通过

### 6.3 实际同步流程（开发者端）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         开发者端同步流程                                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  1. 修改 Protobuf Schema                                                       │
│     文件: proto/api/v1/memo_service.proto                                     │
│     示例: 添加新字段 `optional Location location = 18;`                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  2. ⚠️ 本地手动运行代码生成 ⚠️                                                  │
│     命令: cd proto && buf generate                                             │
│                                                                                 │
│     这是唯一的同步点！                                                          │
│     如果忘记这一步，生成代码与 proto 就会失配                                   │
│                                                                                 │
│     生成内容:                                                                    │
│     ├── proto/gen/api/v1/memo_service.pb.go          (更新)                   │
│     ├── proto/gen/api/v1/apiv1connect/memo_service.connect.go (更新)          │
│     └── web/src/types/proto/api/v1/memo_service_pb.ts (更新)                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  3. 本地检查（可选但推荐）                                                      │
│     ├── buf lint          # 检查 proto 语法                                    │
│     ├── buf format -w     # 格式化 proto 文件                                  │
│     ├── go build ./...   # 检查 Go 编译                                       │
│     └── cd web && pnpm lint  # 检查 TypeScript 类型                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  4. 提交所有相关文件                                                            │
│     需要同时提交：                                                              │
│     ├── ✅ 修改的 proto/api/v1/*.proto                                        │
│     ├── ✅ 重新生成的 proto/gen/**                                             │
│     └── ✅ 重新生成的 web/src/types/proto/**                                   │
│                                                                                 │
│     ⚠️ 危险：如果只提交 .proto 但不提交生成代码，就会产生同步失配！             │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  5. PR CI 检查                                                                 │
│     CI 会执行：                                                                 │
│     ├── buf lint          # ✅ 通过（proto 没问题）                           │
│     ├── buf format        # ✅ 通过（proto 格式化了）                         │
│     ├── go build          # ✅ 通过（如果生成代码也提交了）                    │
│     └── pnpm lint         # ✅ 通过（如果生成代码也提交了）                    │
│                                                                                 │
│     ⚠️ 注意：CI 不会检查生成代码是否与 proto 匹配！                             │
│        如果开发者提交了新 proto 但忘记提交生成代码，                           │
│        CI 仍然可能通过（使用旧的生成代码）                                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 7. 同步失配风险分析

### 7.1 什么是同步失配？

**同步失配** 指：**Protobuf Schema 定义** 与 **生成的代码** 之间不一致。

由于 CI **不会**验证这种一致性，完全依赖开发者手动执行 `buf generate` 并提交所有生成文件，失配风险是真实存在的。

### 7.2 典型失配场景

#### 场景 1：修改 proto 但忘记运行 buf generate

```
开发者操作:
1. 修改 proto/api/v1/memo_service.proto，添加新字段 Location
2. 直接提交 .proto 文件
3. 忘记运行 buf generate
4. 忘记提交生成的代码

结果:
- ✅ CI: buf lint 通过（proto 语法正确）
- ✅ CI: go build 通过（使用旧的生成代码编译）
- ✅ CI: pnpm lint 通过（使用旧的生成代码编译）
- ❌ 运行时: 新字段不会生效，前后端都不知道有这个字段
```

#### 场景 2：运行了 buf generate 但只提交部分文件

```
开发者操作:
1. 修改 proto/api/v1/memo_service.proto
2. 运行 buf generate
3. 只提交 .proto 和 Go 生成代码
4. 忘记提交前端的 web/src/types/proto/*.ts

结果:
- ✅ 后端: 使用新字段
- ❌ 前端: 仍使用旧类型，不知道新字段存在
- ❌ 运行时: 可能出现序列化/反序列化问题
```

#### 场景 3：多人协作时的生成代码冲突

```
场景:
1. 开发者 A 修改 proto 并生成代码，提交 PR
2. 开发者 B 同时修改另一个 proto 并生成代码，提交 PR
3. 其中一个 PR 合并后，另一个 PR 的生成代码可能冲突

风险:
- 生成代码是机器生成的，手动解决冲突容易出错
- 可能引入细微的类型错误
```

#### 场景 4：proto 破坏性变更但生成代码未同步

```
开发者操作:
1. 在 proto 中重命名字段（破坏性行为）
2. 忘记运行 buf generate
3. 提交代码

结果:
- ✅ CI: 通过（使用旧生成代码）
- ❌ 运行时: 字段不匹配，可能导致数据丢失或错误
```

### 7.3 失配的影响层级

| 失配类型 | 编译时（Go/TS） | 运行时 | 影响程度 |
|----------|-----------------|--------|----------|
| 新增字段 | ✅ 无感知（使用旧生成代码） | 后端收到但前端不知道 | 中等 |
| 删除字段 | ✅ 无感知 | 可能 panic 或数据错误 | 高 |
| 修改字段类型 | ✅ 无感知 | 序列化/反序列化失败 | 高 |
| 修改方法签名 | ✅ 无感知 | RPC 调用失败 | 高 |
| 重命名消息 | ❌ 编译失败（如果业务代码引用） | - | 低（早发现） |

**最危险的情况**：新增/删除字段但业务代码不直接引用消息类型。这种情况下 CI 完全通过，但运行时行为不一致。

## 8. 防错建议与最佳实践

### 8.1 本地开发流程强化

#### 建议 1：使用 Makefile 或脚本封装常用操作

```makefile
# 建议添加到项目根目录的 Makefile

.PHONY: proto
proto:  # 一键生成 + 格式化
	cd proto && buf generate && buf format -w

.PHONY: proto-check
proto-check:  # 检查 proto 是否需要生成或格式化
	cd proto && buf lint && buf format -d

.PHONY: generate
generate: proto  # 生成后检查类型
	go build ./...
	cd web && pnpm lint
```

**使用方式**：
```bash
# 修改 proto 后，运行：
make generate

# 这会：
# 1. buf generate（生成代码）
# 2. buf format -w（格式化 proto）
# 3. go build（检查 Go 编译）
# 4. pnpm lint（检查 TypeScript）
```

#### 建议 2：配置 Git Pre-Commit Hook

创建 `.git/hooks/pre-commit`（或使用 [pre-commit](https://pre-commit.com/) 框架）：

```bash
#!/bin/bash

# 检查 proto 文件是否有变更但生成代码未更新
proto_files=$(git diff --cached --name-only -- 'proto/api/v1/*.proto')
if [ -n "$proto_files" ]; then
    echo "⚠️  检测到 proto 文件变更，检查生成代码..."
    
    # 检查生成的 Go 代码是否也在暂存区
    gen_go_files=$(git diff --cached --name-only -- 'proto/gen/**')
    gen_ts_files=$(git diff --cached --name-only -- 'web/src/types/proto/**')
    
    if [ -z "$gen_go_files" ] && [ -z "$gen_ts_files" ]; then
        echo "❌ 错误：proto 文件已修改，但生成代码未提交！"
        echo "   请运行: cd proto && buf generate"
        echo "   然后提交: proto/gen/ 和 web/src/types/proto/"
        exit 1
    fi
fi

# 检查 buf lint
if command -v buf &> /dev/null; then
    if ! cd proto && buf lint; then
        echo "❌ proto lint 检查失败"
        exit 1
    fi
fi

exit 0
```

### 8.2 CI 流程增强建议

当前 CI **不会**检查生成代码与 proto 的同步性。可以考虑增强：

#### 建议 1：添加生成代码同步性检查

```yaml
# 建议新增或修改 .github/workflows/proto-linter.yml

jobs:
  lint:
    name: Lint Protos
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Setup buf
        uses: bufbuild/buf-setup-action@v1

      # 原有检查
      - name: Run buf lint
        uses: bufbuild/buf-lint-action@v1
        with:
          input: proto

      - name: Check buf format
        run: |
          if [[ $(buf format -d) ]]; then
            echo "❌ Proto files are not formatted."
            exit 1
          fi

      # ⬇️ 新增：检查生成代码是否同步 ⬇️
      - name: Check if generated code is in sync
        run: |
          # 1. 保存当前生成代码的状态
          tar -cf /tmp/gen-before.tar proto/gen web/src/types/proto

          # 2. 重新生成代码
          cd proto && buf generate

          # 3. 比较差异
          if ! git diff --quiet proto/gen web/src/types/proto; then
            echo "❌ 生成代码与 proto 不同步！"
            echo "   请在本地运行: cd proto && buf generate"
            echo "   然后提交重新生成的代码"
            echo ""
            echo "   差异详情："
            git diff proto/gen web/src/types/proto
            exit 1
          fi

          echo "✅ 生成代码与 proto 同步"
```

**这个检查的效果**：
- 如果开发者只修改了 proto 但忘记运行 `buf generate`，CI 会失败
- 如果开发者运行了 `buf generate` 但忘记提交生成文件，CI 会失败
- 确保 proto 与生成代码永远同步

#### 建议 2：为 proto 相关文件设置 CODEOWNERS

在 `.github/CODEOWNERS` 中添加：
```
# 确保 proto 相关变更需要特定人员审查
proto/api/v1/*.proto    @api-owners
proto/gen/              @api-owners
web/src/types/proto/    @api-owners
```

### 8.3 代码审查 Checklist

在 PR 审查时，检查以下项目：

```
📋 API 变更审查清单

当 PR 包含 proto 文件变更时：

1. [ ] proto 文件语法是否正确？
   - 可通过本地 buf lint 验证

2. [ ] 生成的代码是否同时更新？
   - 检查 proto/gen/ 目录
   - 检查 web/src/types/proto/ 目录

3. [ ] 字段编号是否正确使用？
   - 新增字段使用新的字段号
   - 删除字段使用 reserved 标记

4. [ ] 是否有破坏性变更？
   - 重命名字段？
   - 修改字段类型？
   - 删除必需字段？

5. [ ] 后端实现是否更新？
   - 服务实现是否处理新字段？
   - 数据库迁移是否同步？

6. [ ] 前端使用是否更新？
   - Hooks 是否使用新字段？
   - UI 是否展示新字段？
```

### 8.4 破坏性变更管理

#### 必须使用 reserved 关键字

当删除字段或消息时，必须使用 `reserved` 标记：

```protobuf
message Memo {
  string name = 1;
  string content = 7;
  
  // ❌ 错误：直接删除字段，编号可能被重用
  // reserved 6;  // 原来的 display_time 字段已删除
  
  // ✅ 正确：使用 reserved 标记已删除的字段编号和名称
  reserved 6;
  reserved "display_time";
}
```

#### 破坏性变更检查

使用 `buf breaking` 检查破坏性变更：

```bash
# 检查相对于 main 分支的破坏性变更
cd proto && buf breaking --against '.git#branch=main'
```

可以在 CI 或 pre-commit hook 中添加此检查。

### 8.5 最佳实践总结

| 实践 | 目的 | 实施方式 |
|------|------|----------|
| **封装生成命令** | 减少手动操作错误 | Makefile / npm scripts |
| **Pre-commit Hook** | 提交前强制检查 | Git hooks / pre-commit 框架 |
| **CI 同步检查** | 确保代码库一致性 | 新增 CI 步骤（建议） |
| **CODEOWNERS** | 确保专业人员审查 | 配置 proto 目录所有者 |
| **审查 Checklist** | 人工检查关键点 | PR 模板 |
| **reserved 关键字** | 防止字段编号冲突 | proto 文件规范 |
| **buf breaking** | 检测破坏性变更 | CI / pre-commit |

## 9. 同步机制流程图（修正版）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         完整同步流程图                                          │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 1: 本地开发（开发者负责）                                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐                                                            │
│  │ 修改 .proto  │                                                            │
│  └──────┬───────┘                                                            │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ ⚠️  关键点：必须手动运行 buf generate                                    │  │
│  │                                                                    │  │
│  │  开发者执行: cd proto && buf generate                               │  │
│  │                                                                    │  │
│  │  输出:                                                              │  │
│  │  ├── proto/gen/api/v1/*.pb.go          (Go 类型)                  │  │
│  │  ├── proto/gen/api/v1/apiv1connect/   (Connect 注册)              │  │
│  │  └── web/src/types/proto/api/v1/*_pb.ts (TS 类型)                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 本地检查（推荐）                                                        │  │
│  │ ├── buf lint              # 检查 proto 语法                           │  │
│  │ ├── buf format -w         # 格式化 proto                               │  │
│  │ ├── go build ./...        # 检查 Go 编译                              │  │
│  │ ├── buf breaking --against # 检查破坏性变更                            │  │
│  │ └── cd web && pnpm lint   # 检查 TypeScript                            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ ⚠️  关键点：必须同时提交所有相关文件                                      │  │
│  │                                                                    │  │
│  │ git add:                                                            │  │
│  │ ├── ✅ proto/api/v1/*.proto          (源文件)                       │  │
│  │ ├── ✅ proto/gen/**                   (生成的 Go 代码)               │  │
│  │ └── ✅ web/src/types/proto/**          (生成的 TS 代码)               │  │
│  │                                                                    │  │
│  │ ⚠️  危险：只提交 .proto 但不提交生成代码 = 同步失配！                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 2: CI 检查（当前实现）                                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  当前 CI 检查范围：                                                            │
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │ proto-linter.yml│    │ backend-tests   │    │ frontend-tests  │        │
│  ├─────────────────┤    ├─────────────────┤    ├─────────────────┤        │
│  │ ✅ buf lint     │    │ ✅ go build     │    │ ✅ tsc --noEmit │        │
│  │ ✅ buf format   │    │ ✅ go test      │    │ ✅ biome check   │        │
│  │                 │    │ ✅ golangci-lint│    │                 │        │
│  │ ❌ 检查同步性   │    │ ❌ 检查同步性   │    │ ❌ 检查同步性   │        │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘        │
│                                                                              │
│  ⚠️  关键限制：当前 CI 不会检查生成代码是否与 proto 同步！                     │
│                                                                              │
│  可能通过但实际失配的情况：                                                    │
│  - 开发者修改了 proto 但忘记运行 buf generate                                │
│  - 开发者运行了 buf generate 但只提交了部分生成文件                           │
│  - CI 仍然通过（使用仓库中旧的生成代码编译）                                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 3: 建议增强（可选但推荐）                                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  建议添加的 CI 检查：                                                          │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 新增：生成代码同步性检查                                                 │  │
│  │                                                                    │  │
│  │ 步骤:                                                               │  │
│  │ 1. CI 中运行 buf generate                                           │  │
│  │ 2. 比较生成的代码与仓库中的代码                                       │  │
│  │ 3. 如果有差异，CI 失败                                               │  │
│  │                                                                    │  │
│  │ 效果:                                                               │  │
│  │ - 确保 proto 与生成代码永远同步                                      │  │
│  │ - 开发者必须在本地运行 buf generate 并提交所有文件                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  建议添加的 pre-commit hook：                                                 │
│  ├── 检查 proto 变更时生成代码是否也提交                                      │
│  └── 运行 buf lint 和 buf breaking 检查                                      │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 10. 修正后的总结

### 10.1 真实同步机制

Memos 项目的 API 类型同步机制：

1. **契约优先**：以 `proto/api/v1/*.proto` 作为唯一事实来源
2. **本地生成**：开发者手动执行 `cd proto && buf generate`
3. **手动提交**：开发者必须同时提交 `.proto` 和生成的代码
4. **CI 有限检查**：CI 仅检查 proto 语法和代码编译，**不检查同步性**

### 10.2 关键风险点

| 风险 | 可能性 | 影响 | 当前防护 | 建议防护 |
|------|--------|------|----------|----------|
| 忘记运行 `buf generate` | 中 | 高 | 无 | pre-commit hook + CI 同步检查 |
| 只提交部分生成文件 | 中 | 高 | 无 | pre-commit hook + CI 同步检查 |
| 破坏性变更未检测 | 低 | 高 | 无 | `buf breaking` 检查 |
| 生成代码冲突 | 中 | 中 | 人工解决 | 代码审查 + 小批量变更 |

### 10.3 核心建议

1. **封装生成流程**：使用 Makefile 或脚本将 `buf generate` 与后续检查封装为一个命令
2. **添加 pre-commit hook**：在提交前强制检查 proto 变更与生成代码的对应关系
3. **增强 CI 检查**（建议）：添加生成代码同步性检查，确保 proto 与生成代码永远一致
4. **代码审查清单**：在 PR 审查时检查 proto 相关变更的完整性
5. **破坏性变更管理**：使用 `reserved` 关键字和 `buf breaking` 检查

### 10.4 后端连接层架构修正

之前的描述有误，真实的后端连接层是**双层架构**：

| 组件 | 职责 | 接口来源 |
|------|------|----------|
| `APIV1Service` | 实现业务逻辑 | `v1pb.Unimplemented*Server`（来自 `*_grpc.pb.go`） |
| `ConnectServiceHandler` | 适配 Connect 协议 | `apiv1connect.*Handler`（来自 `*.connect.go`） |

关键模式：
- `ConnectServiceHandler` 嵌入 `APIV1Service`
- `ConnectServiceHandler` 的方法接收 `*connect.Request[T]`，返回 `*connect.Response[T]`
- 通过 `req.Msg` 提取内部类型，委托调用 `APIV1Service` 的方法
- 使用 `convertGRPCError` 统一转换错误类型
- 注册时使用 `apiv1connect.New*ServiceHandler`（注意包名是 `apiv1connect`，不是 `v1connect`）

这种设计确保了：
1. 业务逻辑只实现一次（在 `APIV1Service`）
2. 同时支持 gRPC-Gateway 和 Connect 两种协议
3. 所有接口和注册函数都由 `buf generate` 生成，与 proto 定义保持一致
