# 1. 简介

官网：https://gin-gonic.com/

github仓库地址：https://github.com/gin-gonic/gin

文档地址：https://pkg.go.dev/github.com/gin-gonic/gin

Gin 是一个 Go 语言编写的的 HTTP Web 框架，它包含的特性如下：

* 快速：基于 Radix 树的路由，小内存占用。没有反射。可预测的 API 性能；
* 支持中间件：传入的 HTTP 请求可以由一系列中间件和最终操作来处理。 例如：Logger，Authorization，GZIP，最终操作 DB；
* Crash 处理：Gin 可以 catch 一个发生在 HTTP 请求中的 panic 并 recover 它。这样，你的服务器将始终可用。例如，你可以向 Sentry 报告这个 panic；
* JSON 验证：Gin 可以解析并验证请求的 JSON，例如检查所需值的存在；
* 路由组：更好地组织路由。是否需要授权，不同的 API 版本…… 此外，这些组可以无限制地嵌套而不会降低性能；
* 错误管理：Gin 提供了一种方便的方法来收集 HTTP 请求期间发生的所有错误。最终，中间件可以将它们写入日志文件，数据库并通过网络发送；
* 内置渲染：Gin 为 JSON，XML 和 HTML 渲染提供了易于使用的 API；

# 2. 使用

## 2.1 示例程序

要使用 Gin，需要要求 Go 1.13 及以上版本。

下载安装 Gin：

```bash
go get -u github.com/gin-gonic/gin
```

在代码中使用包：

```go
import "github.com/gin-gonic/gin"
```

示例代码：

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()                    // 初始化对象
	r.GET("/ping", func(c *gin.Context) { // 注册一个 GET 方法接口
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // 监听并在 0.0.0.0:8080 上启动服务
}
```

执行命令启动程序：

```bash
go run main.go
```

更多官方示例可以参考：https://gin-gonic.com/zh-cn/docs/examples/

## 2.2 程序启动

直接使用 http.ListenAndServe 启动程序。

```go
func main() {
	router := gin.Default()
	http.ListenAndServe(":8080", router)
}
```

也可以自定义 HTTP 配置后启动程序。

```go
func main() {
	router := gin.Default()

	s := &http.Server{
		Addr:           ":8080",
		Handler:        router,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}
	s.ListenAndServe()
}
```

## 2.3 优雅重启或停止

启动程序前，监听特定的信号，然后在收到信号后退出程序，并在退出程序前，保留一定时间完成现有的请求。

不带有 context 的通知，Go 1.8 及更新版本支持。

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}

	// 创建一个协程启动服务
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("listen: %s\n", err)
		}
	}()

	// 监听中断信号
	quit := make(chan os.Signal)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("Shutdown Server ...")

	// 完成服务器停止前的必要工作

	// 设置 5 秒的超时时间
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	// 使用 http.Server 内置的 Shutdown() 方法优雅地关机
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server Shutdown:", err)
	}
	log.Println("Server exiting")
}
```

带有 context 的通知，Go 1.16 及更新版本支持。

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	// 创建 context 来监听信号
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(10 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}

	// 创建一个协程启动服务
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()

	// 监听中断信号
	<-ctx.Done()

	// 完成服务器停止前的必要工作

	// 设置 5 秒的超时时间
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	// 使用 http.Server 内置的 Shutdown() 方法优雅地关机
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server forced to shutdown: ", err)
	}

	log.Println("Server exiting")
}
```

## 2.4 记录日志

Gin 支持指定日志文件路径，然后就可以将输出日志记录到文件里去，也可以配置同时输出到控制台。

```go
func main() {
	// 禁用控制台颜色，将日志写入文件时不需要控制台颜色。
	gin.DisableConsoleColor()

	// 记录到文件。
	f, _ := os.Create("gin.log")
	gin.DefaultWriter = io.MultiWriter(f)

	// 如果需要同时将日志写入文件和控制台，请使用以下代码。
	// gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

	router := gin.Default()
	router.GET("/ping", func(c *gin.Context) {
		c.String(200, "pong")
	})

	router.Run(":8080")
}
```

默认的路由日志格式如下：

```
[GIN-debug] POST   /foo                      --> main.main.func1 (3 handlers)
[GIN-debug] GET    /bar                      --> main.main.func2 (3 handlers)
[GIN-debug] GET    /status                   --> main.main.func3 (3 handlers)
```

以下可以自定义路由日志格式。

```go
	gin.DebugPrintRouteFunc = func(httpMethod, absolutePath, handlerName string, nuHandlers int) {
		log.Printf("endpoint %v %v %v %v\n", httpMethod, absolutePath, handlerName, nuHandlers)
	}
