# 1. 概述

**gRPC** 是 Google 开源的一款高性能、跨语言的 RPC（Remote Procedure Call，远程过程调用）框架。它让你像调用本地函数一样调用远程服务上的方法，而不必关心底层的网络通信细节。gRPC 凭借 HTTP/2 与 Protobuf 带来的高性能、强类型契约和原生流式支持，已经成为 Go 微服务通信的事实标准。

gRPC 的核心特点：

- **基于 HTTP/2**：支持多路复用、头部压缩、双向流，一个 TCP 连接上可以并发处理多个请求。
- **使用 Protocol Buffers**：默认使用 Protobuf 作为接口定义语言（IDL）和序列化格式，体积小、速度快。
- **强类型契约**：通过 `.proto` 文件定义服务接口，自动生成客户端和服务端代码，前后端契约清晰。
- **跨语言**：支持 Go、Java、C++、Python、Node.js、Rust 等主流语言。
- **四种通信模式**：一元调用、服务端流、客户端流、双向流。
- **生态完善**：内置拦截器、超时控制、负载均衡、健康检查、TLS 认证等能力。

gRPC 与 REST 对比：

| 对比项      | gRPC                 | REST                 |
| ----------- | -------------------- | -------------------- |
| 传输协议    | HTTP/2               | 通常为 HTTP/1.1      |
| 数据格式    | Protobuf（二进制）   | JSON（文本）         |
| 接口契约    | `.proto` 强约束      | OpenAPI 等，可选     |
| 性能        | 高，序列化快、体积小 | 相对较低             |
| 流式通信    | 原生支持双向流       | 需借助 WebSocket/SSE |
| 浏览器支持  | 需要 gRPC-Web 或网关 | 原生支持             |
| 可读性/调试 | 二进制，需专用工具   | 直观，curl 即可      |

gRPC 的适用场景：微服务之间的内部通信、对性能和延迟敏感的系统、需要流式数据传输的场景、多语言混合的服务架构。

# 2. Protobuf

Protobuf（Protocol Buffer）是 Google 开发的独立于语言和平台的数据交换格式，采用二进制编码实现高效序列化，具有体积小、传输快的特点，是 gRPC 的基石。

以下是一个囊括基本语法表示的示例：

```protobuf
syntax = "proto3";            // 使用 proto3 语法
package user;                 // proto 包名，避免命名冲突
option go_package = "example.com/demo/proto/user;user";  // 生成的 Go 包路径;包名

// 枚举，第一个值必须为 0
enum Gender {
  GENDER_UNSPECIFIED = 0;
  GENDER_MALE = 1;
  GENDER_FEMALE = 2;
}

// 消息定义
message User {
  int64 id = 1;                       // 字段编号，同一消息内唯一，不可随意修改
  string name = 2;
  Gender gender = 3;
  repeated string tags = 4;           // 列表，对应 Go 的 []string
  map<string, string> extra = 5;      // map
  Address address = 6;                // 嵌套消息
  oneof contact {                     // 多选一
    string email = 7;
    string phone = 8;
  }
}
message Address {
  string city = 1;
  string street = 2;
}
```

字段编号是序列化的关键。已上线的字段编号不要修改或复用；删除字段时建议用 `reserved` 保留编号。

常用类型映射：

| Protobuf            | Go                    |
| ------------------- | --------------------- |
| `double` / `float`  | `float64` / `float32` |
| `int32` / `int64`   | `int32` / `int64`     |
| `uint32` / `uint64` | `uint32` / `uint64`   |
| `bool`              | `bool`                |
| `string`            | `string`              |
| `bytes`             | `[]byte`              |
| `repeated T`        | `[]T`                 |
| `map<K, V>`         | `map[K]V`             |

# 3. 环境准备

**安装 protoc 编译器**

