---
name: go-kratos
description: "Go-Kratos 微服务框架开发助手。触发词：kratos、protobuf API 定义、Go 微服务分层架构、HTTP/gRPC 服务配置、中间件开发、JWT/Casbin 认证、buf 生成工具、proto validate、WebSocket/文件上传。即使未明确提及 kratos，当涉及这些主题或项目包含 buf.yaml、internal/service、internal/biz 目录时也应触发。"
---

# Go-Kratos Skill

协助开发者使用 go-kratos 微服务框架进行开发。

## Quick Decision Tree

你正在做什么？

| 任务 | 参考 |
|------|------|
| 定义新 API / proto 文件 | [proto-api-design.md](references/proto-api-design.md) |
| 开发 Service/Biz/Data 层 | [architecture.md](references/architecture.md) |
| 配置项目 / 启动参数 | [configuration.md](references/configuration.md) |
| 定制 HTTP 响应 / WebSocket / 文件 | [http-customization.md](references/http-customization.md) |
| 添加认证 / JWT / Casbin | [security-auth.md](references/security-auth.md) |
| 编写中间件 / 日志处理 | [middleware-logging.md](references/middleware-logging.md) |
| 遇到问题 / 错误 | [troubleshooting.md](references/troubleshooting.md) |
| MCP / 高级扩展 | [advanced-features.md](references/advanced-features.md) |

---

## Development Workflow

### Phase 1: 确认项目类型

**输入**: 用户请求或项目目录
**输出**: 确认是否为 Kratos 项目

#### Step 1.1: 检查项目信号

检查以下信号判断是否为 Kratos 项目：

| 信号 | 检查方式 |
|------|---------|
| `buf.yaml` / `buf.gen.yaml` | 文件存在 |
| `internal/service/` | 目录存在 |
| `internal/biz/` | 目录存在 |
| `internal/data/` | 目录存在 |
| Go imports | 包含 `github.com/go-kratos/kratos/v2` |

**⚠️ 检查点**: 如果不满足 Kratos 项目特征，询问用户：
- 「当前项目似乎不是 Kratos 项目，是否继续按 Kratos 方式处理？」

### Phase 2: API/服务开发

**输入**: API 需求描述
**输出**: proto 定义 + service/biz/data 层代码

#### Step 2.1: 选择开发类型

| 开发类型 | 参考 |
|---------|------|
| 定义新 API / proto | [proto-api-design.md](references/proto-api-design.md) |
| Service/Biz/Data 层开发 | [architecture.md](references/architecture.md) |
| HTTP 响应定制 / WebSocket / 文件上传 | [http-customization.md](references/http-customization.md) |

#### Step 2.2: proto 定义（如需要）

**⚠️ 检查点**: 确认 proto 包名和 go_package 路径

```protobuf
syntax = "proto3";
package myservice.v1;  // 确认包名

option go_package = "github.com/myorg/myproject/api/myservice/v1;v1";  // 确认路径
```

#### Step 2.3: 分层代码生成

按 Kratos 分层架构生成代码：

```
proto → service 层（协议转换）
      → biz 层（业务逻辑）
      → data 层（数据访问）
```

**⚠️ 检查点**: 确认是否需要生成完整三层，或仅部分层

### Phase 3: 配置与集成

**输入**: 服务代码已完成
**输出**: 配置文件、中间件、模块注册

#### Step 3.1: 项目配置

参考 [configuration.md](references/configuration.md) 配置启动参数。

#### Step 3.2: 安全认证（如需要）

**⚠️ 检查点**: 如果服务需要认证，询问用户方案：
- 「是否需要 JWT 认证？」
- 「是否需要 Casbin 权限控制？」

参考 [security-auth.md](references/security-auth.md)。

#### Step 3.3: 中间件开发

参考 [middleware-logging.md](references/middleware-logging.md) 添加中间件。

#### Step 3.4: fx 模块注册

确保各层模块正确注册到 fx：

```go
// cmd/server/main.go
app := fx.New(
    server.Module,
    data.Module,
    biz.Module,
    service.Module,
    appModule,
)
```

