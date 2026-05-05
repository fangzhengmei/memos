# Memos API Schema 与前端类型同步机制分析报告

## 1. 概述

Memos 项目采用 **Protocol Buffers (Protobuf)** 作为接口契约定义语言，通过 **Buf Build** 工具链实现从后端 API schema 到前端类型定义的全链路自动化同步。这种设计确保了前后端在 API 契约上的强一致性，避免了手动维护类型定义带来的错误和不一致。

## 2. 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              接口契约层 (Source of Truth)                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  proto/api/v1/*.proto                                                     │ │
│  │  - memo_service.proto    (备忘录服务定义)                                  │ │
│  │  - user_service.proto    (用户服务定义)                                    │ │
│  │  - auth_service.proto    (认证服务定义)                                    │ │
│  │  - attachment_service.proto (附件服务定义)                                │ │
│  │  - common.proto          (公共类型定义)                                    │ │
│  │  - ...                                                                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                        │
│                                        ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                         Buf 代码生成工具链                                 │ │
│  │  buf.yaml (构建配置)  +  buf.gen.yaml (生成配置)                          │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                        │
│                    ┌───────────────────┼───────────────────┐                  │
│                    ▼                   ▼                   ▼                  │
┌─────────────────────────────┐ ┌─────────────────────┐ ┌─────────────────────────┐
│       后端生成代码           │ │   OpenAPI 文档      │ │      前端生成代码         │
│  proto/gen/api/v1/          │ │ proto/gen/openapi   │ │ web/src/types/proto/    │
│  - *.pb.go        (Protobuf)│ │      .yaml          │ │  - *_pb.ts    (类型定义) │
│  - *_grpc.pb.go   (gRPC)    │ │                     │ │  - 服务描述符             │
│  - *.connect.go   (Connect) │ │                     │ │                         │
│  - *.pb.gw.go     (Gateway) │ │                     │ │                         │
└─────────────────────────────┘ └─────────────────────┘ └─────────────────────────┘
│                                        │                                        │
▼                                        ▼                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    运行时集成层                                         │
│  ┌─────────────────────────────┐              ┌─────────────────────────────┐      │
│  │      后端服务实现            │              │      前端客户端调用          │      │
│  │  server/router/api/v1/*.go  │              │  web/src/connect.ts         │      │
│  │  - 导入 v1pb 生成类型        │              │  - 创建 Connect RPC 客户端   │      │
│  │  - 实现服务接口              │              │  - 认证拦截器                │      │
│  │                             │              │                             │      │
│  │  类型: v1pb.CreateMemoRequest│              │  web/src/hooks/*.ts         │      │
│  │       v1pb.Memo             │              │  - useMemoQueries.ts         │      │
│  │                             │              │  - useUserQueries.ts         │      │
│  │                             │              │  - 使用生成的类型进行类型安全 │      │
│  └─────────────────────────────┘              └─────────────────────────────┘      │
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
    option (google.api.method_signature) = "memo";
  }
  
  // ListMemos 分页列出备忘录
  rpc ListMemos(ListMemosRequest) returns (ListMemosResponse) {
    option (google.api.http) = {get: "/api/v1/memos"};
  }
  
  // GetMemo 获取单个备忘录
  rpc GetMemo(GetMemoRequest) returns (Memo) {
    option (google.api.http) = {get: "/api/v1/{name=memos/*}"};
  }
  
  // UpdateMemo 更新备忘录
  rpc UpdateMemo(UpdateMemoRequest) returns (Memo) {
    option (google.api.http) = {
      patch: "/api/v1/{memo.name=memos/*}"
      body: "memo"
    };
  }
  
  // DeleteMemo 删除备忘录
  rpc DeleteMemo(DeleteMemoRequest) returns (google.protobuf.Empty) {
    option (google.api.http) = {delete: "/api/v1/{name=memos/*}"};
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
| **方法签名** | `google.api.method_signature` 定义客户端调用的规范签名 |

### 3.2 Buf 构建配置

#### 3.2.1 `proto/buf.yaml` - 项目配置

```yaml
version: v2
deps:
  - buf.build/googleapis/googleapis  # 依赖 Google API 公共定义
lint:
  use:
    - BASIC  # 使用基础 lint 规则集
  except:
    - ENUM_VALUE_PREFIX      # 排除枚举值前缀要求
    - FIELD_NOT_REQUIRED      # 排除字段必需性检查
    - PACKAGE_DIRECTORY_MATCH # 排除包与目录匹配要求
    - PACKAGE_NO_IMPORT_CYCLE # 排除包循环导入检查
    - PACKAGE_VERSION_SUFFIX  # 排除包版本后缀要求
  disallow_comment_ignores: true
breaking:
  use:
    - FILE  # 破坏性变更检查（文件级别）
  except:
    - EXTENSION_NO_DELETE
    - FIELD_SAME_DEFAULT
```

#### 3.2.2 `proto/buf.gen.yaml` - 代码生成配置（核心）

```yaml
version: v2
managed:
  enabled: true
  disable:
    - file_option: go_package
      module: buf.build/googleapis/googleapis
  override:
    - file_option: go_package_prefix
      value: github.com/usememos/memos/proto/gen

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
      - target=ts               # 生成 TypeScript 而非 JavaScript
    include_imports: true       # 包含所有依赖的导入
```

#### 3.2.3 插件输出对照表

| 插件 | 输出目录 | 生成内容 | 用途 |
|------|----------|----------|------|
| `protocolbuffers/go` | `proto/gen/api/v1/*.pb.go` | Protobuf 消息的 Go 类型定义 | 后端业务逻辑中的类型使用 |
| `grpc/go` | `proto/gen/api/v1/*_grpc.pb.go` | gRPC 服务端接口和客户端存根 | 纯 gRPC 通信（如内部服务） |
| `connectrpc/go` | `proto/gen/api/v1/apiv1connect/*.connect.go` | Connect RPC 服务实现和客户端 | 主要的 HTTP/gRPC 双协议服务 |
| `grpc-ecosystem/gateway` | `proto/gen/api/v1/*.pb.gw.go` | gRPC-Gateway 反向代理代码 | HTTP 到 gRPC 的转换（可选） |
| `google-gnostic-openapi` | `proto/gen/openapi.yaml` | OpenAPI 3.0 文档 | API 文档生成、第三方客户端 |
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
│       ├── attachment_service_pb.ts # Attachment 服务的类型和描述符
│       ├── common_pb.ts             # 公共类型（State, Order 等）
│       ├── instance_service_pb.ts   # Instance 服务
│       ├── shortcut_service_pb.ts   # Shortcut 服务
│       └── idp_service_pb.ts        # IDP 服务
└── google/
    └── api/
        ├── annotations_pb.ts
        ├── client_pb.ts
        ├── field_behavior_pb.ts
        ├── http_pb.ts
        ├── launch_stage_pb.ts
        └── resource_pb.ts
```

### 4.2 生成的 TypeScript 代码示例

**文件**: `web/src/types/proto/api/v1/memo_service_pb.ts`

```typescript
// @generated by protoc-gen-es v2.12.0 with parameter "target=ts"
// @generated from file api/v1/memo_service.proto (package memos.api.v1, syntax proto3)

import type { GenEnum, GenFile, GenMessage, GenService } from "@bufbuild/protobuf/codegenv2";
import { enumDesc, fileDesc, messageDesc, serviceDesc } from "@bufbuild/protobuf/codegenv2";

/**
 * @generated from message memos.api.v1.Memo
 */
