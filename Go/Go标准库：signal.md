# 1. 介绍

文档地址：https://pkg.go.dev/os/signal

Go 标准库 os/signal 用于处理信号。

操作系统所有的信号数字id和名称可以通过命令 `kill -l` 来查看，要对某个进程发送某个信号则可以执行 `kill -n <pid>`。

当接收到中断信号时，将信号发送至通道：

```go
c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt)

// Block until a signal is received.
s := <-c
```

当接收到任何信号时，将信号发送至通道：

```go
c := make(chan os.Signal, 1)
signal.Notify(c)

// Block until a signal is received.
s := <-c
```

# 2. 函数

```go
// Ignore 忽略信号
func Ignore(sig ...os.Signal)

// Ignored 是否已忽略某个信号
func Ignored(sig os.Signal) bool

// Notify 接收到信号后发送至通道
func Notify(c chan<- os.Signal, sig ...os.Signal)

// Reset 取消Notify对某个信号的接收
func Reset(sig ...os.Signal)

// Stop 取消Notify对某个通道的发送
func Stop(c chan<- os.Signal)
```

