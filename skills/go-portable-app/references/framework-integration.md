# Go 框架集成指南

> fx + ent + Echo v5 集成，构建模块化的便携式应用

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