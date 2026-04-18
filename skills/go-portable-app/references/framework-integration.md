# Go 框架集成指南

> Wire/fx + ent + Echo v5 集成，构建模块化的便携式应用

**依赖注入选择指南**：

| 特性 | Wire（Google） | fx（Uber） |
|------|---------------|------------|
| 类型 | 编译时（代码生成） | 运行时（反射） |
| 性能 | 更快（无反射开销） | 有反射开销 |
| 错误检测 | 编译时检测 | 运行时检测 |
| 适用场景 | 便携式应用、追求性能 | 大型应用、模块化需求 |
| 调试难度 | 较低（可读生成代码） | 较高（运行时行为） |

---

## Echo v5 核心概念

Echo 是高性能、极简的 Go Web 框架。

### 核心类型

```go
type Echo struct { ... }       // 主框架实例
type Group struct { ... }      // 路由分组
type Context struct { ... }    // 请求上下文
type MiddlewareFunc func(HandlerFunc) HandlerFunc  // 中间件
type HandlerFunc func(Context) error               // 处理器
```

### 创建实例

```go
import "github.com/labstack/echo/v5"

e := echo.New()                          // 默认配置
e := echo.NewWithConfig(echo.Config{...}) // 自定义配置
```

---

## Echo v5 路由 API

### 基础路由

```go
// HTTP 方法路由
func (e *Echo) GET(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) POST(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) PUT(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) DELETE(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) PATCH(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) HEAD(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) OPTIONS(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) CONNECT(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) TRACE(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo

// 匹配多个方法
func (e *Echo) Any(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (e *Echo) Match(methods []string, path string, h HandlerFunc, m ...MiddlewareFunc) Routes
```

### 路由分组（Group）

```go
// 创建分组
func (e *Echo) Group(prefix string, m ...MiddlewareFunc) (g *Group)

// 分组支持所有路由方法
func (g *Group) GET(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (g *Group) POST(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (g *Group) PUT(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo
func (g *Group) DELETE(path string, h HandlerFunc, m ...MiddlewareFunc) RouteInfo

// 嵌套分组
subGroup := group.Group("/sub", middleware...)

// 分组添加中间件
func (g *Group) Use(middleware ...MiddlewareFunc)
```

### 静态文件服务

```go
// 本地文件系统
func (e *Echo) Static(pathPrefix, fsRoot string, m ...MiddlewareFunc) RouteInfo
e.Static("/static", "public")

// 嵌入文件系统（embed.FS）
func (e *Echo) StaticFS(pathPrefix string, filesystem fs.FS, m ...MiddlewareFunc) RouteInfo
func (e *Echo) FileFS(path, file string, filesystem fs.FS, m ...MiddlewareFunc) RouteInfo

// 使用示例
//go:embed web/dist
var webFS embed.FS
distFS, _ := fs.Sub(webFS, "web/dist")
e.StaticFS("/", http.FS(distFS))
```

---

## Echo v5 中间件

### 添加中间件

```go
// 路由级中间件（在路由匹配后执行）
func (e *Echo) Use(middleware ...MiddlewareFunc)

// 前置中间件（在路由匹配前执行）
func (e *Echo) Pre(middleware ...MiddlewareFunc)

// 获取已注册中间件
func (e *Echo) Middlewares() []MiddlewareFunc
func (e *Echo) PreMiddlewares() []MiddlewareFunc
```

### 内置中间件

```go
import "github.com/labstack/echo/v5/middleware"

// 常用中间件
e.Use(middleware.Logger())        // 日志记录
e.Use(middleware.Recover())       // panic 恢复
e.Use(middleware.CORS())          // CORS 支持
e.Use(middleware.Gzip())          // gzip 压缩
e.Use(middleware.BasicAuth(...))  // 基础认证
e.Use(middleware.JWT(...))        // JWT 认证
e.Use(middleware.RateLimiter(...)) // 速率限制
e.Use(middleware.RequestID())     // 请求 ID
e.Use(middleware.Secure())        // 安全头

// CORS 配置示例
e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
    AllowOrigins: []string{"*"},
    AllowMethods: []string{echo.GET, echo.POST, echo.PUT, echo.DELETE},
    AllowHeaders: []string{"Content-Type", "Authorization"},
}))
```

### 自定义中间件

```go
func MyMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 前置处理
        c.Set("custom", "value")
        
        // 执行下一个处理器
        err := next(c)
        
        // 后置处理
        return err
    }
}

e.Use(MyMiddleware)
```

---

## Echo v5 Context API

### 获取数据