```

## 2.5 将请求体绑定到结构体

要将请求体绑定到结构体中，使用模型绑定。 Gin目前支持JSON、XML、YAML和标准表单值的绑定（foo=bar＆boo=baz）。Gin使用 go-playground/validator/v10 进行验证。

使用 Bind 方法时，Gin 会尝试根据 Content-Type 推断如何绑定，结构体成员变量的标签指定了从哪个请求体参数中绑定。

你也可以指定必须绑定的字段， 如果一个字段的 tag 加上了 `binding:"required"`，但绑定时是空值，Gin 会报错。

```go
type Login struct {
	User     string `form:"user" json:"user" xml:"user"  binding:"required"`
	Password string `form:"password" json:"password" xml:"password" binding:"required"`
}
```

Gin 提供了两类绑定方法：

* Must bind：包含方法 Bind, BindJSON, BindXML, BindQuery, BindYAML，如果发生绑定错误，则请求终止。
* Shoud bind：包含方法 ShouldBind, ShouldBindJSON, ShouldBindXML, ShouldBindQuery, ShouldBindYAML，如果发生绑定错误，Gin 会返回错误并由开发者处理错误和请求。

通过调用 c.ShouldBind 方法将请求体 c.Request.Body 绑定到结构体，但是只能调用一次。

```go
type formA struct {
	Foo string `json:"foo" xml:"foo" binding:"required"`
}

type formB struct {
	Bar string `json:"bar" xml:"bar" binding:"required"`
}

func SomeHandler(c *gin.Context) {
	objA := formA{}
	objB := formB{}
	if errA := c.ShouldBind(&objA); errA == nil { // c.ShouldBind 使用了 c.Request.Body，不可重用。
		c.String(http.StatusOK, `the body should be formA`)
	} else if errB := c.ShouldBind(&objB); errB == nil { // 因为现在 c.Request.Body 是 EOF，所以这里会报错。
		c.String(http.StatusOK, `the body should be formB`)
	} else {
	}
}
```

要想多次绑定，可以使用 c.ShouldBindBodyWith。

```go
func SomeHandler(c *gin.Context) {
	objA := formA{}
	objB := formB{}
	if errA := c.ShouldBindBodyWith(&objA, binding.JSON); errA == nil { // 读取 c.Request.Body 并将结果存入上下文。
		c.String(http.StatusOK, `the body should be formA`)
	} else if errB := c.ShouldBindBodyWith(&objB, binding.JSON); errB == nil { // 这时, 复用存储在上下文中的 body。
		c.String(http.StatusOK, `the body should be formB JSON`)
	} else if errB2 := c.ShouldBindBodyWith(&objB, binding.XML); errB2 == nil { // 可以接受其他格式
		c.String(http.StatusOK, `the body should be formB XML`)
	} else {
	}
}
```

## 2.6 中间件

需要修改初始化 Engine 对象的方法 Default 改为 New。

```go
// Default 使用 Logger 和 Recovery 中间件
r := gin.Default()