### Phase 4: 问题排查

**输入**: 错误信息或问题描述
**输出**: 解决方案或调试步骤

参考 [troubleshooting.md](references/troubleshooting.md)。

---

## Quick Code Patterns

常用任务的代码片段——无需阅读完整参考文档即可使用。

### 1. Proto 定义模板（含 buf.validate）

```protobuf
syntax = "proto3";
package myservice.v1;

import "buf/validate/validate.proto";
import "google/api/annotations.proto";

option go_package = "github.com/myorg/myproject/api/myservice/v1;v1";

service MyService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {get: "/v1/users/{user_id}"};
  }
}

message GetUserRequest {
  string user_id = 1 [
    (buf.validate.field).string.min_len = 1,
    (buf.validate.field).string.max_len = 64
  ];
}

message GetUserResponse {
  string name = 1;
  string email = 2;
}
```

### 2. Service 层骨架

```go
package service

import (
    "buf.build/go/protovalidate"
    "context"
    "github.com/go-kratos/kratos/v2/log"
    v1 "github.com/myorg/myproject/api/myservice/v1"
    "github.com/myorg/myproject/internal/biz"
)

type MyService struct {
    v1.UnimplementedMyServiceServer
    log  *log.Helper
    uc   *biz.MyUseCase
    validator protovalidate.Validator
}

func NewMyService(uc *biz.MyUseCase, logger log.Logger, validator protovalidate.Validator) *MyService {
    return &MyService{
        uc: uc,
        validator: validator,
        log: log.NewHelper(log.With(logger, "module", "service/myService")),
    }
}

func (s *MyService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserResponse, error) {
    if err := s.validator.Validate(req); err != nil {
        return nil, err
    }
    // Call biz layer and convert result
    user, err := s.uc.GetUser(ctx, req.UserId)
    if err != nil {
        return nil, err
    }
    // Convert biz.User to proto response
    return &v1.GetUserResponse{
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### 3. fx 模块注册模式

```go
package service

import "go.uber.org/fx"