- macOS：`brew install protobuf`
- Ubuntu：`apt install -y protobuf-compiler`
- Windows / 其他：从 [GitHub Releases](https://github.com/protocolbuffers/protobuf/releases) 下载并加入 PATH

验证：

```bash
protoc --version
```

**安装 Go 代码生成插件**

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

- `protoc-gen-go`：生成消息结构体代码（`*.pb.go`）
- `protoc-gen-go-grpc`：生成 gRPC 服务接口代码（`*_grpc.pb.go`）

确保 `$(go env GOPATH)/bin` 在 PATH 中，在当前终端直接执行，或者加入 `~/.bash_profile` 中：

```bash
export PATH="$PATH:$(go env GOPATH)/bin"
```

# 4. 示例

## 4.1 初始化项目

新建一个目录并初始化 go 项目：

```bash
mkdir grpc-demo && cd grpc-demo
go mod init grpc-demo
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

项目结构如下：

```
grpc-demo/
├── go.mod
├── proto/
│   └── hello/
│       └── hello.proto
├── server/
│   └── main.go
└── client/
    └── main.go
```

## 4.2 proto

创建 `proto/hello/hello.proto`，一次性定义 gRPC 的四种通信模式：

```protobuf
syntax = "proto3";
package hello;
option go_package = "grpc-demo/proto/hello;hello";

service Greeter {
  // 1. 一元 RPC：一个请求，一个响应
  rpc SayHello(HelloRequest) returns (HelloReply);

  // 2. 服务端流式 RPC：一个请求，多个响应
  rpc LotsOfReplies(HelloRequest) returns (stream HelloReply);

  // 3. 客户端流式 RPC：多个请求，一个响应
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloReply);

  // 4. 双向流式 RPC：多个请求，多个响应
  rpc BidiHello(stream HelloRequest) returns (stream HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

在项目根目录执行命令生成代码：

```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/hello/hello.proto
```

执行后会在 `proto/hello/` 下生成两个文件：

- `hello.pb.go`：包含 `HelloRequest`、`HelloReply` 结构体及序列化方法；
- `hello_grpc.pb.go`：包含 `GreeterServer` 接口、`GreeterClient` 客户端、`RegisterGreeterServer` 注册函数等；

## 4.3 服务端

创建 `server/main.go`：

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"strings"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/reflection"
	"google.golang.org/grpc/status"

	pb "grpc-demo/proto/hello"
)

// server 实现 pb.GreeterServer 接口
type server struct {
	// 必须嵌入，保证向前兼容：proto 新增方法时未实现也能编译通过
	pb.UnimplementedGreeterServer
}

// 1. 一元 RPC
func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
	if req.GetName() == "" {
		return nil, status.Error(codes.InvalidArgument, "name 不能为空")
	}
	return &pb.HelloReply{Message: "Hello, " + req.GetName()}, nil
}

// 2. 服务端流式 RPC
func (s *server) LotsOfReplies(req *pb.HelloRequest, stream pb.Greeter_LotsOfRepliesServer) error {
	for i := 1; i <= 5; i++ {
		// 客户端取消或超时时及时退出
		if err := stream.Context().Err(); err != nil {
			return status.FromContextError(err).Err()
		}
		reply := &pb.HelloReply{Message: fmt.Sprintf("Hello %s, 第 %d 条消息", req.GetName(), i)}
		if err := stream.Send(reply); err != nil {
			return err
		}
		time.Sleep(500 * time.Millisecond)
	}
	return nil // 返回 nil 表示流正常结束
}

// 3. 客户端流式 RPC
func (s *server) LotsOfGreetings(stream pb.Greeter_LotsOfGreetingsServer) error {
	var names []string
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			// 客户端发送完毕，返回汇总结果
			return stream.SendAndClose(&pb.HelloReply{
				Message: "Hello, " + strings.Join(names, ", "),
			})
		}
		if err != nil {
			return err
		}
		names = append(names, req.GetName())
	}
}

// 4. 双向流式 RPC
func (s *server) BidiHello(stream pb.Greeter_BidiHelloServer) error {
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		if err := stream.Send(&pb.HelloReply{Message: "Echo: " + req.GetName()}); err != nil {
			return err
		}
	}
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("监听失败: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})

	// 注册反射服务，方便 grpcurl 等工具调试
	reflection.Register(s)

	log.Println("gRPC 服务启动，监听 :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("服务启动失败: %v", err)
	}
}
```

## 4.4 客户端

创建 `client/main.go`：

```go
package main

import (
	"context"
	"io"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	pb "grpc-demo/proto/hello"
)

