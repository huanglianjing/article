# 1. 概念

Go 语言的设计哲学之一是**显式错误处理**：函数通过返回 `error` 告知调用方出了什么问题，而不是像 Java、Python 那样使用 `try/catch` 异常机制。但 Go 并非完全没有"异常"，它提供了 `panic` 和 `recover` 这对内建函数，用来处理**真正异常的、不可预期的、程序无法继续正常执行**的情况。

## 1.1 panic

`panic` 是一个内建函数，签名如下：

```go
func panic(v any)
```

调用 `panic` 后会发生以下事情：

1. 当前函数的正常执行**立即停止**；
2. 当前函数中已注册的 `defer` 函数按照**后进先出（LIFO）**的顺序依次执行；
3. 返回到调用者，调用者表现得如同自己也调用了 `panic`，继续执行它的 `defer`；
4. 沿着调用栈一直向上，直到当前 goroutine 的所有函数都返回；
5. 如果整个过程中没有被 `recover` 捕获，程序打印 panic 信息和调用栈，并以**退出码 2** 终止。

```go
package main

import "fmt"

func main() {
    defer fmt.Println("main defer")
    foo()
    fmt.Println("这行不会执行")
}

func foo() {
    defer fmt.Println("foo defer")
    panic("something bad happened")
}
```

输出：

```go
foo defer
main defer
panic: something bad happened

goroutine 1 [running]:
main.foo()
        /tmp/main.go:13 +0x...
main.main()
        /tmp/main.go:7 +0x...
exit status 2
```

## 1.2 recover

`recover` 也是一个内建函数：

```go
func recover() any
```

它用于"捕获" panic，让程序恢复正常执行流程。其行为规则是：

- 只有在 **defer 函数中直接调用** 时才有效，而不是在 defer 函数内的某个子函数内 recover；
- 如果当前 goroutine 正在 panic，`recover` 返回传给 `panic` 的值，并终止 panic 过程；
- 如果当前 goroutine 没有 panic，或者不是在 defer 中直接调用，`recover` 返回 `nil`，没有任何效果。

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    result = a / b // b 为 0 时触发 panic
    return
}