export type Memo = Message<"memos.api.v1.Memo"> & {
  /**
   * The resource name of the memo.
   * Format: memos/{memo}, memo is the user defined id or uuid.
   *
   * @generated from field: string name = 1;
   */
  name: string;

  /**
   * The state of the memo.
   *
   * @generated from field: memos.api.v1.State state = 2;
   */
  state: State;

  /**
   * The name of the creator.
   * Format: users/{user}
   *
   * @generated from field: string creator = 3;
   */
  creator: string;

  /**
   * Required. The content of the memo in Markdown format.
   *
   * @generated from field: string content = 7;
   */
  content: string;

  /**
   * The visibility of the memo.
   *
   * @generated from field: memos.api.v1.Visibility visibility = 9;
   */
  visibility: Visibility;

  /**
   * Output only. The tags extracted from the content.
   *
   * @generated from field: repeated string tags = 10;
   */
  tags: string[];

  /**
   * Whether the memo is pinned.
   *
   * @generated from field: bool pinned = 11;
   */
  pinned: boolean;

  // ... 更多字段
};

/**
 * Describes the message memos.api.v1.Memo.
 * Use `create(MemoSchema)` to create a new message.
 */
export const MemoSchema: GenMessage<Memo> = /*@__PURE__*/
  messageDesc(file_api_v1_memo_service, 1);

