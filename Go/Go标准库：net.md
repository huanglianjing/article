# 1. 介绍

文档地址：https://pkg.go.dev/net

Go 标准库 net 提供了对TCP/IP、UDP、域名、socket的网络I/O接口。

通过 Dial 访问域名：

```go
conn, err := net.Dial("tcp", "golang.org:80")
fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
status, err := bufio.NewReader(conn).ReadString('\n')
```

通过 Listen 监听端口，通过 Accept 接收请求并处理：

```go
ln, err := net.Listen("tcp", ":8080")
for {
	conn, err := ln.Accept()
	go handleConnection(conn)
}
```

# 2. 类型

## 2.1 Conn

Conn 表示一个流式的网络连接。

类型定义：

```go
type Conn interface {
	// Read reads data from the connection.
	Read(b []byte) (n int, err error)

	// Write writes data to the connection.
	Write(b []byte) (n int, err error)

	// Close closes the connection.
	Close() error

	// LocalAddr returns the local network address, if known.
	LocalAddr() Addr

	// RemoteAddr returns the remote network address, if known.
	RemoteAddr() Addr

	// SetDeadline sets the read and write deadlines associated
	// with the connection. It is equivalent to calling both
	// SetReadDeadline and SetWriteDeadline.
	SetDeadline(t time.Time) error

	// SetReadDeadline sets the deadline for future Read calls
	// and any currently-blocked Read call.
	SetReadDeadline(t time.Time) error

	// SetWriteDeadline sets the deadline for future Write calls
	// and any currently-blocked Write call.
	SetWriteDeadline(t time.Time) error
}
```

函数：

```go
// Dial 连接网络，network 可以是 tcp、udp、ip4、ip6等，address 则是IP地址或域名加上端口号
func Dial(network, address string) (Conn, error)

// DialTimeout 带超时的连接网络
func DialTimeout(network, address string, timeout time.Duration) (Conn, error)

// FileConn 连接到文件
func FileConn(f *os.File) (c Conn, err error)
```

## 2.2 IPConn

IPConn 表示一个IP网络的连接。

类型定义：

```go
type IPConn struct {
	// contains filtered or unexported fields
}
```

函数：

```go
// DialIP 连接IP网络
func DialIP(network string, laddr, raddr *IPAddr) (*IPConn, error)

// ListenIP 监听IP网络
func ListenIP(network string, laddr *IPAddr) (*IPConn, error)
```

方法：

```go
// LocalAddr 本地网络地址
func (c *IPConn) LocalAddr() Addr

// Read 读取数据
func (c *IPConn) Read(b []byte) (int, error)

// Write 写入数据
func (c *IPConn) Write(b []byte) (int, error)

// Close 关闭连接
func (c *IPConn) Close() error
```

## 2.3 Listener

Listener 表示流式网络的监听。

类型定义：

```go
type Listener interface {
	// Accept waits for and returns the next connection to the listener.
	Accept() (Conn, error)

	// Close closes the listener.
	// Any blocked Accept operations will be unblocked and return errors.
	Close() error

	// Addr returns the listener's network address.
	Addr() Addr
}
```

函数：

```go
// Listen 监听网络
func Listen(network, address string) (Listener, error)
```