func main() {
    res, err := safeDivide(10, 0)
    fmt.Println(res, err)
    // 输出: 0 recovered from panic: runtime error: integer divide by zero
}
```

recover 之后，函数会从 defer 处"正常返回"，此时具名返回值 `err` 已在 defer 中被修改，从而实现了"把 panic 转换为 error"。程序不会回到 panic 发生的那一行继续执行，而是让"包含该 defer 的函数"正常返回到它的调用者。

# 2. 实现原理

以下分析基于 Go 运行时源码 `src/runtime/panic.go`。

## 2.1 数据结构

每个 goroutine（运行时中的 `g` 结构体）上挂着两条链表：

```go
type g struct {
    // ...
    _panic *_panic // 当前正在进行的 panic 链表（最内层在表头）
    _defer *_defer // defer 链表（最近注册的在表头）
    // ...
}
```

`_panic` 结构体简化表示：

```go
type _panic struct {
    argp      unsafe.Pointer // 指向正在执行的 defer 调用的参数位置，用于校验 recover 调用者
    arg       any            // panic 的参数
    link      *_panic        // 指向更早的 panic（defer 中再次 panic 时形成链表）
    recovered bool           // 是否已被 recover
    goexit    bool           // 是否由 runtime.Goexit 触发
    // ... pc、sp 等用于恢复执行的字段
}
```

`_defer` 结构体记录了延迟调用的函数、栈指针 `sp`、程序计数器 `pc` 等信息，并通过 `link` 串成链表。

## 2.2 panic

无论是用户代码显式调用 `panic(v)`，还是运行时检测到错误（如数组越界），最终都会进入 `runtime.gopanic`。其主要流程如下：

1. 调用 `gopanic(e)`，创建 `_panic` 结构体 `p`，设置 `p.arg = e`。
2. 将 `p` 压入 `g._panic` 链表头部。
3. 依次取出并执行 defer，包括普通 defer 和开放编码 defer。
4. 每次执行后检查 `p.recovered`：若为 `true`，移除 `p`，调用 `recovery` 恢复执行，不再返回 `gopanic`；否则继续执行下一个 defer。
5. 所有 defer 执行完仍未 recover，则通过 `printpanics` 打印 panic 链信息，再由 `fatalpanic` 打印调用栈，以 `exit(2)` 退出。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/go_panic_gopanic.jpg)

## 2.3 recover

编译器会把 `recover()` 转换为对 `runtime.gorecover` 的调用：

```go
func gorecover(argp uintptr) any {
    gp := getg()
    p := gp._panic
    if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) {
        p.recovered = true
        return p.arg
    }
    return nil
}
```

这段代码揭示了 recover 的所有限制条件：

| 条件             | 含义                                                        |
| ---------------- | ----------------------------------------------------------- |
| `p != nil`       | 当前 goroutine 必须正在 panic                               |
| `!p.goexit`      | `runtime.Goexit` 触发的退出不能被 recover                   |
| `!p.recovered`   | 同一个 panic 只能被 recover 一次                            |
| `argp == p.argp` | **调用 recover 的函数必须是被 panic 直接调用的 defer 函数** |

recover 必须在 defer 函数中**直接**调用，`argp` 是调用者的参数指针，如果 recover 被嵌套在 defer 函数调用的另一个函数里，`argp` 对不上，recover 就返回 `nil`。

当 defer 函数返回后，`gopanic` 发现 `p.recovered` 为 true，于是调用 `recovery`。它的核心工作是：

1. 将 goroutine 的 `sp`（栈指针）和 `pc`（程序计数器）恢复到**注册该 defer 的函数**的栈帧；
2. 让执行流跳转到该函数的 `deferreturn` 处，继续执行该函数中剩余的 defer；
3. 函数随后正常返回，此时具名返回值保持 defer 中设置的值。

如果在 defer 中又发生了 panic，新的 `_panic` 会压入 `g._panic` 链表头部，旧的 panic 被标记为"已中止"（aborted）。最终如果没有 recover，会打印出整条 panic 链。

# 3. 场景

## 3.1 运行时 panic

这类 panic 的值实现了 `runtime.Error` 接口，可以被 recover 正常捕获。

**空指针解引用**

```go
var p *User
fmt.Println(p.Name)
// panic: runtime error: invalid memory address or nil pointer dereference
```

常见变种：

* 调用 nil 指针接收者的方法并在方法内访问字段、未初始化的结构体指针字段；
* 函数返回 `(nil, err)` 后忽略 err 直接使用结果；

**数组/切片越界**

```go
s := []int{1, 2, 3}
_ = s[5]
// panic: runtime error: index out of range [5] with length 3

_ = s[2:1]
// panic: runtime error: slice bounds out of range [2:1]
```

**向 nil map 写入**

```go
var m map[string]int
m["a"] = 1
// panic: assignment to entry in nil map
```

读取或 delete nil map 不会产生 panic。

**单返回值形式类型断言失败**

```go
var i any = "hello"
n := i.(int)
// panic: interface conversion: interface {} is string, not int
```

类型断言应该使用 `n, ok := i.(int)` 的双返回值形式。

**整数除以零**

```go
a, b := 10, 0
_ = a / b
// panic: runtime error: integer divide by zero
```

浮点数除以零不会 panic，结果为 `+Inf`、`-Inf` 或 `NaN`。

常量除以零在编译期就会报错。

**channel 相关操作**

```go
ch := make(chan int)
close(ch)
close(ch)
// panic: close of closed channel

ch2 := make(chan int)
close(ch2)
ch2 <- 1
// panic: send on closed channel

