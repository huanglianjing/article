# 1. 介绍

文档地址：https://pkg.go.dev/net/http

Go 标准库 net/http 提供了 HTTP 的客户端和服务端功能实现。

通过 Get 请求获取数据：

```go
resp, err := http.Get("http://example.com/")
defer resp.Body.Close()
body, err := io.ReadAll(resp.Body)
```

注册接口并启动 HTTP 服务监听请求：

```go
type countHandler struct {
	mu sync.Mutex // guards n
	n  int
}
func (h *countHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	h.mu.Lock()
	defer h.mu.Unlock()
	h.n++
	fmt.Fprintf(w, "count is %d\n", h.n)
}
http.Handle("/count", new(countHandler))

f := func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
}
http.HandleFunc("/bar", f)

log.Fatal(http.ListenAndServe(":8080", nil))
```

# 2. 常量

HTTP 请求方法：

```go
const (
	MethodGet     = "GET"
	MethodHead    = "HEAD"
	MethodPost    = "POST"
	MethodPut     = "PUT"
	MethodPatch   = "PATCH" // RFC 5789
	MethodDelete  = "DELETE"
	MethodConnect = "CONNECT"
	MethodOptions = "OPTIONS"
	MethodTrace   = "TRACE"
)
```

HTTP 状态码：

```go
const (
	StatusContinue           = 100 // RFC 9110, 15.2.1
	StatusSwitchingProtocols = 101 // RFC 9110, 15.2.2
	StatusProcessing         = 102 // RFC 2518, 10.1
	StatusEarlyHints         = 103 // RFC 8297

	StatusOK                   = 200 // RFC 9110, 15.3.1
	StatusCreated              = 201 // RFC 9110, 15.3.2
	StatusAccepted             = 202 // RFC 9110, 15.3.3
	StatusNonAuthoritativeInfo = 203 // RFC 9110, 15.3.4
	StatusNoContent            = 204 // RFC 9110, 15.3.5
	StatusResetContent         = 205 // RFC 9110, 15.3.6
	StatusPartialContent       = 206 // RFC 9110, 15.3.7
	StatusMultiStatus          = 207 // RFC 4918, 11.1
	StatusAlreadyReported      = 208 // RFC 5842, 7.1
	StatusIMUsed               = 226 // RFC 3229, 10.4.1

	StatusMultipleChoices  = 300 // RFC 9110, 15.4.1
	StatusMovedPermanently = 301 // RFC 9110, 15.4.2
	StatusFound            = 302 // RFC 9110, 15.4.3
	StatusSeeOther         = 303 // RFC 9110, 15.4.4
	StatusNotModified      = 304 // RFC 9110, 15.4.5
	StatusUseProxy         = 305 // RFC 9110, 15.4.6

	StatusTemporaryRedirect = 307 // RFC 9110, 15.4.8
	StatusPermanentRedirect = 308 // RFC 9110, 15.4.9

	StatusBadRequest                   = 400 // RFC 9110, 15.5.1
	StatusUnauthorized                 = 401 // RFC 9110, 15.5.2
	StatusPaymentRequired              = 402 // RFC 9110, 15.5.3
	StatusForbidden                    = 403 // RFC 9110, 15.5.4
	StatusNotFound                     = 404 // RFC 9110, 15.5.5
	StatusMethodNotAllowed             = 405 // RFC 9110, 15.5.6
	StatusNotAcceptable                = 406 // RFC 9110, 15.5.7
	StatusProxyAuthRequired            = 407 // RFC 9110, 15.5.8
	StatusRequestTimeout               = 408 // RFC 9110, 15.5.9
	StatusConflict                     = 409 // RFC 9110, 15.5.10
	StatusGone                         = 410 // RFC 9110, 15.5.11
	StatusLengthRequired               = 411 // RFC 9110, 15.5.12
	StatusPreconditionFailed           = 412 // RFC 9110, 15.5.13
	StatusRequestEntityTooLarge        = 413 // RFC 9110, 15.5.14
	StatusRequestURITooLong            = 414 // RFC 9110, 15.5.15
	StatusUnsupportedMediaType         = 415 // RFC 9110, 15.5.16
	StatusRequestedRangeNotSatisfiable = 416 // RFC 9110, 15.5.17
	StatusExpectationFailed            = 417 // RFC 9110, 15.5.18
	StatusTeapot                       = 418 // RFC 9110, 15.5.19 (Unused)
	StatusMisdirectedRequest           = 421 // RFC 9110, 15.5.20
	StatusUnprocessableEntity          = 422 // RFC 9110, 15.5.21
	StatusLocked                       = 423 // RFC 4918, 11.3
	StatusFailedDependency             = 424 // RFC 4918, 11.4
	StatusTooEarly                     = 425 // RFC 8470, 5.2.
	StatusUpgradeRequired              = 426 // RFC 9110, 15.5.22
	StatusPreconditionRequired         = 428 // RFC 6585, 3
	StatusTooManyRequests              = 429 // RFC 6585, 4
	StatusRequestHeaderFieldsTooLarge  = 431 // RFC 6585, 5
	StatusUnavailableForLegalReasons   = 451 // RFC 7725, 3

	StatusInternalServerError           = 500 // RFC 9110, 15.6.1
	StatusNotImplemented                = 501 // RFC 9110, 15.6.2
	StatusBadGateway                    = 502 // RFC 9110, 15.6.3
	StatusServiceUnavailable            = 503 // RFC 9110, 15.6.4
	StatusGatewayTimeout                = 504 // RFC 9110, 15.6.5
	StatusHTTPVersionNotSupported       = 505 // RFC 9110, 15.6.6
	StatusVariantAlsoNegotiates         = 506 // RFC 2295, 8.1
	StatusInsufficientStorage           = 507 // RFC 4918, 11.5
	StatusLoopDetected                  = 508 // RFC 5842, 7.2
	StatusNotExtended                   = 510 // RFC 2774, 7
	StatusNetworkAuthenticationRequired = 511 // RFC 6585, 6
)
```