/**
 * @generated from message memos.api.v1.CreateMemoRequest
 */
export type CreateMemoRequest = Message<"memos.api.v1.CreateMemoRequest"> & {
  /**
   * Required. The memo to create.
   *
   * @generated from field: memos.api.v1.Memo memo = 1;
   */
  memo: Memo;

  /**
   * Optional. The memo ID to use for this memo.
   *
   * @generated from field: string memo_id = 2;
   */
  memoId: string;
};

export const CreateMemoRequestSchema: GenMessage<CreateMemoRequest> = /*@__PURE__*/
  messageDesc(file_api_v1_memo_service, 2);

/**
 * @generated from service memos.api.v1.MemoService
 */
export const MemoService: GenService = {
  typeName: "memos.api.v1.MemoService",
  methods: {
    /**
     * CreateMemo creates a memo.
     */
    createMemo: {
      name: "CreateMemo",
      I: CreateMemoRequestSchema,
      O: MemoSchema,
      kind: MethodKind.Unary,
      idempotencyLevel: IdempotencyLevel.IDEMPOTENT_UNKNOWN,
    },
    /**
     * ListMemos lists memos with pagination and filter.
     */
    listMemos: {
      name: "ListMemos",
      I: ListMemosRequestSchema,
      O: ListMemosResponseSchema,
      kind: MethodKind.Unary,
      idempotencyLevel: IdempotencyLevel.NO_SIDE_EFFECTS,
    },
    // ... 更多方法
  },
};
```

### 4.3 前端 Connect RPC 客户端配置

**文件**: `web/src/connect.ts`

```typescript
import { Code, ConnectError, createClient, type Interceptor } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-web";

// 导入生成的服务描述符
import { MemoService } from "./types/proto/api/v1/memo_service_pb";
import { UserService } from "./types/proto/api/v1/user_service_pb";
import { AuthService } from "./types/proto/api/v1/auth_service_pb";
import { AttachmentService } from "./types/proto/api/v1/attachment_service_pb";
// ... 其他服务

// ============================================================================
// 认证拦截器（自动处理 Token 刷新）
// ============================================================================

const authInterceptor: Interceptor = (next) => async (req) => {
  const isRetryAttempt = req.header.get(RETRY_HEADER) === RETRY_HEADER_VALUE;
  const token = await getRequestToken();
  setAuthorizationHeader(req, token);

  try {
    return await next(req);
  } catch (error) {
    if (!shouldHandleUnauthenticatedRetry(error, isRetryAttempt)) {
      throw error;
    }

    try {
      const newToken = await refreshAndGetAccessToken();
      setAuthorizationHeader(req, newToken);
      req.header.set(RETRY_HEADER, RETRY_HEADER_VALUE);
      return await next(req);
    } catch (refreshError) {
      redirectOnAuthFailure();
      throw refreshError;
    }
  }
};

// ============================================================================
// Transport 配置
// ============================================================================

const transport = createConnectTransport({
  baseUrl: window.location.origin,
  useBinaryFormat: true,  // 使用 Protobuf 二进制格式（更高效）
  fetch: fetchWithCredentials,
  interceptors: [authInterceptor],  // 注册认证拦截器
});

// ============================================================================
// 创建服务客户端
// ============================================================================

// 核心服务客户端
export const instanceServiceClient = createClient(InstanceService, transport);
export const authServiceClient = createClient(AuthService, transport);
export const userServiceClient = createClient(UserService, transport);

// 内容服务客户端
export const memoServiceClient = createClient(MemoService, transport);
export const attachmentServiceClient = createClient(AttachmentService, transport);
export const aiServiceClient = createClient(AIService, transport);
export const shortcutServiceClient = createClient(ShortcutService, transport);

// 配置服务客户端
export const identityProviderServiceClient = createClient(IdentityProviderService, transport);
```

### 4.4 React Query Hooks 中的类型使用

**文件**: `web/src/hooks/useMemoQueries.ts`

```typescript
import { create } from "@bufbuild/protobuf";
import { FieldMaskSchema } from "@bufbuild/protobuf/wkt";
import type { InfiniteData } from "@tanstack/react-query";
import { useInfiniteQuery, useMutation, useQuery, useQueryClient } from "@tanstack/react-query";