```go
func (c *Context) Param(name string) string      // 路径参数
func (c *Context) QueryParam(name string) string // 查询参数
func (c *Context) FormValue(name string) string  // 表单值
func (c *Context) Header(name string) string     // 请求头
func (c *Context) Cookie(name string) (*Cookie, error) // Cookie

func (c *Context) Bind(i interface{}) error     // 绑定请求体
func (c *Context) Get(name string) interface{}  // 获取上下文值
```

### 响应数据

```go
func (c *Context) JSON(code int, i interface{}) error      // JSON 响应
func (c *Context) JSONBlob(code int, b []byte) error       // JSON blob
func (c *Context) JSONP(code int, callback string, i interface{}) error // JSONP
func (c *Context) XML(code int, i interface{}) error       // XML 响应
func (c *Context) String(code int, s string) error         // 字符串响应
func (c *Context) HTML(code int, html string) error        // HTML 响应
func (c *Context) Blob(code int, contentType string, b []byte) error // 二进制响应
func (c *Context) Stream(code int, contentType string, r io.Reader) error // 流响应
func (c *Context) File(file string) error                   // 文件响应
func (c *Context) Attachment(file, name string) error       // 附件下载
func (c *Context) Inline(file, name string) error           // 内联显示
func (c *Context) Redirect(code int, url string) error      // 重定向
func (c *Context) NoContent(code int) error                 // 无内容响应
```

### 设置数据

```go
func (c *Context) Set(name string, val interface{})        // 设置上下文值
func (c *Context) SetCookie(cookie *Cookie)                // 设置 Cookie
func (c *Context) SetHeader(name, value string)            // 设置响应头
```

### HTTP 错误

```go
func (c *Context) Error(err error)                         // 发送错误
echo.NewHTTPError(code int, message ...interface{})        // 创建 HTTP 错误

// 使用示例
if err != nil {
    return echo.NewHTTPError(http.StatusNotFound, "user not found")
}
```

---

## 启动服务器

```go
func (e *Echo) Start(address string) error
func (e *Echo) StartTLS(address string, certFile, keyFile string) error
func (e *Echo) StartAutoTLS(address string) error  // 自动 TLS（Let's Encrypt）

// 示例
e.Start(":8080")
e.StartTLS(":443", "cert.pem", "key.pem")

// 自定义 HTTP Server
s := &http.Server{
    Addr:    ":8080",
    Handler: e,
}
s.ListenAndServe()
```

---

## Ent + Echo 集成

### 数据层初始化

```go
package data

import (
    "context"
    "your-project/ent"
    _ "github.com/lib-x/entsqlite"
)

type Data struct {
    Client *ent.Client
}

func NewData() (*Data, error) {
    client, err := ent.Open("sqlite3", 
        "file:./data.db?cache=shared&_pragma=foreign_keys(1)&_pragma=journal_mode(WAL)&_pragma=synchronous(NORMAL)&_pragma=busy_timeout(10000)")
    if err != nil {
        return nil, err
    }
    
    // 自动迁移
    if err := client.Schema.Create(context.Background()); err != nil {
        return nil, err
    }
    
    return &Data{Client: client}, nil
}

func (d *Data) Close() error {
    return d.Client.Close()
}
```

### Handler 示例

```go
package handler

import (
    "net/http"
    "github.com/labstack/echo/v5"
    "your-project/ent"
    "your-project/ent/user"
)

type UserHandler struct {
    Client *ent.Client
}

func (h *UserHandler) List(c echo.Context) error {
    users, err := h.Client.User.Query().All(c.Request().Context())
    if err != nil {
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusOK, users)
}

func (h *UserHandler) Get(c echo.Context) error {
    id := c.Param("id")
    u, err := h.Client.User.Query().
        Where(user.ID(id)).
        Only(c.Request().Context())
    if err != nil {
        if ent.IsNotFound(err) {
            return echo.NewHTTPError(http.StatusNotFound, "user not found")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusOK, u)
}

func (h *UserHandler) Create(c echo.Context) error {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    if err := c.Bind(&req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    
    u, err := h.Client.User.Create().
        SetName(req.Name).
        SetEmail(req.Email).
        Save(c.Request().Context())
    if err != nil {
        if ent.IsConstraintError(err) {
            return echo.NewHTTPError(http.StatusConflict, "email already exists")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusCreated, u)
}
```

---

## Uber fx 集成（可选）

fx 是 Uber 的依赖注入框架，适合构建模块化应用。

### fx 核心概念

```go
import "go.uber.org/fx"

// 应用生命周期
func main() {
    fx.New(
        fx.Provide(NewData, NewServer, NewHandler),
        fx.Invoke(StartServer),
        fx.Namer(fx.DefaultNamer),
    ).Run()
}
```

