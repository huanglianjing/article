# 1. 程序启动

## 1.1 入口函数

Go程序的入口函数，是 main 包中的 main 函数，一般定义在项目根目录的 main.go 文件中，它是必须定义的，否则项目将无法编译运行。main 函数没有参数和返回值。

我们将代码逻辑通过 main 函数进行调用。

```go
package main

import "fmt"

func main() {
    fmt.Println("hello")
}
```

## 1.2 导入包

在程序中，我们通常会导入不同的包，来实现不同的功能。

**包的搜索路径**

对于 import 语句导入的包，会先去 GOROOT 搜索对应的包，没有则再去 GOPATH/src 中的每个路径搜索，还是没有则会编译错误。

以如下路径为例，编译器将先搜索 /usr/lib/golang，没有则搜索 /root/go/src。

```bash
go env GOROOT
# /usr/lib/golang

go env GOPATH
# /root/go
```

**包的导入过程**

在编译 Go 程序时，编译器从 main 包开始，找到 import 语句并依次导入对应的包，如果这些包本身又有导入的包，则会递归地继续导入它所 import 的包。这其中重复的包只会导入一次。导入顺序如下示意图：

![go_import_order](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_import_order.png)

导入一个包将会完成以下事情：完成导入当前包所 import 的其它包、初始化常量、初始化变量、调用包的 init 函数。而执行 main 函数则会完成以下事情：完成导入 main 包所 import 的包、初始化常量、初始化变量、调用 main 包的 init 函数，调用 main 函数。

## 1.3 init函数

init 函数是定义在一个包中的初始化函数，没有参数和返回值。从包的导入过程可以看出，程序的 init 函数将在 main 函数之前执行，而各个导入包的 init 函数将会按照导入顺序依次执行。

```go
func init() {
	// do sth.
}
```

我们不能显示地调用 init 函数，在一个包可以定义多个 init 函数，但它们之间的执行顺序是没有明确定义的。

```go
func init() {
	// do sth.
}

func init() {
	// do sth.
}
```

例如在使用 pprof 进行程序分析时，需要空白导入 pprof 包，但是不调用它的任何函数，就是为了调用它的 init 函数进行初始化。

```go
import _ "net/http/pprof"
```

# 2. 退出

## 2.1 正常退出

首先一种正常退出的方式就是执行完 main 函数，或者在 main 函数中 return。

另外可以通过 os.Exit 退出并指定返回码为0。

```go
os.Exit(0)
```

## 2.2 异常退出

通过 os.Exit 退出并指定返回码为非0的值，指定程序返回码。

```go
os.Exit(1)
```

通过调用 panic 也可以引发程序的异常退出。或者在一些其它情况如引用数组下标超过范围，也会引发 panic 异常退出程序。

```go
panic("crash")
```

## 2.3 优雅退出

当我们开发服务器程序时，程序一般都会长期运行在服务器上，监听端口对请求作出相应，或者是从消息队列或其它外部数据源中获取消息进行处理，或者是需要定时地完成某些任务。这时候我们发送信号令服务直接中断，就有可能对正在响应的请求、处理的任务粗暴地中止了，影响请求的响应或是任务的完整执行。

因此，我们需要让服务实现优雅退出，也就是通过监听捕获对应信号，并且在收到信号时做出一些操作，比如完成现在正在处理的请求和任务，进行一些资源回收等收尾工作，然后再结束程序的运行。

常见终止程序的信号有：1（SIGHUP）、2（SIGINT）、3（SIGQUIT）、9（SIGKILL）、15（SIGTERM）、19（SIGSTOP），其中 SIGKILL 和 SIGSTOP 不能被捕获、阻塞、忽略。

这些信号的触发方式：

* SIGHUP：终端被关闭
* SIGINT：按下 CTRL + C
* SIGQUIT：按下 CTRL + \
* SIGKILL：kill -9 pid
* SIGTERM：kill pid
* SIGSTOP：kill -19 pid

### 2.3.1 捕获信号

如下示例代码，程序启动后开始监听几个信号，并开启新的协程来捕获信号并处理，然后执行业务逻辑。当捕获相应的信号后，进行程序退出前的清理工作，如数据库连接断开、关闭HTTP服务、打印退出日志等。

```go
func main() {
	c := make(chan os.Signal)
	signal.Notify(c, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	go func() {
		select {
		case sig := <-c:
			switch sig {
			case syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT:
				fmt.Printf("catch signal %s\n", sig)
				// 完成清理操作
				os.Exit(1)
			}
		}
	}()

	// 业务逻辑
}
```

或者换一种写法，程序启动后先执行业务逻辑，开启新的协程进行端口监听或消息队列消费等操作，然后创建一个通道捕获程序终止信号，捕获后进行程序退出前的清理工作。

```go
func main() {
	// 业务逻辑

	c := make(chan os.Signal)
	signal.Notify(c, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	sig := <-c
	switch sig {
	case syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT:
		// 完成清理操作
		os.Exit(1)
	}
}
```

### 2.3.2 使用context通知协程

如果捕获退出信号后，需要在子协程内执行一些清理操作，可以使用context来通知子协程接收终止的消息并完成清理操作。

同时通过 sync.WaitGroup 实现主协程等待子协程处理完毕后再停止运行。

```go
func main() {
	sig := make(chan os.Signal)
	signal.Notify(sig, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	ctx, cancle := context.WithCancel(context.Background())

	wg := sync.WaitGroup{}
	wg.Add(1)
	go func(ctx context.Context) {
		defer wg.Done()
		for {
			select {
			case <-ctx.Done():
				// 完成清理操作
				return
			}
		}
	}(ctx)

	<-sig     // 收到信号
	cancle()  // 取消context
	wg.Wait() // 等待所有的子协程都优雅关闭
}
```