func main() {
	// 创建连接（grpc.NewClient 是 v1.63+ 推荐的方式，替代已废弃的 grpc.Dial）
	conn, err := grpc.NewClient("localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()), // 不使用 TLS
	)
	if err != nil {
		log.Fatalf("创建连接失败: %v", err)
	}
	defer conn.Close()

	client := pb.NewGreeterClient(conn)

	callUnary(client)
	callServerStream(client)
	callClientStream(client)
	callBidiStream(client)
}

// 1. 一元调用
func callUnary(client pb.GreeterClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "Gopher"})
	if err != nil {
		log.Fatalf("SayHello 调用失败: %v", err)
	}
	log.Printf("[Unary] %s", resp.GetMessage())
}

// 2. 服务端流
func callServerStream(client pb.GreeterClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.LotsOfReplies(ctx, &pb.HelloRequest{Name: "Gopher"})
	if err != nil {
		log.Fatalf("LotsOfReplies 调用失败: %v", err)
	}
	for {
		resp, err := stream.Recv()
		if err == io.EOF {
			break // 服务端发送完毕
		}
		if err != nil {
			log.Fatalf("接收失败: %v", err)
		}
		log.Printf("[ServerStream] %s", resp.GetMessage())
	}
}

// 3. 客户端流
func callClientStream(client pb.GreeterClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.LotsOfGreetings(ctx)
	if err != nil {
		log.Fatalf("LotsOfGreetings 调用失败: %v", err)
	}
	for _, name := range []string{"Alice", "Bob", "Carol"} {
		if err := stream.Send(&pb.HelloRequest{Name: name}); err != nil {
			log.Fatalf("发送失败: %v", err)
		}
	}
	// 关闭发送并等待服务端响应
	resp, err := stream.CloseAndRecv()
	if err != nil {
		log.Fatalf("接收失败: %v", err)
	}
	log.Printf("[ClientStream] %s", resp.GetMessage())
}

// 4. 双向流
func callBidiStream(client pb.GreeterClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.BidiHello(ctx)
	if err != nil {
		log.Fatalf("BidiHello 调用失败: %v", err)
	}

	done := make(chan struct{})
	// 接收协程
	go func() {
		defer close(done)
		for {
			resp, err := stream.Recv()
			if err == io.EOF {
				return
			}
			if err != nil {
				log.Printf("接收失败: %v", err)
				return
			}
			log.Printf("[Bidi] %s", resp.GetMessage())
		}
	}()

	// 发送
	for _, name := range []string{"one", "two", "three"} {
		if err := stream.Send(&pb.HelloRequest{Name: name}); err != nil {
			log.Fatalf("发送失败: %v", err)
		}
		time.Sleep(300 * time.Millisecond)
	}
	stream.CloseSend() // 通知服务端发送结束
	<-done
}
```

## 4.5 运行

在两个终端分别启动服务端和客户端：

```bash
# 启动服务端
go run ./server

# 启动客户端
go run ./client
```

客户端输出：

```
2026/10/26 22:00:34 [Unary] Hello, Gopher
2026/10/26 22:00:34 [ServerStream] Hello Gopher, 第 1 条消息
2026/10/26 22:00:35 [ServerStream] Hello Gopher, 第 2 条消息
2026/10/26 22:00:35 [ServerStream] Hello Gopher, 第 3 条消息
2026/10/26 22:00:36 [ServerStream] Hello Gopher, 第 4 条消息
2026/10/26 22:00:36 [ServerStream] Hello Gopher, 第 5 条消息
2026/10/26 22:00:37 [ClientStream] Hello, Alice, Bob, Carol
2026/10/26 22:00:37 [Bidi] Echo: one
2026/10/26 22:00:37 [Bidi] Echo: two
2026/10/26 22:00:38 [Bidi] Echo: three
```

# 5. 概念

## 5.1 错误处理

gRPC 定义了一套标准状态码（`google.golang.org/grpc/codes`），常用的有：

| 状态码             | 含义                     |
| ------------------ | ------------------------ |
| `OK`               | 成功                     |
| `InvalidArgument`  | 参数错误                 |
| `NotFound`         | 资源不存在               |
| `AlreadyExists`    | 资源已存在               |
| `PermissionDenied` | 无权限                   |
| `Unauthenticated`  | 未认证                   |
| `DeadlineExceeded` | 超时                     |
| `Unavailable`      | 服务不可用（通常可重试） |
| `Internal`         | 服务内部错误             |

服务端返回错误：

```go
return nil, status.Errorf(codes.NotFound, "用户 %d 不存在", id)
```

客户端解析错误：

```go
resp, err := client.SayHello(ctx, req)
if err != nil {
	st, ok := status.FromError(err)
	if ok {
		switch st.Code() {
		case codes.InvalidArgument:
			log.Printf("参数错误: %s", st.Message())
		case codes.DeadlineExceeded:
			log.Println("请求超时")
		default:
			log.Printf("调用失败: code=%s, msg=%s", st.Code(), st.Message())
		}
	}
	return
}
```

## 5.2 超时与取消

gRPC 的超时通过 `context` 传递，并且会**自动跨服务传播**（deadline 会编码进 HTTP/2 头部）。

客户端每次调用都应设置超时，避免请求无限挂起。

客户端可以设置超时时间：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
resp, err := client.SayHello(ctx, req)
```