// 导入生成的客户端
import { memoServiceClient } from "@/connect";

// 导入生成的类型定义（类型安全的核心）
import type { ListMemosRequest, ListMemosResponse, Memo } from "@/types/proto/api/v1/memo_service_pb";
import { ListMemoCommentsRequestSchema, ListMemosRequestSchema, MemoSchema } from "@/types/proto/api/v1/memo_service_pb";

// ============================================================================
// Query Keys 工厂（用于缓存管理）
// ============================================================================

export const memoKeys = {
  all: ["memos"] as const,
  lists: () => [...memoKeys.all, "list"] as const,
  list: (filters: Partial<ListMemosRequest>) => [...memoKeys.lists(), filters] as const,
  details: () => [...memoKeys.all, "detail"] as const,
  detail: (name: string) => [...memoKeys.details(), name] as const,
  comments: (name: string) => [...memoKeys.all, "comments", name] as const,
  linkMetadata: (url: string) => [...memoKeys.all, "linkMetadata", url] as const,
};

// ============================================================================
// Query Hooks
// ============================================================================

/**
 * 获取备忘录列表（类型安全）
 * @param request - 筛选条件，类型为 Partial<ListMemosRequest>
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
 * 无限滚动获取备忘录
 */
export function useInfiniteMemos(request: Partial<ListMemosRequest> = {}, options?: { enabled?: boolean }) {
  return useInfiniteQuery({
    queryKey: memoKeys.list(request),
    queryFn: async ({ pageParam }) => {
      const response = await memoServiceClient.listMemos(
        create(ListMemosRequestSchema, {
          ...request,
          pageToken: pageParam || "",
        } as Record<string, unknown>),
      );
      return response;
    },
    initialPageParam: "",
    getNextPageParam: (lastPage) => lastPage.nextPageToken || undefined,
    staleTime: 1000 * 60,
    gcTime: 1000 * 60 * 5,
    enabled: options?.enabled ?? true,
  });
}

/**
 * 获取单个备忘录
 */
export function useMemo(name: string, options?: { enabled?: boolean }) {
  return useQuery({
    queryKey: memoKeys.detail(name),
    queryFn: async () => {
      // 直接使用对象字面量，类型会被检查
      const memo = await memoServiceClient.getMemo({ name });
      return memo;
    },
    enabled: options?.enabled ?? true,
    staleTime: 1000 * 10,
  });
}

// ============================================================================
// Mutation Hooks
// ============================================================================

/**
 * 创建备忘录
 */
export function useCreateMemo() {
  const queryClient = useQueryClient();

  return useMutation({
    // 参数类型严格限定为 Memo
    mutationFn: async (memoToCreate: Memo) => {
      const memo = await memoServiceClient.createMemo({ memo: memoToCreate });
      return memo;
    },
    onSuccess: (newMemo) => {
      // 类型推断：newMemo 自动为 Memo 类型
      queryClient.invalidateQueries({ queryKey: memoKeys.lists() });
      queryClient.setQueryData(memoKeys.detail(newMemo.name), newMemo);
    },
  });
}

/**
 * 更新备忘录（使用 FieldMask 进行部分更新）
 */
export function useUpdateMemo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ update, updateMask }: { update: Partial<Memo>; updateMask: string[] }) => {
      const memo = await memoServiceClient.updateMemo({
        // 创建类型安全的 Memo 对象
        memo: create(MemoSchema, update as Record<string, unknown>),
        // 创建 FieldMask
        updateMask: create(FieldMaskSchema, { paths: updateMask }),
      });
      return memo;
    },
    // ... optimistic update 逻辑
  });
}
```

## 5. 后端服务实现

### 5.1 后端类型导入

**文件**: `server/router/api/v1/memo_service.go`

```go
package v1

import (
	"context"
	"fmt"
	"time"

	"github.com/pkg/errors"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/emptypb"

	// 导入生成的 API 类型（v1 版本）
	v1pb "github.com/usememos/memos/proto/gen/api/v1"
	// 导入生成的 Store 类型
	storepb "github.com/usememos/memos/proto/gen/store"
	"github.com/usememos/memos/store"
)