限制：

```go
const DefaultMaxHeaderBytes = 1 << 20 // 1 MB
const DefaultMaxIdleConnsPerHost = 2
```

# 3. 变量

默认客户端：

```go
var DefaultClient = &Client{}
```

默认服务端：

```go
var DefaultServeMux = &defaultServeMux
```

# 4. 类型

## 4.1 Client

Client 表示一个 HTTP 客户端。

类型定义：

```go
type Client struct {
	// Transport specifies the mechanism by which individual
	// HTTP requests are made.
	// If nil, DefaultTransport is used.
	Transport RoundTripper

	// CheckRedirect specifies the policy for handling redirects.
	CheckRedirect func(req *Request, via []*Request) error

	// Jar specifies the cookie jar.
	Jar CookieJar

	// Timeout specifies a time limit for requests made by this
	// Client.
	Timeout time.Duration
}
```

方法：

```go
// Do 发送请求并获取响应数据
func (c *Client) Do(req *Request) (*Response, error)

// Get 发送 GET 请求
func (c *Client) Get(url string) (resp *Response, err error)

// Post 发送 POST 请求
func (c *Client) Post(url, contentType string, body io.Reader) (resp *Response, err error)

// PostForm 发送 form 表单的 POST 请求
func (c *Client) PostForm(url string, data url.Values) (resp *Response, err error)

// Head 发送 HEAD 请求
func (c *Client) Head(url string) (resp *Response, err error)

// CloseIdleConnections 关闭连接
func (c *Client) CloseIdleConnections()
```

## 4.2 Cookie

Cookie 表示请求的 cookie。

类型定义：

```go
type Cookie struct {
	Name  string
	Value string

	Path       string    // optional
	Domain     string    // optional
	Expires    time.Time // optional
	RawExpires string    // for reading cookies only

	// MaxAge=0 means no 'Max-Age' attribute specified.
	// MaxAge<0 means delete cookie now, equivalently 'Max-Age: 0'
	// MaxAge>0 means Max-Age attribute present and given in seconds
	MaxAge   int
	Secure   bool
	HttpOnly bool
	SameSite SameSite
	Raw      string
	Unparsed []string // Raw text of unparsed attribute-value pairs
}
```