// 替换为 New
r := gin.New()
```

中间件是在绑定的接口执行前后执行的代码，可以设置全局中间件，可以为某一个路由添加中间件，也可以为路由组设置中间件。

```go
func main() {
	// 新建一个没有任何默认中间件的路由
	r := gin.New()

	// 全局中间件
	// Logger 中间件将日志写入 gin.DefaultWriter，默认是 os.Stdout
	r.Use(gin.Logger())
	// Recovery 中间件会 recover 任何 panic。如果有 panic 的话，会写入 500。
	r.Use(gin.Recovery())

	// 你可以为每个路由添加任意数量的中间件。
	r.GET("/benchmark", benchmarkHandler(), middleware1, middleware2)

	// 路由组中间件
	authorized := r.Group("/", middleware3)
	authorized.Use(middleware4)
	{
		authorized.POST("/login", middleware5, loginEndpoint)
		authorized.POST("/submit", submitEndpoint)
		authorized.POST("/read", readEndpoint)

		// 嵌套路由组
		testing := authorized.Group("testing")
		testing.Use(middleware6)
		testing.GET("/analytics", analyticsEndpoint)
	}

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

自定义中间件并使用：

```go
func Logger() gin.HandlerFunc {
	return func(c *gin.Context) {
		t := time.Now()
		c.Set("example", "12345")

		c.Next() // 该调用前面的代码在请求之前执行，后面的代码在请求之后执行

		latency := time.Since(t)
		log.Print(latency)
		status := c.Writer.Status()
		log.Println(status)
	}
}

func main() {
	r := gin.New()
	r.Use(Logger()) // 使用自定义中间件

	r.GET("/test", func(c *gin.Context) {
		example := c.MustGet("example").(string)
		log.Println(example)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

## 2.7 路由匹配

通过 Engine 的各个方法，创建不同 HTTP 方法的路由。其中第一个参数指定了请求路由，如下面 GET 请求路由将匹配 /someGet。

```go
	router := gin.Default()
	router.GET("/someGet", getting)
	router.POST("/somePost", posting)
	router.PUT("/somePut", putting)
	router.DELETE("/someDelete", deleting)
	router.PATCH("/somePatch", patching)
	router.HEAD("/someHead", head)
	router.OPTIONS("/someOptions", options)
```

路由匹配还可以不固定，而是灵活地匹配实际的名称，并在代码中获取路径名称作为变量。

```go
func main() {
	router := gin.Default()

	// 此 handler 将匹配 /user/john 但不会匹配 /user/ 或者 /user
	router.GET("/user/:name", func(c *gin.Context) {
		name := c.Param("name")
		c.String(http.StatusOK, "Hello %s", name)
	})

	// 此 handler 将匹配 /user/john/ 和 /user/john/send
	// 如果没有其他路由匹配 /user/john，它将重定向到 /user/john/
	router.GET("/user/:name/*action", func(c *gin.Context) {
		name := c.Param("name")
		action := c.Param("action")
		message := name + " is " + action
		c.String(http.StatusOK, message)
	})

	router.Run(":8080")
}
```

通过路由组，可以将同样前缀的路由定义在一起，使路由设置更加简洁明了。

```go
func main() {
	router := gin.Default()

	// 简单的路由组: v1
	v1 := router.Group("/v1")
	{
		v1.POST("/login", loginEndpoint)
		v1.POST("/submit", submitEndpoint)
		v1.POST("/read", readEndpoint)
	}

	// 简单的路由组: v2
	v2 := router.Group("/v2")
	{
		v2.POST("/login", loginEndpoint)
		v2.POST("/submit", submitEndpoint)
		v2.POST("/read", readEndpoint)
	}

	router.Run(":8080")
}
```

## 2.8 重定向

外部重定向：

```go
r.GET("/test", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")
})
```

内部进行 HTTP 重定向：

```go
r.POST("/test", func(c *gin.Context) {
	c.Redirect(http.StatusFound, "/foo")
})
```

路由重定向：

```go
r.GET("/test", func(c *gin.Context) {
	c.Request.URL.Path = "/test2"
	r.HandleContext(c)
})
r.GET("/test2", func(c *gin.Context) {
	c.JSON(200, gin.H{"hello": "world"})
})
```

## 2.9 静态文件服务

对于路由指向特定目录或文件。

```go
func main() {
	router := gin.Default()
	router.Static("/assets", "./assets")
	router.StaticFS("/more_static", http.Dir("my_file_system"))
	router.StaticFile("/favicon.ico", "./resources/favicon.ico")

	// 监听并在 0.0.0.0:8080 上启动服务
	router.Run(":8080")
}
```

# 3. 开发文档

## 3.1 常量

常见的 Content-Type 和 MIME 类型。

```go
const (
	MIMEJSON              = binding.MIMEJSON
	MIMEHTML              = binding.MIMEHTML
	MIMEXML               = binding.MIMEXML
	MIMEXML2              = binding.MIMEXML2
	MIMEPlain             = binding.MIMEPlain
	MIMEPOSTForm          = binding.MIMEPOSTForm
	MIMEMultipartPOSTForm = binding.MIMEMultipartPOSTForm
	MIMEYAML              = binding.MIMEYAML
	MIMETOML              = binding.MIMETOML
)
```

模式。

```go
const (
	// DebugMode indicates gin mode is debug.
	DebugMode = "debug"
	// ReleaseMode indicates gin mode is release.
	ReleaseMode = "release"
	// TestMode indicates gin mode is test.
	TestMode = "test"
)
```

## 3.2 函数

```go
// DisableBindValidation 关闭默认的验证器
func DisableBindValidation()