### fx + ent + echo 完整示例

```go
package main

import (
    "context"
    "net/http"
    "go.uber.org/fx"
    "github.com/labstack/echo/v5"
    "github.com/labstack/echo/v5/middleware"
    "your-project/ent"
    _ "github.com/lib-x/entsqlite"
)

// 数据模块
func NewEntClient(lc fx.Lifecycle) (*ent.Client, error) {
    client, err := ent.Open("sqlite3", 
        "file:./data.db?cache=shared&_pragma=foreign_keys(1)&_pragma=journal_mode(WAL)")
    if err != nil {
        return nil, err
    }
    
    // 自动迁移
    if err := client.Schema.Create(context.Background()); err != nil {
        return nil, err
    }
    
    lc.Append(fx.Hook{
        OnStop: func(ctx context.Context) error {
            return client.Close()
        },
    })
    
    return client, nil
}

// Echo 模块
func NewEchoServer(lc fx.Lifecycle) *echo.Echo {
    e := echo.New()
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())
    
    lc.Append(fx.Hook{
        OnStop: func(ctx context.Context) error {
            return e.Shutdown(ctx)
        },
    })
    
    return e
}

// Handler 模块
type HandlerParams struct {
    fx.In
    Echo   *echo.Echo
    Client *ent.Client
}

func RegisterHandlers(p HandlerParams) {
    // API 分组
    api := p.Echo.Group("/api/v1")
    
    // 用户路由
    userHandler := &UserHandler{Client: p.Client}
    api.GET("/users", userHandler.List)
    api.GET("/users/:id", userHandler.Get)
    api.POST("/users", userHandler.Create)
    
    // 前端兜底（embed）
    //go:embed web/dist
    var webFS embed.FS
    distFS, _ := fs.Sub(webFS, "web/dist")
    p.Echo.StaticFS("/", http.FS(distFS))
}

// 启动服务器
func StartServer(e *echo.Echo, lc fx.Lifecycle) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go e.Start(":8080")
            return nil
        },
    })
}

// 主函数
func main() {
    fx.New(
        fx.Provide(NewEntClient, NewEchoServer),
        fx.Invoke(RegisterHandlers, StartServer),
    ).Run()
}
```

### fx 模块化组织

```go
// data.Module.go
var DataModule = fx.Module("data",
    fx.Provide(NewEntClient),
)

// server.Module.go
var ServerModule = fx.Module("server",
    fx.Provide(NewEchoServer),
    fx.Invoke(RegisterHandlers, StartServer),
)

// main.go
func main() {
    fx.New(
        DataModule,
        ServerModule,
    ).Run()
}
```

---

## Google Wire 集成（推荐用于便携式应用）

Wire 是 Google 的编译时依赖注入框架，通过代码生成实现，性能更好，适合便携式应用。

### Wire 核心概念

```go
import "github.com/google/wire"

// ProviderSet：一组 Provider 的集合
var Set = wire.NewSet(NewA, NewB, NewC)

// wire.Build：在 Injector 函数中声明依赖图
func InitializeApp() *App {
    wire.Build(NewA, NewB, NewC, NewApp)
    return nil  // 实际返回值由 wire 生成
}

// go:generate 触发代码生成
//go:generate wire
```

### Wire 核心函数

| 函数 | 用途 |
|------|------|
| `wire.NewSet(...)` | 创建 ProviderSet，组织一组 Provider |
| `wire.Build(...)` | 在 Injector 中声明依赖图 |
| `wire.Bind(new(Interface), new(*Struct))` | 接口绑定 |
| `wire.Value(T)` | 直接提供值（无需 Provider） |
| `wire.Struct(new(T), "Field1", "Field2")` | 从结构体字段提取依赖 |
| `wire.FieldsOf(new(T), "Field")` | 从已有结构体字段注入 |
| `wire.InterfaceValue(new(Interface), value)` | 提供接口值 |

---

### Wire + ent + Echo 完整示例

#### 1. 定义 Provider

```go
// data/wire.go
package data

import (
    "context"
    "github.com/google/wire"
    "your-project/ent"
    _ "github.com/lib-x/entsqlite"
)

// ProviderSet：数据层依赖集合
var DataSet = wire.NewSet(NewEntClient, NewData)

// Provider：创建 ent client
func NewEntClient() (*ent.Client, error) {
    client, err := ent.Open("sqlite3", 
        "file:./data.db?cache=shared&_pragma=foreign_keys(1)&_pragma=journal_mode(WAL)&_pragma=synchronous(NORMAL)&_pragma=busy_timeout(10000)")
    if err != nil {
        return nil, err
    }
    
    // 自动迁移
    if err := client.Schema.Create(context.Background()); err != nil {
        return nil, err
    }
    
    return client, nil
}

// Provider：创建 Data 结构
func NewData(client *ent.Client) *Data {
    return &Data{Client: client}
}

type Data struct {
    Client *ent.Client
}

func (d *Data) Close() error {
    return d.Client.Close()
}
```