var ch3 chan int
close(ch3)
// panic: close of nil channel
```

从已关闭的 channel 读取会立即返回零值。

向 nil channel 收发会永久阻塞。

**调用 nil 函数**

```go
var fn func()
fn()
// panic: runtime error: invalid memory address or nil pointer dereference
```

**make 参数非法**

```go
n := -1
_ = make([]int, n)
// panic: runtime error: makeslice: len out of range
```

**切片转数组长度不足**

```go
s := []int{1, 2}
_ = [4]int(s)     // Go 1.20+
// panic: runtime error: cannot convert slice with length 2 to array or pointer to array with length 4
```

**标准库中的主动 panic**

- `regexp.MustCompile`、`template.Must` 等 `Must` 系列函数在出错时 panic；
- `sync.WaitGroup` 计数器变为负数：`panic: sync: negative WaitGroup counter`；
- `reflect` 包的误用，如对不可寻址的值调用 `Set`、对非指针调用 `Elem`；
- `strings.Repeat` 传入负数次数；
- `bytes.Buffer` 增长过大等。

**显式调用 panic**

```go
func MustParse(s string) Config {
    cfg, err := Parse(s)
    if err != nil {
        panic(err)
    }
    return cfg
}
```

**panic(nil)**

在 Go 1.21 之前，`panic(nil)` 会导致 `recover()` 返回 `nil`，调用方无法区分"没有 panic"和"panic 了 nil"。

从 **Go 1.21** 开始，`panic(nil)` 会被运行时转换为 `*runtime.PanicNilError`，`recover()` 将返回这个非 nil 的错误值（可通过 `GODEBUG=panicnil=1` 恢复旧行为）。

## 3.2 致命错误

这一类不是 panic，而是**运行时直接调用 `fatal/throw` 终止进程**，defer 不会执行，recover 也无能为力。

**并发读写 map**

```go
m := make(map[int]int)
go func() {
    for { m[1] = 1 }
}()
for { _ = m[1] }
// fatal error: concurrent map read and map write
```

**所有 goroutine 死锁**

```go
func main() {
    ch := make(chan int)
    <-ch
}
// fatal error: all goroutines are asleep - deadlock!
```

**栈溢出**

```go
func f() { f() }
// runtime: goroutine stack exceeds 1000000000-byte limit
// fatal error: stack overflow
```

**内存耗尽**

```
fatal error: runtime: out of memory
```

**解锁未加锁的 Mutex**

```go
var mu sync.Mutex
mu.Unlock()
// fatal error: sync: unlock of unlocked mutex
```

**以 nil 函数启动协程**

```go
func main() {
    var f func()
    go f()
}
```

**主动终止程序**

主动终止退出程序，也无法被 recover 捕获。

```go
log.Fatal("init failed")

os.Exit(1)
```

其他如 cgo 中的非法内存访问、运行时内部一致性检查失败等也会造成致命错误。

不要寄希望于 recover 兜底所有问题。对于并发安全问题，应通过 `sync.Mutex`、`sync.Map`、`-race` 检测等手段从根源上避免。

# 4. 最佳实践

**在 defer 函数内调用 recover**

recover 必须在 defer 函数中直接调用，且不能直接 `defer recover()`，这样才能在调用站内部发生 panic 时，层层检查 defer 函数时调用到某一层 defer 中的 recover。

在 defer 函数中嵌套一层函数调用时参数不匹配，会导致 recover 返回 nil。

以下是一些反面例子：

```go
// ❌ 错误：不在 defer 中
func bad1() {
    recover()
    panic("x")
}

// ❌ 错误：嵌套了一层函数调用
func helper() {
    recover() // 无效，argp 不匹配
}
func bad2() {
    defer func() {
        helper()
    }()
    panic("x")
}