服务端应在耗时操作中检查 `ctx`：

```go
select {
case <-ctx.Done():
	return nil, status.FromContextError(ctx.Err()).Err()
case result := <-doWork():
	return result, nil
}
```

## 5.3 Metadata

Metadata（元数据）类似 HTTP Header，用于传递认证 Token、链路追踪 ID 等非业务数据。

metadata 的 key 会被统一转为小写；以 `-bin` 结尾的 key 表示二进制值。

客户端将 metadata 塞入 `ctx` 中发送：

```go
import "google.golang.org/grpc/metadata"

ctx := metadata.AppendToOutgoingContext(context.Background(),
	"authorization", "Bearer my-token",
	"x-request-id", "abc-123",
)
resp, err := client.SayHello(ctx, req)
```

服务端读取：

```go
func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if ok {
		if ids := md.Get("x-request-id"); len(ids) > 0 {
			log.Printf("request id: %s", ids[0])
		}
	}
	// ...
}
```

服务端回传 Header / Trailer：

```go
grpc.SetHeader(ctx, metadata.Pairs("x-server", "v1"))
grpc.SetTrailer(ctx, metadata.Pairs("x-cost", "12ms"))
```

客户端通过 `grpc.Header(&header)`、`grpc.Trailer(&trailer)` 调用选项接收。

## 5.4 拦截器

拦截器（Interceptor）相当于 gRPC 的中间件，可以统一处理日志、认证、监控、限流、panic 恢复等横切逻辑。

分为四类：

- 服务端一元拦截器 `grpc.UnaryServerInterceptor`；
- 服务端流拦截器 `grpc.StreamServerInterceptor`；
- 客户端一元拦截器 `grpc.UnaryClientInterceptor`；
- 客户端流拦截器 `grpc.StreamClientInterceptor`；

社区项目 [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware) 提供了大量现成的拦截器：recovery、日志、认证、限流、重试、校验等，生产环境可直接使用。

```go
// 服务端日志拦截器
func loggingUnaryInterceptor(
	ctx context.Context,
	req any,
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (any, error) {
	start := time.Now()
	resp, err := handler(ctx, req) // 调用实际的业务方法
	log.Printf("method=%s cost=%s code=%s",
		info.FullMethod, time.Since(start), status.Code(err))
	return resp, err
}

// 服务端认证拦截器
func authUnaryInterceptor(
	ctx context.Context,
	req any,
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (any, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "缺少 metadata")
	}
	tokens := md.Get("authorization")
	if len(tokens) == 0 || tokens[0] != "Bearer my-token" {
		return nil, status.Error(codes.Unauthenticated, "token 无效")
	}
	return handler(ctx, req)
}

// 流拦截器
func loggingStreamInterceptor(
	srv any,
	ss grpc.ServerStream,
	info *grpc.StreamServerInfo,
	handler grpc.StreamHandler,
) error {
	start := time.Now()
	err := handler(srv, ss)
	log.Printf("stream method=%s cost=%s err=%v", info.FullMethod, time.Since(start), err)
	return err
}
```

服务端注册拦截器，支持链式。拦截器按注册顺序执行，前一个包裹后一个。