#### 2. 定义 Server Provider

```go
// server/wire.go
package server

import (
    "embed"
    "io/fs"
    "net/http"
    
    "github.com/google/wire"
    "github.com/labstack/echo/v5"
    "github.com/labstack/echo/v5/middleware"
    "your-project/data"
)

// ProviderSet：服务器依赖集合
var ServerSet = wire.NewSet(NewEcho, NewRouter, NewFrontendFS)

// Provider：创建 Echo 实例
func NewEcho() *echo.Echo {
    e := echo.New()
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())
    e.Use(middleware.CORS())
    return e
}

// Provider：创建路由器
func NewRouter(e *echo.Echo, d *data.Data) *Router {
    return &Router{
        Echo:  e,
        Data:  d,
    }
}

type Router struct {
    Echo *echo.Echo
    Data *data.Data
}

// Provider：创建前端文件系统
func NewFrontendFS() (http.FileSystem, error) {
    //go:embed web/dist
    var webFS embed.FS
    distFS, err := fs.Sub(webFS, "web/dist")
    if err != nil {
        return nil, err
    }
    return http.FS(distFS), nil
}
```

#### 3. 定义 Handler Provider

```go
// handler/wire.go
package handler

import (
    "github.com/google/wire"
    "github.com/labstack/echo/v5"
    "your-project/data"
)

// ProviderSet：Handler 依赖集合
var HandlerSet = wire.NewSet(
    NewUserHandler,
    NewAuthHandler,
)

func NewUserHandler(d *data.Data) *UserHandler {
    return &UserHandler{Data: d}
}

func NewAuthHandler(d *data.Data) *AuthHandler {
    return &AuthHandler{Data: d}
}

type UserHandler struct { Data *data.Data }
type AuthHandler struct { Data *data.Data }
```

#### 4. 定义 Injector

```go
// wire.go（主包）
package main

import (
    "github.com/google/wire"
    "your-project/data"
    "your-project/server"
    "your-project/handler"
)

// 超级集合：组合所有 ProviderSet
var SuperSet = wire.NewSet(
    data.DataSet,
    server.ServerSet,
    handler.HandlerSet,
    NewApp,
)

// Injector：Wire 生成初始化函数
func InitializeApp() (*App, error) {
    wire.Build(SuperSet)
    return nil, nil
}

//go:generate wire
```

#### 5. App 结构和启动

```go
// app.go
package main

import (
    "net/http"
    
    "github.com/labstack/echo/v5"
    "your-project/data"
    "your-project/handler"
    "your-project/server"
)

type App struct {
    Echo       *echo.Echo
    Router     *server.Router
    Data       *data.Data
    FrontendFS http.FileSystem
}

func NewApp(
    e *echo.Echo,
    r *server.Router,
    d *data.Data,
    fs http.FileSystem,
) (*App, error) {
    return &App{
        Echo:       e,
        Router:     r,
        Data:       d,
        FrontendFS: fs,
    }, nil
}

func (a *App) SetupRoutes() {
    api := a.Echo.Group("/api/v1")
    
    // 注册 Handler
    uh := &handler.UserHandler{Data: a.Data}
    api.GET("/users", uh.List)
    api.POST("/users", uh.Create)
    
    // 前端兜底
    a.Echo.StaticFS("/", a.FrontendFS)
}

func (a *App) Run() error {
    a.SetupRoutes()
    return a.Echo.Start(":8080")
}

func (a *App) Close() error {
    return a.Data.Close()
}

// main.go
package main

import "log"

func main() {
    app, err := InitializeApp()
    if err != nil {
        log.Fatalf("failed to initialize app: %v", err)
    }
    defer app.Close()
    
    if err := app.Run(); err != nil {
        log.Fatalf("failed to run app: %v", err)
    }
}
```

---

### Wire 高级用法

#### 接口绑定

```go
// 定义接口
type Repository interface {
    Find(id string) (*User, error)
    Save(user *User) error
}

// 实现接口
type EntRepository struct {
    Client *ent.Client
}

func NewEntRepository(client *ent.Client) *EntRepository {
    return &EntRepository{Client: client}
}

// Wire 绑定接口到实现
var RepoSet = wire.NewSet(
    NewEntRepository,
    wire.Bind(new(Repository), new(*EntRepository)),
)
```