// DisableConsoleColor 关闭终端输出的颜色
func DisableConsoleColor()

// ForceConsoleColor 终端输出颜色
func ForceConsoleColor()

// Mode 返回当前的模式
func Mode() string

// SetMode 设置模式
func SetMode(value string)
```

## 3.3 Context

Content 可以在 middleware 之间传递变量，检验请求 JSON 的格式，生成返回的 JSON。

```go
type Context struct {
	Request *http.Request
	Writer  ResponseWriter

	Params Params

	Keys map[string]any // 保存KV数据，用于上下文间的数据共享

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string

	// contains filtered or unexported fields
	handlers HandlersChain // 从路由树取出来的中间件 handlers
	index    int8          // 执行到的 handlers 索引位置
	fullPath string

	engine       *Engine
	params       *Params
	skippedNodes *[]skippedNode
}
```

方法：

```go
// Bind 将请求体绑定到结构体
func (c *Context) Bind(obj any) error
func (c *Context) BindHeader(obj any) error
func (c *Context) BindJSON(obj any) error
func (c *Context) BindQuery(obj any) error
func (c *Context) BindUri(obj any) error
func (c *Context) BindWith(obj any, b binding.Binding) error
func (c *Context) MustBindWith(obj any, b binding.Binding) error
func (c *Context) ShouldBind(obj any) error
func (c *Context) ShouldBindWith(obj any, b binding.Binding) error

// Abort 防止等待中的 handler 被调用，但不会停止它
func (c *Context) Abort()
func (c *Context) AbortWithError(code int, err error) *Error
func (c *Context) AbortWithStatus(code int)
func (c *Context) AbortWithStatusJSON(code int, jsonObj any)

// ClientIP 返回客户端的IP
func (c *Context) ClientIP() string

// ContentType 获取content-type
func (c *Context) ContentType() string

// Cookie 获取cookie
func (c *Context) Cookie(name string) (string, error)

// SetCookie 设置cookie
func (c *Context) SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool)

// FullPath 请求路径匹配的路由
func (c *Context) FullPath() string

// Param 获取url参数
func (c *Context) Param(key string) string

// Get 根据key获取value
func (c *Context) Get(key string) (value any, exists bool)

// Set 设置key/value
func (c *Context) Set(key string, value any)