// APIV1Service 实现了 v1pb.MemoService 等接口
func (s *APIV1Service) CreateMemo(ctx context.Context, request *v1pb.CreateMemoRequest) (*v1pb.Memo, error) {
	user, err := s.fetchCurrentUser(ctx)
	if err != nil {
		return nil, status.Errorf(codes.Internal, "failed to get user")
	}
	if user == nil {
		return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
	}

	// 使用生成的类型
	memoUID, err := ValidateAndGenerateUID(request.MemoId)
	if err != nil {
		return nil, err
	}

	// 转换 API 类型到 Store 类型
	create := &store.Memo{
		UID:        memoUID,
		CreatorID:  user.ID,
		Content:    request.Memo.Content,
		Visibility: convertVisibilityToStore(request.Memo.Visibility),
	}

	// 设置自定义时间戳
	if request.Memo.CreateTime != nil && request.Memo.CreateTime.IsValid() {
		createdTs := request.Memo.CreateTime.AsTime().Unix()
		create.CreatedTs = createdTs
	}

	// ... 业务逻辑

	memo, err := s.Store.CreateMemo(ctx, create)
	if err != nil {
		return nil, err
	}

	// 转换 Store 类型回 API 类型
	return convertMemoFromStore(memo), nil
}
```

### 5.2 Connect 服务注册

**文件**: `server/router/api/v1/connect_services.go`

```go
package v1

import (
	"net/http"

	"connectrpc.com/connect"
)

// RegisterConnectServices 注册所有 Connect RPC 服务
func (s *APIV1Service) RegisterConnectServices(mux *http.ServeMux, compression bool) {
	opts := []connect.HandlerOption{
		connect.WithInterceptors(s.connectInterceptors...),
	}
	if compression {
		opts = append(opts, connect.WithAcceptCompression("gzip", connect.NewGzipCompressor()))
	}

	// 使用生成的 Connect 代码注册服务
	path, handler := v1connect.NewMemoServiceHandler(s, opts...)
	mux.Handle(path, handler)

	path, handler = v1connect.NewUserServiceHandler(s, opts...)
	mux.Handle(path, handler)

	path, handler = v1connect.NewAuthServiceHandler(s, opts...)
	mux.Handle(path, handler)

	path, handler = v1connect.NewAttachmentServiceHandler(s, opts...)
	mux.Handle(path, handler)

	// ... 其他服务
}
```

## 6. CI/CD 保障机制

### 6.1 Proto Linter 工作流

**文件**: `.github/workflows/proto-linter.yml`

```yaml
name: Proto Linter

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths:
      - "proto/**"  # 仅当 proto 文件变更时触发

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint Protos
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Setup buf
        uses: bufbuild/buf-setup-action@v1
        with:
          github_token: ${{ github.token }}

      # 1. 运行 buf lint 检查
      - name: Run buf lint
        uses: bufbuild/buf-lint-action@v1
        with:
          input: proto

      # 2. 检查格式是否正确
      - name: Check buf format
        run: |
          if [[ $(buf format -d) ]]; then
            echo "❌ Proto files are not formatted. Run 'buf format -w' to fix."
            exit 1
          fi