方法：

```go
// String 返回内容
func (c *Cookie) String() string

// Valid 是否有效
func (c *Cookie) Valid() error
```

## 4.3 Header

Header 表示 HTTP 请求的头部。

类型定义：

```go
type Header map[string][]string
```

方法：

```
// Add 添加key/value
func (h Header) Add(key, value string)

// Set 设置key对应的value
func (h Header) Set(key, value string)

// Get 获取key对应的value
func (h Header) Get(key string) string

// Del 删除key
func (h Header) Del(key string)

// Clone 复制
func (h Header) Clone() Header
```

## 4.4 Request

Request 表示 HTTP 请求。

类型定义：

```go
type Request struct {
	// Method specifies the HTTP method (GET, POST, PUT, etc.).
	Method string

	// URL specifies either the URI being requested (for server
	// requests) or the URL to access (for client requests).
	URL *url.URL

	// The protocol version for incoming server requests.
	Proto      string // "HTTP/1.0"
	ProtoMajor int    // 1
	ProtoMinor int    // 0

	// Header contains the request header fields either received
	// by the server or to be sent by the client.
	Header Header

	// Body is the request's body.
	Body io.ReadCloser

	// GetBody defines an optional func to return a new copy of
	// Body.
	GetBody func() (io.ReadCloser, error)

	// ContentLength records the length of the associated content.
	ContentLength int64

	// TransferEncoding lists the transfer encodings from outermost to
	// innermost.
	TransferEncoding []string

	// Close indicates whether to close the connection after
	// replying to this request (for servers) or after sending this
	// request and reading its response (for clients).
	Close bool

	// For server requests, Host specifies the host on which the
	// URL is sought.
	// For client requests, Host optionally overrides the Host
	// header to send.
	Host string

	// Form contains the parsed form data, including both the URL
	// field's query parameters and the PATCH, POST, or PUT form data.
	Form url.Values

	// PostForm contains the parsed form data from PATCH, POST
	// or PUT body parameters.
	PostForm url.Values

	// MultipartForm is the parsed multipart form, including file uploads.
	MultipartForm *multipart.Form

	// Trailer specifies additional headers that are sent after the request
	// body.
	Trailer Header

	// RemoteAddr allows HTTP servers and other software to record
	// the network address that sent the request, usually for
	// logging.
	RemoteAddr string

	// RequestURI is the unmodified request-target of the
	// Request-Line (RFC 7230, Section 3.1.1) as sent by the client
	// to a server.
	RequestURI string

	// TLS allows HTTP servers and other software to record
	// information about the TLS connection on which the request
	// was received.
	TLS *tls.ConnectionState

	// Cancel is an optional channel whose closure indicates that the client
	// request should be regarded as canceled.
	Cancel <-chan struct{}

	// Response is the redirect response which caused this request
	// to be created.
	Response *Response

	// contains filtered or unexported fields
}
```

函数：

```go
// 新建请求
func NewRequest(method, url string, body io.Reader) (*Request, error)
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error)

// ReadRequest 从 Reader 读取请求
func ReadRequest(b *bufio.Reader) (*Request, error)
```

方法：

```go
// AddCookie 添加 cookie
func (r *Request) AddCookie(c *Cookie)

// 返回 cookie
func (r *Request) Cookie(name string) (*Cookie, error)
func (r *Request) Cookies() []*Cookie

// Write 写到 Writer
func (r *Request) Write(w io.Writer) error
```

## 4.5 Response

Response 表示 HTTP 响应数据。

类型定义：