// GetPostForm 从POST请求体获取key的value
func (c *Context) GetPostForm(key string) (string, bool)
func (c *Context) PostForm(key string) (value string)

// GetQuery 从url参数获取key的value
func (c *Context) GetQuery(key string) (string, bool)
func (c *Context) Query(key string) (value string)

// GetRawData 获取流数据
func (c *Context) GetRawData() ([]byte, error)

// Data 将数据写入返回体
func (c *Context) Data(code int, contentType string, data []byte)
func (c *Context) String(code int, format string, values ...any)
func (c *Context) TOML(code int, obj any)
func (c *Context) XML(code int, obj any)
func (c *Context) YAML(code int, obj any)

// Error 将error放到当前context
func (c *Context) Error(err error) *Error

// Status 设置HTTP返回码
func (c *Context) Status(code int)

// HTML 根据模板返回响应内容
func (c *Context) HTML(code int, name string, obj any)

// JSON 将响应序列化
func (c *Context) JSON(code int, obj any)

// Header 获取响应的头
func (c *Context) Header(key, value string)

// Handler 返回handler
func (c *Context) Handler() HandlerFunc
func (c *Context) HandlerName() string
func (c *Context) HandlerNames() []string

// Next 在middleware中调用，执行中间件链中下一个
func (c *Context) Next()

// Redirect 重定向
func (c *Context) Redirect(code int, location string)
```

## 3.4 Engine

Engine 是框架的实例，包含了中间件、配置。

```go
type Engine struct {
	RouterGroup

	RemoteIPHeaders []string

	HTMLRender render.HTMLRender
	FuncMap    template.FuncMap

	// ...
	// contains filtered or unexported fields
	pool             sync.Pool   // gin.Context 缓存池
	trees            methodTrees // 不同 HTTP 方法的路由树
}
```

新建对象：

```go
// Default 包含默认的 Logger 和 Recovery 中间件
func Default() *Engine

// New 无中间件
func New() *Engine
```

方法：

```go
// Run 监听端口并启动HTTP服务
func (engine *Engine) Run(addr ...string) (err error)

// Routes 获取已绑定的routes
func (engine *Engine) Routes() (routes RoutesInfo)

// Use 绑定中间件
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes
```

## 3.5 Handler

HandlerFunc 定义了以 gin.Context 为参数的函数，用于中间件返回。

```go
type HandlerFunc func(*Context)
```

一些 Gin 实现的中间件：

```go
// BasicAuth 基础的 HTTP 验证
func BasicAuth(accounts Accounts) HandlerFunc

// Bind 根据参数返回中间件
func Bind(val any) HandlerFunc

// Logger 打印日志
func Logger() HandlerFunc

// CustomRecovery 从 panic 中恢复
func Recovery() HandlerFunc
func CustomRecovery(handle RecoveryFunc) HandlerFunc
```

HandlersChain 是 HandlerFunc 的切片。

```go
type HandlersChain []HandlerFunc
```

方法：

```go
// Last 返回最后一个handler，也就是主handler
func (c HandlersChain) Last() HandlerFunc
```

RecoveryFunc 定义了 CustomRecovery 用到的参数。

```go
type RecoveryFunc func(c *Context, err any)
```

## 3.6 Route

IRouter 定义了所有路由，包括单个路由和路由组。

```go
type IRouter interface {
	IRoutes
	Group(string, ...HandlerFunc) *RouterGroup
}
```

IRoutes 定义了路由。

```go
type IRoutes interface {
	Use(...HandlerFunc) IRoutes

	Handle(string, string, ...HandlerFunc) IRoutes
	Any(string, ...HandlerFunc) IRoutes
	GET(string, ...HandlerFunc) IRoutes
	POST(string, ...HandlerFunc) IRoutes
	DELETE(string, ...HandlerFunc) IRoutes
	PATCH(string, ...HandlerFunc) IRoutes
	PUT(string, ...HandlerFunc) IRoutes
	OPTIONS(string, ...HandlerFunc) IRoutes
	HEAD(string, ...HandlerFunc) IRoutes
	Match([]string, string, ...HandlerFunc) IRoutes

	StaticFile(string, string) IRoutes
	StaticFileFS(string, string, http.FileSystem) IRoutes
	Static(string, string) IRoutes
	StaticFS(string, http.FileSystem) IRoutes
}
```

RouteInfo 表示路由的详细内容。

```go
type RouteInfo struct {
	Method      string
	Path        string
	Handler     string
	HandlerFunc HandlerFunc
}
```

RoutesInfo 表示 RouteInfo 的切片。

```go
type RoutesInfo []RouteInfo
```

RouterGroup 表示路由组，包含前缀以及一系列的中间件。

```go
type RouterGroup struct {
	Handlers HandlersChain
	// contains filtered or unexported fields
}
```

方法：

```go
// BasePath 返回路由组的基础路径
func (group *RouterGroup) BasePath() string