```

### 6.2 构建流程中的类型同步

在 `release.yml` 和 `build-canary-image.yml` 中，虽然没有显式运行 `buf generate`，但：

1. **生成的代码已提交到仓库**：`proto/gen/` 和 `web/src/types/proto/` 目录下的文件都在 Git 版本控制中
2. **开发者本地生成**：开发者在修改 proto 文件后需要手动运行 `buf generate`
3. **CI 验证**：通过 `proto-linter.yml` 确保 proto 文件的规范性

## 7. 同步机制流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        开发者工作流（类型同步触发点）                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  1. 修改 Protobuf Schema                                                       │
│     位置: proto/api/v1/*.proto                                                 │
│     操作: 添加/修改服务、消息、字段                                              │
│     示例: 在 memo_service.proto 中添加新的 RPC 方法                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  2. 本地代码生成                                                                 │
│     命令: cd proto && buf generate                                             │
│     触发: 开发者手动执行                                                        │
│                                                                                 │
│     生成输出:                                                                    │
│     ├── Go 代码    → proto/gen/api/v1/                                        │
│     ├── TS 代码    → web/src/types/proto/api/v1/                              │
│     └── OpenAPI    → proto/gen/openapi.yaml                                   │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  3. 类型检查（开发时）                                                          │
│                                                                                 │
│     前端:                                                                        │
│     ├── IDE 类型提示: 使用生成的 TypeScript 类型                                │
│     ├── 编译检查: pnpm lint (tsc --noEmit)                                    │
│     └── 运行时: Connect 客户端自动序列化/反序列化                              │
│                                                                                 │
│     后端:                                                                        │
│     ├── IDE 类型提示: 使用生成的 Go 类型                                        │
│     ├── 编译检查: go build / golangci-lint                                     │
│     └── 运行时: Protobuf 编解码                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  4. 提交代码                                                                    │
│     包含:                                                                        │
│     ├── 修改的 .proto 文件                                                      │
│     ├── 重新生成的 Go 代码 (proto/gen/)                                        │
│     └── 重新生成的 TS 代码 (web/src/types/proto/)                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  5. PR/Merge CI 检查                                                           │
│                                                                                 │
│     proto-linter.yml:                                                           │
│     ├── buf lint: 检查 proto 文件语法和风格                                    │
│     └── buf format: 检查 proto 文件格式                                        │
│                                                                                 │
│     frontend-tests.yml:                                                         │
│     └── pnpm lint: TypeScript 类型检查 + Biome lint                           │
│                                                                                 │
│     backend-tests.yml:                                                          │
│     ├── go build: Go 编译检查                                                   │
│     └── golangci-lint: Go lint 检查                                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  6. 运行时一致性保证                                                            │
│                                                                                 │
│     Connect 协议:                                                               │
│     ├── 基于 HTTP/2 或 HTTP/1.1                                                │
│     ├── 支持 Protobuf 二进制格式                                                │
│     ├── 服务端和客户端使用相同的 Schema 进行编解码                             │
│     └── 字段编号匹配，而非字段名                                                │
│                                                                                 │
│     向后兼容:                                                                    │
│     ├── Protobuf 字段编号不变                                                   │
│     ├── 新增字段不影响旧客户端                                                  │
│     └── 删除字段需保留编号（使用 reserved）                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 8. 关键技术点分析

### 8.1 为什么选择 Connect RPC 而非纯 gRPC？

| 特性 | Connect RPC | gRPC |
|------|-------------|------|
| **浏览器支持** | ✅ 原生支持（通过 connect-web） | ❌ 需要 gRPC-Web 代理 |
| **HTTP 版本** | HTTP/1.1 + HTTP/2 | 仅 HTTP/2 |
| **序列化格式** | Protobuf 二进制 + JSON | 仅 Protobuf 二进制 |
| **调试友好** | ✅ 可使用 curl、Postman 直接调用 | ❌ 需要特殊工具 |
| **代码生成** | 统一的 buf 工具链 | 需要 protoc + 多个插件 |

### 8.2 类型安全的多层次保障

```
┌─────────────────────────────────────────────────────────────────┐
│                      类型安全层级                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第 1 层: Protobuf Schema (Source of Truth)                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  proto/api/v1/*.proto                                    │  │
│  │  - 强类型定义 (string, int32, message, enum)            │  │
│  │  - 字段编号确保兼容性                                     │  │
│  │  - google.api.field_behavior 标记必填/可选               │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  第 2 层: 代码生成 (Buf Generate)                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  自动生成:                                                │  │
│  │  - Go: *.pb.go (类型 + 序列化)                           │  │
│  │  - TS: *_pb.ts (类型 + 服务描述符)                       │  │
│  │  确保: 前后端类型完全一致                                 │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  第 3 层: 编译时检查                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  前端: TypeScript 编译器 (tsc --noEmit)                 │  │
│  │  后端: Go 编译器 (go build)                              │  │
│  │  确保: 调用参数、返回值类型匹配                           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  第 4 层: 运行时编解码                                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Protobuf 二进制序列化:                                   │  │
│  │  - 基于字段编号，而非字段名                               │  │
│  │  - 未知字段自动忽略（向后兼容）                           │  │
│  │  - 类型不匹配时抛出明确错误                               │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 字段行为标记的作用

在 Protobuf 中使用 `google.api.field_behavior` 注解：

```protobuf
import "google/api/field_behavior.proto";

message Memo {
  // IDENTIFIER: 资源标识符，通常只读
  string name = 1 [
    (google.api.field_behavior) = IDENTIFIER
  ];

  // REQUIRED: 必需字段
  State state = 2 [
    (google.api.field_behavior) = REQUIRED
  ];

  // OUTPUT_ONLY: 仅输出字段，客户端不应设置
  string creator = 3 [
    (google.api.field_behavior) = OUTPUT_ONLY
  ];

  // OPTIONAL: 可选字段
  string content = 7 [
    (google.api.field_behavior) = OPTIONAL
  ];
}
```

这些标记在生成的代码中会被保留，用于：
1. **文档生成**：OpenAPI 文档中显示字段要求
2. **服务端验证**：可在拦截器中自动验证必填字段
3. **客户端提示**：IDE 可以显示字段的使用说明

## 9. 实际操作指南

### 9.1 添加新 API 的完整流程

```bash
# 1. 修改 Protobuf Schema
cd proto
# 编辑 api/v1/memo_service.proto，添加新方法或消息

# 2. 运行代码生成
buf generate

# 3. 检查生成的文件
# - Go: proto/gen/api/v1/memo_service.pb.go
# - TS: web/src/types/proto/api/v1/memo_service_pb.ts

# 4. 实现后端服务
# 编辑 server/router/api/v1/memo_service.go
# 实现新的 RPC 方法

# 5. 添加前端调用
# 编辑 web/src/hooks/useMemoQueries.ts
# 使用生成的客户端和类型

# 6. 运行类型检查
cd ../web
pnpm lint  # TypeScript 检查

# 7. 提交所有变更
git add proto/api/v1/*.proto
git add proto/gen/
git add web/src/types/proto/
git add server/router/api/v1/*.go
git add web/src/hooks/*.ts
```

### 9.2 常用 Buf 命令

```bash
cd proto

# 代码生成（最常用）
buf generate

# 检查 lint 问题
buf lint

# 格式化 proto 文件
buf format -w

# 检查破坏性变更（相对于 main 分支）
buf breaking --against '.git#branch=main'

# 构建（不生成代码，仅检查）
buf build
```

## 10. 优势与局限性

### 10.1 优势

| 优势 | 说明 |
|------|------|
| **单一事实来源** | Protobuf Schema 是唯一的 API 契约定义 |
| **类型安全** | 编译时检查，避免运行时类型错误 |
| **自动同步** | 一次定义，多端生成，无需手动维护 |
| **向后兼容** | Protobuf 字段编号机制确保版本兼容性 |
| **高效序列化** | Protobuf 二进制格式比 JSON 更小、更快 |
| **多协议支持** | Connect RPC 同时支持 HTTP/1.1 和 HTTP/2 |
| **IDE 友好** | 生成的代码提供完整的类型提示和自动补全 |

### 10.2 局限性与注意事项

| 局限性 | 说明 |
|--------|------|
| **学习曲线** | 需要了解 Protobuf 语法和 Buf 工具链 |
| **生成代码体积** | 生成的 TypeScript 代码可能增加包体积 |
| **需要手动触发** | 开发者需要记得运行 `buf generate` |
| **版本控制** | 生成的代码需要提交到仓库，可能产生冲突 |
| **调试复杂度** | 二进制格式不如 JSON 直观易读 |

### 10.3 最佳实践建议

1. **始终先修改 Schema**：任何 API 变更都应该从修改 `.proto` 文件开始
2. **及时生成代码**：修改 Schema 后立即运行 `buf generate`
3. **提交生成的代码**：将 `proto/gen/` 和 `web/src/types/proto/` 纳入版本控制
4. **使用字段编号**：永远不要修改已存在字段的编号
5. **使用 reserved**：删除字段时使用 `reserved` 标记其编号和名称
6. **CI 验证**：确保 PR 检查包含 `buf lint` 和类型检查

## 11. 总结

Memos 项目采用了一套完整且现代化的 API 契约管理方案：

1. **契约优先**：以 Protobuf Schema 作为单一事实来源
2. **代码生成**：通过 Buf Build 工具链自动生成多语言代码
3. **类型安全**：从定义到运行时的多层类型保障
4. **CI 保障**：自动化的 lint 和格式检查

这套机制确保了：
- ✅ 后端 API 定义与实现的一致性
- ✅ 前端类型定义与后端 Schema 的一致性
- ✅ 前后端通信的数据结构一致性
- ✅ API 版本迭代的向后兼容性

这种设计是现代全栈应用的最佳实践之一，特别适合需要前后端紧密协作、API 频繁迭代的项目。