#### 结构体字段注入

```go
// wire.Struct：自动从结构体提取字段作为依赖
var Set = wire.NewSet(
    wire.Struct(new(Config), "DB", "Port"),  // 只注入 DB 和 Port
    wire.Struct(new(Config), "*"),           // 注入所有字段
)

type Config struct {
    DB   *ent.Client
    Port int
    Host string  // 不注入（如果使用 wire.Struct(new(Config), "DB", "Port")）
}
```

#### 值绑定

```go
// wire.Value：直接提供值，无需 Provider
var ConfigSet = wire.NewSet(
    wire.Value(Config{Port: 8080}),
    wire.InterfaceValue(new(http.FileSystem), http.Dir("static")),
)

// wire.FieldsOf：从已有结构体提取字段
var Set = wire.NewSet(
    wire.FieldsOf(new(AppConfig), "DB"),  // 提取 AppConfig.DB 字段
)
```

---

### Wire 生成代码示例

运行 `wire` 后生成的代码：

```go
// wire_gen.go（自动生成）
// Code generated by wire. DO NOT EDIT.

package main

import (
    "your-project/data"
    "your-project/server"
    "your-project/handler"
)

func InitializeApp() (*App, error) {
    client, err := data.NewEntClient()
    if err != nil {
        return nil, err
    }
    d := data.NewData(client)
    e := server.NewEcho()
    r := server.NewRouter(e, d)
    fs, err := server.NewFrontendFS()
    if err != nil {
        return nil, err
    }
    app, err := NewApp(e, r, d, fs)
    if err != nil {
        return nil, err
    }
    return app, nil
}
```

---

### Wire vs fx 对比选择

| 场景 | 推荐 |
|------|------|
| 便携式应用、单文件部署 | **Wire**（编译时，无反射） |
| 追求最佳性能 | **Wire** |
| 需要编译时错误检测 | **Wire** |
| 大型应用、复杂模块化 | fx（运行时灵活） |
| 动态加载模块 | fx（Wire 不支持） |
| 团队已熟悉 Uber 技术栈 | fx |

---

### Wire 常见问题

#### 生成代码不更新

```bash
# 清理并重新生成
go clean
wire ./...
```

#### 循环依赖

```
wire: cycle detected: A -> B -> C -> A
```

**解决**：重构 Provider，打破循环依赖

#### 缺少 Provider

```
wire: no provider found for *ent.Client
```

**解决**：添加对应的 Provider 到 ProviderSet

---

### Wire 最佳实践

1. **每个包一个 ProviderSet**：按功能组织
2. **Injector 放在主包**：统一入口
3. **接口绑定显式声明**：避免隐式依赖
4. **生成代码检查**：查看 wire_gen.go 确认依赖图
5. **go:generate 注释**：方便 `go generate ./...` 触发

---

## WrapHandler 转换标准 HTTP Handler

```go
// 将标准 http.Handler 包装为 Echo Handler
func WrapHandler(h http.Handler) HandlerFunc

// 示例：包装文件服务器
fileServer := http.FileServer(http.FS(distFS))
e.GET("/*", echo.WrapHandler(http.StripPrefix("/static", fileServer)))
```

---

## 常见错误处理模式

```go
// Handler 中的错误处理
func (h *UserHandler) Update(c echo.Context) error {
    var req UpdateRequest
    if err := c.Bind(&req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, 
            "invalid request body")
    }
    
    user, err := h.Client.User.UpdateOneID(id).
        SetName(req.Name).
        Save(c.Request().Context())
    
    if err != nil {
        switch {
        case ent.IsNotFound(err):
            return echo.NewHTTPError(http.StatusNotFound, 
                "user not found")
        case ent.IsConstraintError(err):
            return echo.NewHTTPError(http.StatusConflict, 
                "constraint violation")
        default:
            return echo.NewHTTPError(http.StatusInternalServerError, 
                "internal error")
        }
    }
    
    return c.JSON(http.StatusOK, user)
}
```

---

## 参考

- [Echo v5 Documentation](https://pkg.go.dev/github.com/labstack/echo/v5)
- [Echo Middleware](https://echo.labstack.com/docs/category/middleware)
- [Ent Documentation](https://entgo.io/docs)
- [Uber fx](https://pkg.go.dev/go.uber.org/fx)
- [Google Wire](https://pkg.go.dev/github.com/google/wire)
- [Wire Tutorial](https://github.com/google/wire/blob/main/_tutorial/README.md)