// ❌ 错误：defer 直接调用 recover（recover 是内建函数，此时并非在 defer 函数"内部"调用）
func bad3() {
    defer recover()
    panic("x")
}
```

**recover 只捕获当前协程的 panic**

每个 goroutine 的 panic 只能由该 goroutine 自己的 defer 捕获。

任何一个 goroutine 中未被捕获的 panic，都会导致**整个进程崩溃**。

例如 HTTP 服务中，`net/http` 会为每个请求的 handler 做 recover（并打印日志），但你在 handler 里自己启动的 goroutine 并不受保护。

**defer 中的 panic 会覆盖原 panic**

这会导致原始错误信息丢失，排查问题时非常棘手。defer 中的清理逻辑要尽量保证自身不会 panic。

**recover 不会自动保留调用栈**

recover 之后，panic 时的调用栈信息就丢失了。如果只打印 `r` 的值，往往无法定位问题。务必使用 `runtime/debug.Stack()` 记录堆栈：

```go
defer func() {
    if r := recover(); r != nil {
        log.Printf("panic: %v\n%s", r, debug.Stack())
    }
}()
```

**不要静默吞掉 panic**

```go
// ❌ 最糟糕的写法
defer func() { recover() }()
```

这会隐藏程序中的 bug，让问题在更隐蔽的地方以更奇怪的方式爆发。

**库代码不要让 panic 越过包边界**

公共库的导出 API 应当返回 error，而不是 panic 给调用方。如果库内部为了简化深层递归的错误传递而使用 panic（标准库 `encoding/json`、`text/template` 内部就有类似做法），必须在导出函数的入口处 recover 并转换为 error。

**Must 模式仅用于初始化阶段**

```go
// ✅ 包级变量初始化，正则写错时程序启动即失败，便于尽早发现
var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

// ❌ 不要对运行时的用户输入使用 Must
func handler(pattern string) {
    re := regexp.MustCompile(pattern) // 用户输入非法正则会导致 panic
    _ = re
}
```

**在 goroutine 入口统一做 recover**

封装一个安全启动 goroutine 的工具函数：

```go
func SafeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("[PANIC] goroutine panic: %v\n%s", r, debug.Stack())
                // 可以在这里上报监控、告警
            }
        }()
        fn()
    }()
}

// 使用
SafeGo(func() {
    doAsyncWork()
})
```

需要把 panic 传递回调用方时，可以通过 channel 转换为 error：

```go
func RunWithError(fn func() error) <-chan error {
    errCh := make(chan error, 1)
    go func() {
        defer func() {
            if r := recover(); r != nil {
                errCh <- fmt.Errorf("panic: %v\n%s", r, debug.Stack())
            }
        }()
        errCh <- fn()
    }()
    return errCh
}
```

**在服务边界设置 recover 中间件**

服务边界的 recover 的意义在于：**单个请求的 bug 不应拖垮整个服务**。

以 HTTP 服务示例，通过中间件捕获 panic。

```go
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                // http.ErrAbortHandler 是用于主动中止请求的哨兵值，应继续抛出
                if rec == http.ErrAbortHandler {
                    panic(rec)
                }
                log.Printf("panic: %v\nmethod=%s path=%s\n%s",
                    rec, r.Method, r.URL.Path, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**防御性编程，从源头减少 panic**

```go
// 检查 nil
if user == nil {
    return errors.New("user is nil")
}

// 使用双返回值类型断言
if s, ok := v.(string); ok {
    // ...
}

// 初始化 map
m := make(map[string]int)

// 访问切片前检查长度
if idx < len(s) {
    _ = s[idx]
}

// 遵循"谁发送谁关闭"的 channel 关闭原则，必要时用 sync.Once 保证只关闭一次
var once sync.Once
closeCh := func() { once.Do(func() { close(ch) }) }
```

其他一些检测工具：

- `go vet`：静态检查常见错误；
- `go test -race`：检测数据竞争，避免 `concurrent map writes` 这类无法 recover 的致命错误；
- `staticcheck`、`nilaway` 等静态分析工具：提前发现潜在的空指针问题。