```go
type Response struct {
	Status     string // e.g. "200 OK"
	StatusCode int    // e.g. 200
	Proto      string // e.g. "HTTP/1.0"
	ProtoMajor int    // e.g. 1
	ProtoMinor int    // e.g. 0

	// Header maps header keys to values.
	Header Header

	// Body represents the response body.
	Body io.ReadCloser

	// ContentLength records the length of the associated content.
	ContentLength int64

	// Contains transfer encodings from outer-most to inner-most.
	TransferEncoding []string

	// Close records whether the header directed that the connection be
	// closed after reading Body.
	Close bool

	// Uncompressed reports whether the response was sent compressed but
	// was decompressed by the http package.
	Uncompressed bool

	// Trailer maps trailer keys to values in the same format as Header.
	Trailer Header

	// Request is the request that was sent to obtain this Response.
	Request *Request

	// TLS contains information about the TLS connection on which the
	// response was received. It is nil for unencrypted responses.
	TLS *tls.ConnectionState
}
```

函数：

```go
// Get 发送 GET 请求
func Get(url string) (resp *Response, err error)

// Post 发送 POST 请求
func Post(url, contentType string, body io.Reader) (resp *Response, err error)

// PostForm 发送 form 表单的 POST 请求
func PostForm(url string, data url.Values) (resp *Response, err error)

// Head 发送 HEAD 请求
func Head(url string) (resp *Response, err error)
```

方法：

```go
// Cookies 获取 cookie
func (r *Response) Cookies() []*Cookie
```

## 4.6 Server

Server 表示运行 HTTP 服务端的参数。

类型定义：

```go
type Server struct {
	// Addr optionally specifies the TCP address for the server to listen on,
	// in the form "host:port".
	Addr string

	Handler Handler // handler to invoke, http.DefaultServeMux if nil

	DisableGeneralOptionsHandler bool

	TLSConfig *tls.Config
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)

	ReadTimeout time.Duration
	ReadHeaderTimeout time.Duration
	WriteTimeout time.Duration
	IdleTimeout time.Duration

	// MaxHeaderBytes controls the maximum number of bytes the
	// server will read parsing the request header's keys and
	// values, including the request line.
	MaxHeaderBytes int

	// ConnState specifies an optional callback function that is
	// called when a client connection changes state.
	ConnState func(net.Conn, ConnState)

	// ErrorLog specifies an optional logger for errors accepting
	// connections, unexpected behavior from handlers, and
	// underlying FileSystem errors.
	ErrorLog *log.Logger

	BaseContext func(net.Listener) context.Context
	ConnContext func(ctx context.Context, c net.Conn) context.Context

	// contains filtered or unexported fields
}
```

方法：

```go
// 启动服务端
func (srv *Server) ListenAndServe() error
func (srv *Server) ListenAndServeTLS(certFile, keyFile string) error

// SetKeepAlivesEnabled 设置请求保持活跃
func (srv *Server) SetKeepAlivesEnabled(v bool)

// 接收连接请求
func (srv *Server) Serve(l net.Listener) error
func (srv *Server) ServeTLS(l net.Listener, certFile, keyFile string) error

// Shutdown 等待请求之行结束后关闭服务端
func (srv *Server) Shutdown(ctx context.Context) error

// Close 立刻关闭服务端
func (srv *Server) Close() error
```

## 4.7 ServerMux

ServerMux 表示接收 HTTP 请求的路由表。

类型定义：

```go
type ServeMux struct {
	// contains filtered or unexported fields
}
```

函数：

```go
// NewServeMux 创建
func NewServeMux() *ServeMux
```

方法：

```go
// 注册路由
func (mux *ServeMux) Handle(pattern string, handler Handler)
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request))

// Handler 获取某个请求的处理 handler
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string)

// ServeHTTP 将请求分发到 handler
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request)
```

# 5. 函数

```go
// Handle 在默认服务端 DefaultServeMux 下，注册请求路径，以及实现 Handler 接口的变量
func Handle(pattern string, handler Handler)

// HandleFunc 在默认服务端 DefaultServeMux 下，注册请求路径，以及处理函数
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))

// ListenAndServe 监听地址启动服务
func ListenAndServe(addr string, handler Handler) error

// ListenAndServeTLS 监听地址启动 HTTPS 服务
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error

// Serve 对于每个 HTTP 连接，创建一个协程进行处理
func Serve(l net.Listener, handler Handler) error

// SetCookie 设置 cookie
func SetCookie(w ResponseWriter, cookie *Cookie)

// StatusText 获取状态码的描述
func StatusText(code int) string
```