// Group 创建下层的路由组
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup

// Use 添加中间件
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes

// Any 注册匹配所有 HTTP 方法的路由
func (group *RouterGroup) Any(relativePath string, handlers ...HandlerFunc) IRoutes

// 注册不同的 HTTP 方法
func (group *RouterGroup) Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) Match(methods []string, relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) PUT(relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) HEAD(relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) OPTIONS(relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) PATCH(relativePath string, handlers ...HandlerFunc) IRoutes
func (group *RouterGroup) DELETE(relativePath string, handlers ...HandlerFunc) IRoutes

```

## 3.7 Param

Param 表示一个 URL 参数。

```go
type Param struct {
	Key   string
	Value string
}
```

Params 表示 Param 的切片。

```go
type Params []Param
```

方法：

```go
// 获取参数对应的value
func (ps Params) ByName(name string) (va string)
func (ps Params) Get(name string) (string, bool)
```

## 3.8 ResponseWriter

ResponseWriter 表示写入响应的接口。

```go
type ResponseWriter interface {
	http.ResponseWriter
	http.Hijacker
	http.Flusher
	http.CloseNotifier

	// Status HTTP 返回码
	Status() int

	// Size 已写入的字节数
	Size() int

	// WriteString 写入响应体
	WriteString(string) (int, error)

	// Written 是否已写入响应体
	Written() bool

	// WriteHeaderNow 写入响应头
	WriteHeaderNow()

	// Pusher 获取 http.Pusher
	Pusher() http.Pusher
}
```

# 4. 原理分析

## 4.1 流程与数据结构

Gin 是在 Go 的标准库 net/http 的基础之上进行封装而成，一个基础的服务启动包括了调用 gin.Default() 创建 gin.Engine 对象，注册路由，以及 Engine.Run() 启动 HTTP 服务监听端口三部分。

它们之间的交互流程如下图：

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/gin_procedure.png)

其中设计的数据结构主要有用于缓存 gin.Context 对象的 sync.Pool，储存路由组的 RouterGroup，每个 HTTP 方法对应的路由树 gin.methodTrees。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/gin_struct.jpg)

## 4.2 服务启动

首先需要调用 gin.Default() 创建一个 Engine 对象，将会创建一个路由树，然后创建一个 gin.Context 池用于后面的使用。

```go
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery()) // 默认中间件
	return engine
}