var Module = fx.Options(
    fx.Provide(NewMyService),
)
```

Similarly for `biz.Module`, `data.Module`, `server.Module`:
```go
// cmd/server/main.go
app := fx.New(
    server.Module,
    data.Module,
    biz.Module,
    service.Module,
    appModule,
)
```

同理适用于 `biz.Module`、`data.Module`、`server.Module`：
```go
// cmd/server/main.go
app := fx.New(
    server.Module,
    data.Module,
    biz.Module,
    service.Module,
    appModule,
)
```

### 4. ResponseEncoder 示例

```go
func CustomResponseEncoder() http.ServerOption {
    return http.ResponseEncoder(func(w http.ResponseWriter, r *http.Request, i interface{}) error {
        reply := &v1.BaseResponse{Code: 0}
        if m, ok := i.(proto.Message); ok {
            payload, err := anypb.New(m)
            if err != nil {
                return err
            }
            reply.Data = payload
        }
        codec := encoding.GetCodec("json")
        data, err := codec.Marshal(reply)
        if err != nil {
            return err
        }
        w.Header().Set("Content-Type", "application/json")
        if _, err := w.Write(data); err != nil {
            return err
        }
        return nil
    })
}
```

### 5. 从 Context 获取 JWT Payload

```go
func getPayloadFromCtx(ctx context.Context, partName string) (string, error) {
    if claims, ok := jwt.FromContext(ctx); ok {
        if m, ok := claims.(jwtV4.MapClaims); ok {
            if v, ok := m[partName].(string); ok {
                return v, nil
            }
        }
    }
    return "", errors.New("invalid Jwt")
}
```

### 6. 中间件骨架

```go
func MyMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (reply interface{}, err error) {
            // Pre-processing
            if tr, ok := transport.FromServerContext(ctx); ok {
                // Access headers, operation, etc.
                userAgent := tr.RequestHeader().Get("User-Agent")
            }
            
            // Call next handler
            reply, err = handler(ctx, req)
            
            // Post-processing
            return reply, err
        }
    }
}
```

---

## 这是 Kratos 项目吗？

检查以下信号：
- `buf.yaml` 或 `buf.gen.yaml` 存在
- `internal/service/` 目录存在
- `internal/biz/` 目录存在
- `internal/data/` 目录存在
- Go imports 包含 `github.com/go-kratos/kratos/v2`

---

## 相关 Skills

配合使用：
- `golang-patterns` — Go 惯用模式和模式
- `effective-go` — Go 最佳实践
- `go-best-practices` — 生产级 Go 模式

使用时机：获得 Kratos 指导后，应用 Go 最佳实践到实现中。

---

## 核心概念

### 分层架构

Kratos 遵循整洁架构：
- **Service 层**: 协议转换（HTTP/gRPC → 内部），简单校验
- **Biz 层**: 业务逻辑，领域模型，用例
- **Data 层**: 数据访问，仓储实现，外部服务

**为什么重要**: 每层职责清晰。协议变更不影响业务逻辑。业务逻辑不依赖存储细节。

### buf 管理 Proto

Kratos 推荐使用 buf 而非原生 protoc：
- 通过 `buf.gen.yaml` 集中管理插件
- 通过 `buf.yaml` 管理依赖
- 支持 lint 和破坏性变更检测

**为什么重要**: 团队间 proto 生成一致，依赖更新更简单。

### protovalidate 字段校验

使用 buf 的 protovalidate（不是 envoy 的 protoc-gen-validate）：
- CEL 表达式支持复杂规则
- 业务逻辑前进行运行时校验
- 基于 proto 的校验定义

**为什么重要**: 校验规则与数据定义在一起，无需单独校验代码。

---

## 错误处理与边界条件

### 常见错误场景

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| proto 校验失败 | 字段不符合 buf.validate 规则 | 返回详细校验错误，指明哪个字段 |
| biz 层返回错误 | 业务逻辑异常 | 使用 Kratos 错误码规范，封装 `v1.ErrorXxx` |
| data 层数据库错误 | 连接失败/查询异常 | 包装为 biz 错误，记录日志 |
| JWT 解析失败 | token 无效/过期 | 返回 401，引导用户重新登录 |
| fx 模块注册失败 | 依赖缺失 | 检查 Provider 顺序，确认所有依赖已注册 |

### Kratos 错误码规范

```go
// api/errors/errors.proto
enum ErrorReason {
  USER_NOT_FOUND = 0;
  INVALID_REQUEST = 1;
}

// 使用
import "google/rpc/error_details.proto"

// service 层返回
if user == nil {
    return nil, v1.ErrorUserNotFound("用户 %s 不存在", req.UserId)
}
```

### 错误包装与传递

```go
// biz 层
func (uc *MyUseCase) GetUser(ctx context.Context, id string) (*User, error) {
    user, err := uc.repo.FindByID(ctx, id)
    if err != nil {
        // 包装底层错误，添加上下文
        return nil, errors.Wrapf(err, "查找用户 %s 失败", id)
    }
    if user == nil {
        // 返回业务错误
        return nil, v1.ErrorUserNotFound("用户不存在")
    }
    return user, nil
}

// service 层
func (s *MyService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserResponse, error) {
    user, err := s.uc.GetUser(ctx, req.UserId)
    if err != nil {
        // biz 错误直接返回，Kratos 自动转换 HTTP 状态码
        return nil, err
    }
    return &v1.GetUserResponse{Name: user.Name}, nil
}
```

### 边界条件处理

| 边界情况 | 判断条件 | 处理方式 |
|---------|---------|---------|
| 空请求 | `req == nil` 或字段全空 | 返回 `INVALID_REQUEST` 错误 |
| ID 格式非法 | 非 UUID/数字格式 | proto 校验拦截，返回 400 |
| 分页越界 | `page > total_pages` | 返回空列表，不报错 |
| 权限不足 | `role < required` | 返回 403 Forbidden |
| 服务降级 | 外部服务不可用 | 返回 fallback 数据或缓存 |

---