```go
s := grpc.NewServer(
	grpc.ChainUnaryInterceptor(
		loggingUnaryInterceptor,
		authUnaryInterceptor,
	),
	grpc.ChainStreamInterceptor(
		loggingStreamInterceptor,
	),
)
```

客户端拦截器：

```go
func clientTimingInterceptor(
	ctx context.Context,
	method string,
	req, reply any,
	cc *grpc.ClientConn,
	invoker grpc.UnaryInvoker,
	opts ...grpc.CallOption,
) error {
	start := time.Now()
	err := invoker(ctx, method, req, reply, cc, opts...)
	log.Printf("client call %s cost=%s", method, time.Since(start))
	return err
}

conn, err := grpc.NewClient("localhost:50051",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithChainUnaryInterceptor(clientTimingInterceptor),
)
```

## 5.5 TLS 安全传输

生产环境务必启用 TLS。

本地生成自签名证书用于测试：

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout server.key -out server.crt \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost"
```

服务端：

```go
import "google.golang.org/grpc/credentials"

creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
if err != nil {
	log.Fatal(err)
}
s := grpc.NewServer(grpc.Creds(creds))
```

客户端：

```go
creds, err := credentials.NewClientTLSFromFile("server.crt", "localhost")
if err != nil {
	log.Fatal(err)
}
conn, err := grpc.NewClient("localhost:50051", grpc.WithTransportCredentials(creds))
```

## 5.6 优雅关闭

gRPC 有官方的健康检查协议，Kubernetes 等平台可直接使用：

```go
import (
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

healthServer := health.NewServer()
healthpb.RegisterHealthServer(s, healthServer)
healthServer.SetServingStatus("hello.Greeter", healthpb.HealthCheckResponse_SERVING)
```

程序监听退出信号，先标记为不健康，再停止接受请求并等待现有请求完成，然后优雅关闭。

```go
import (
	"os"
	"os/signal"
	"syscall"
)

go func() {
	if err := s.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

log.Println("正在关闭服务...")
healthServer.Shutdown() // 先标记为不健康，让负载均衡摘除流量
s.GracefulStop()        // 停止接收新请求，等待进行中的请求完成
log.Println("服务已关闭")
```

# 6. 调试工具

[grpcurl](https://github.com/fullstorydev/grpcurl) 是 gRPC 版的 curl。由于服务端注册了反射服务，可以直接调用：

```bash
# 安装
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# 列出服务
grpcurl -plaintext localhost:50051 list

# 查看服务方法
grpcurl -plaintext localhost:50051 describe hello.Greeter

# 调用方法
grpcurl -plaintext -d '{"name": "Gopher"}' localhost:50051 hello.Greeter/SayHello
```

输出结果：

```json
{
  "message": "Hello, Gopher"
}
```

此外，图形化工具 **Postman**、**Apifox**、**grpcui** 也都支持 gRPC 调试。

# 7. 最佳实践

1. **Proto 规范**：统一管理 proto 文件，遵循 [Google API 设计指南](https://cloud.google.com/apis/design)；字段编号一旦发布不可修改，删除时用 `reserved`。
2. **始终嵌入 `UnimplementedXxxServer`**，保证向前兼容。
3. **每次调用都设置超时**，并在服务端尊重 `ctx` 的取消信号。
4. **复用 `ClientConn`**：`grpc.ClientConn` 是并发安全的且内部维护连接池，应全局创建一次复用，而不是每次请求都新建。
5. **使用标准错误码**，不要把所有错误都返回 `Unknown` 或 `Internal`。
6. **通过拦截器处理横切关注点**：日志、认证、panic 恢复、监控、限流。
7. **生产环境启用 TLS**，内部服务可考虑 mTLS。
8. **接入健康检查与优雅关闭**，配合 Kubernetes 实现平滑发布。
9. **消息大小限制**：默认接收上限为 4MB，大数据应使用流式传输，或通过 `grpc.MaxRecvMsgSize` 调整。
10. **Keepalive 配置**：长连接场景下合理配置 `keepalive.ServerParameters` / `keepalive.ClientParameters`，避免被中间设备断开。

# 8. 参考

* [gRPC 官方文档](https://grpc.io/docs/languages/go/)
* [grpc/grpc-go 源码](https://github.com/grpc/grpc-go)
* [Protocol Buffers Documentation](https://protobuf.dev/)