func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{ // 初始化路由树
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
		ForwardedByClientIP:    true,
		RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
		TrustedPlatform:        defaultPlatform,
		UseRawPath:             false,
		RemoveExtraSlash:       false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
		trees:                  make(methodTrees, 0, 9),
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJSONPrefix:       "while(1);",
		trustedProxies:         []string{"0.0.0.0/0", "::/0"},
		trustedCIDRs:           defaultTrustedCIDRs,
	}
	engine.RouterGroup.engine = engine
	engine.pool.New = func() any { // gin.Context 池
		return engine.allocateContext(engine.maxParams)
	}
	return engine
}
```

然后创建路由，将对应的函数加到路由树上。

这里以 POST 请求的路由添加为例：

```go
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPost, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath) // 绝对路径
	handlers = group.combineHandlers(handlers) // 将路由组的 handlers 加上参数传入的 handlers 作为该请求的 handlers，最后一个是该请求的业务处理函数
	group.engine.addRoute(httpMethod, absolutePath, handlers) // 将 HTTP 方法、绝对路径、handlers 加入路由树
	return group.returnObj()
}
```

这里将请求添加到路由树里，每个 HTTP 方法对应一个路由树。

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)

	root := engine.trees.get(method) // 每个 HTTP 方法对应一个路由树
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	root.addRoute(path, handlers)

	if paramsCount := countParams(path); paramsCount > engine.maxParams { // 获取路由规则里的变量
		engine.maxParams = paramsCount
	}

	if sectionsCount := countSections(path); sectionsCount > engine.maxSections {
		engine.maxSections = sectionsCount
	}
}
```

然后调用 Engine.Run() 启动服务，其内部实际上是调用 http.ListenAndServe() 来监听端口，然后调用 accept() 和 serve() 函数来监听并服务接口调用。

```go
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address) // 端口，默认 8080
	err = http.ListenAndServe(address, engine.Handler()) // 启动 http web server 监听端口
	return
}
```

http.ListenAndServe 的第二个参数是 http.Handler 接口，需要实现 ServeHTTP() 方法，而 gin.Engine 则实现了该方法。在这里 Gin 使用了 sync.Pool 来存储和复用 gin.Context 对象，降低了性能损耗。

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context) // 从 gin.Context 池获取对象
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c) // 处理 http 请求

	engine.pool.Put(c) // 将对象放回池子里
}
```

## 4.3 中间件

路由组结构体定义如下：

```go
type RouterGroup struct {
	Handlers HandlersChain // 中间件 handler 切片
	basePath string        // 基础路径
	engine   *Engine       // Engine 对象的指针
	root     bool          // 当前路由组对象是否为根路由
}
```

初始化 gin.Engine 对象时，创建了一个 Handlers 为 nil，basePath 为 "/"，root 为 true 的路由组对象。

而在根路由组对象中继续创建子路由组，basePath 等于父路由组的加上相对路径，如果调用 Group() 方法传入了 handlers 也就是中间件函数的 handler，则追加到后面。

```go
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers), // 父路由组的 handlers 追加 handlers
		basePath: group.calculateAbsolutePath(relativePath), // 将相对路径追加到 basePath
		engine:   group.engine,
	}
}
```

注册中间件时，如果在 Engine 对象注册中间件，则注册到根路由组。如果是在 RouterGroup 对象直接通过调用 Use 方法，则直接讲中间件注册到当前对象的 Handlers。如果是注册请求时带了中间件，则将路由组的 handlers 加上参数传入的 handlers 作为该请求的 handlers，再将 HTTP 方法、绝对路径、handlers 加入路由树。

```go
// Engine 对象注册中间件
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...) // 注册到根路由组
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}

// 路由组注册中间件
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...) // 追加到 Handlers
	return group.returnObj()
}

// 注册请求
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPost, relativePath, handlers)
}
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath) // 绝对路径
	handlers = group.combineHandlers(handlers) // 将路由组的 handlers 加上参数传入的 handlers 作为该请求的 handlers，最后一个是该请求的业务处理函数
	group.engine.addRoute(httpMethod, absolutePath, handlers) // 将 HTTP 方法、绝对路径、handlers 加入路由树
	return group.returnObj()
}
```

注册请求时，路由组的 handlers 和请求的 handler 合起来，放入路由树。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/gin_post_handler.jpg)

执行一个 HTTP 请求时调用 Engine.handleHTTPRequest。

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method
	rPath := c.Request.URL.Path

	// 根据 HTTP 方法，从路由树切片中找到对应的路由树
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root

		// 根据路径从路由树找到节点
		value := root.getValue(rPath, c.params, c.skippedNodes, unescape)
		if value.params != nil {
			c.Params = *value.params
		}
		if value.handlers != nil {
			c.handlers = value.handlers // 将中间件 handlers 设置到 gin.Context 中
			c.fullPath = value.fullPath
			c.Next() // 依次执行 handlers 的中间件函数
			c.writermem.WriteHeaderNow()
			return
		}
		break
	}

    // 找不到路由，404
	c.handlers = engine.allNoRoute
	serveError(c, http.StatusNotFound, default404Body)
}
```

主要流程步骤包含：

* 根据 HTTP 方法，从路由树切片中找到对应的路由树
* 根据路径从路由树找到节点
* 将中间件 handlers 设置到 gin.Context 中
* 依次执行 handlers 的中间件函数

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/gin_handle_request.jpg)

上面找到对应路由树的节点后，获取了该路由请求所添加的中间件 handlers，调用 c.Next() 方法开始依次遍历调用中间件函数，handlers 中最后一个是该请求的业务处理函数。

```go
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c) // 执行中间件 handlers 函数
		c.index++ // 索引指向下一位置
	}
}
```

在一个中间件函数中，可以调用 c.Next()，这样前面的代码段就在执行请求前执行，后面的代码段就在执行完请求后执行。且多个中间件函数之间存在嵌套关系，执行请求前依次执行中间件函数的前半段，执行完请求后就会反序依次执行中间件函数的后半段。

```go
func handler1() gin.HandlerFunc {
	return func(c *gin.Context) {
		// prev process
		c.Next()
		// post process
	}
}

func handler2() gin.HandlerFunc {
	return func(c *gin.Context) {
		// prev process
		c.Next()
		// post process
	}
}

func handler3() gin.HandlerFunc {
	return func(c *gin.Context) {
		// prev process
		c.Next()
		// post process
	}
}
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/gin_next_handler.jpg)

Gin 限制了 handlers 索引到达 63 时会 panic，也就是一个请求最多只支持 62 个中间件。

用户可以在某个 handler 中通过调用 Context.Abort 方法实现 handlers 链路的提前熔断。

## 4.4 路由树

路由树存储在 Engine.trees 成员中，每个 HTTP 方法有一个自己的路由树，注册请求时会遍历 Engine.trees 找到对应的那个路由树进行注册。

```go
// 不同 HTTP 方法的路由树
type methodTrees []methodTree

// 路由树
type methodTree struct {
	method string // HTTP 方法
	root   *node  // 根节点
}

// 节点
type node struct {
	path      string // 当前一层的路由路径名称
	indices   string
	wildChild bool
	nType     nodeType
	priority  uint32
	children  []*node // 子结点
	handlers  HandlersChain // 中间件切片
	fullPath  string // 完整路径
}
```

前缀树（trie 树）是一种树形数据结构，是一种基于字符串公共前缀构建索引的树状结构，它的特点是：

* 除根节点之外，每个节点对应一个字符
* 从根节点到某一节点，路径上经过的字符串联起来，即为该节点对应的字符串
* 尽可能复用公共前缀，如无必要不分配新的节点

而 Gin 使用了压缩前缀树（radix 树），它在前缀树的基础上进行了优化，倘若某个子节点是其父节点的唯一孩子，则与父节点进行合并，减少了存储和查询的损耗。

如图从左边几个已注册的路由，生成了右边的压缩前缀树。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/gin_radix_tree.jpg)

# 5. 参考

* [文档 | Gin Web Framework](https://gin-gonic.com/zh-cn/docs/)
* [解析 Gin 框架底层原理 - 知乎](https://zhuanlan.zhihu.com/p/611116090)

