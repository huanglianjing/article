# 1. 概述

在 Go 中，内存泄漏大致分为两类：

1. **Goroutine 泄漏**：goroutine 永远阻塞、无法退出，它的栈以及栈上引用的所有对象都无法释放。这是 Go 中最常见、最隐蔽的泄漏来源。
2. **对象引用泄漏**：对象被全局变量、长生命周期结构体、底层数组等意外持有，导致无法被回收。

Go 的内存泄漏本质上都是**本该结束生命周期的对象，仍被某处引用着**。其中 goroutine 泄漏占了绝大多数，而它的根源几乎都是“启动 goroutine 时没有想清楚它如何退出”。

定位的核心思路可以概括为三步：

1. **用指标确认**：goroutine 数、存活堆大小是否持续增长；
2. **用 pprof 对比**：通过 `-base` 对比两个时间点的 goroutine/heap profile，找出增量；
3. **顺着调用栈追溯**：找到分配点后，向上追溯“谁持有了引用”或“为什么 goroutine 无法退出”。

配合编码规范、静态检查和 goleak 测试，大部分泄漏可以在上线前被发现和消除。

# 2. 内存泄漏场景

## 2.1 goroutine

goroutine 初始栈只有 2KB，但它可能引用大量堆对象。泄漏成千上万个 goroutine 时，内存和调度开销都会很可观。

**channel 发送/接收永久阻塞**

超时返回后，后台 goroutine 向无缓冲 channel 发送数据，但已经没有接收者了。

```go
func queryWithTimeout() (string, error) {
    ch := make(chan string) // 无缓冲
    go func() {
        result := slowQuery() // 耗时操作
        ch <- result          // 超时后无人接收，永久阻塞
    }()

    select {
    case r := <-ch:
        return r, nil
    case <-time.After(time.Second):
        return "", errors.New("timeout") // 返回后不会再走到接收 channel 的 case
    }
}
```

修复：使用容量为 1 的缓冲 channel，保证发送方无论如何都能完成发送并退出。

**range 一个永不关闭的 channel**

生产者忘记 close，消费者永远阻塞在 range 上。

```go
func consumer(ch <-chan int) {
    for v := range ch {
        process(v)
    }
}
```

修复：遵守“谁发送、谁关闭”的原则，生产者结束时 `close(ch)`；或者通过 `context` 控制退出。

**没有退出条件的后台循环**

协程循环执行，永远不会退出。

```go
func startWorker() {
    go func() {
        for {
            doWork()
            time.Sleep(time.Second)
        }
    }()
}
```

修复：引入 `context.Context` 作为退出信号。

```go
func startWorker(ctx context.Context) {
    go func() {
        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done(): // 收到 ctx 结束通知后退出
                return
            case <-ticker.C:
                doWork()
            }
        }
    }()
}
```

**nil channel 操作**

对 nil channel 的发送和接收都会**永久阻塞**。如果 channel 因为某个分支没有被初始化，相关 goroutine 就会悄无声息地泄漏。

```go
var ch chan int // nil
go func() { ch <- 1 }() // 永久阻塞
```

**锁使用不当**

- `wg.Add` 和 `wg.Done` 次数不匹配，`wg.Wait()` 永远不返回；
- `Mutex` 加锁后在某个 return 分支忘记解锁，后续所有获取锁的 goroutine 全部阻塞。

```go
mu.Lock()
if err != nil {
    return err
}
mu.Unlock() // 前面已经退出，无法走到解锁
```

应该尽量使用 defer 来执行 `mu.Unlock()` 和 `wg.Done` 这一类释放资源的操作。

**context 创建后未 cancel**

创建了带 cancel 的 context，但是 cancel 被丢弃了。

```go
ctx, _ := context.WithTimeout(parent, 10*time.Second)
```

`WithCancel` / `WithTimeout` 创建的子 context 会挂在父 context 上。如果父 context 是长生命周期的（例如 `context.Background()` 派生的服务级 ctx），而子 context 从不 cancel，就会在父节点的 children map 中不断累积；`WithTimeout` 的内部定时器也要等到超时才会释放。`go vet` 会对此给出 `lostcancel` 警告。

修复：通过 defer 来 cancel。

```go
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel()
```

## 2.2 定时器

**循环中使用 time.After**

```go
for {
    select {
    case msg := <-msgCh:
        handle(msg)
    case <-time.After(5 * time.Minute):
        return
    }
}
```

如果 `msgCh` 消息很频繁，每秒会产生大量 5 分钟后才到期的 Timer，内存会明显上涨。

Go 1.23 对 Timer 实现做了改进：未被引用的 Timer/Ticker 即使没有 Stop 也可以被 GC 回收，且 `Reset` 不再需要手动排空 channel，但还是建议使用 Timer 复用减少分配。

修复：复用 Timer。

```go
timer := time.NewTimer(5 * time.Minute)
defer timer.Stop()
for {
    timer.Reset(5 * time.Minute)
    select {
    case msg := <-msgCh:
        handle(msg)
    case <-timer.C:
        return
    }
}
```

**ticker 未调用 Stop**

在 Go 1.23 之前，未 `Stop()` 的 `time.Ticker` 永远不会被回收。任何时候都应 `defer ticker.Stop()`。

## 2.3 切片与字符串

**子切片持有大数组**

```go
func getHeader() []byte {
    data := readFile() // 10MB
    return data[:16]
}
```

切片只是底层数组的一个“视图”，只要子切片存活，整个底层数组都无法回收。

修复：拷贝出需要的部分，

```go
func getHeader() []byte {
    data := readFile()
    header := make([]byte, 16)
    copy(header, data[:16])
    return header
}
```

**子字符串持有大字符串**

字符串同理，s[:10] 与原字符串共享底层内存。

```go
small := strings.Clone(big[:10])
```

**指针切片删除元素后未置 nil**

移除切片最后一个元素时，仅缩短 len，被截掉的元素依然在底层数组中被引用。

```go
type Item struct{ data [1 << 20]byte }
items := []*Item{...}
items = items[:len(items)-1]
```

`len` 之外、`cap` 之内的元素对 GC 依然可达，不会被回收。

修复：对删除的元素设置为 nil 后再从切片删除。

```go
items[len(items)-1] = nil
items = items[:len(items)-1]
```

Go 1.22+ 中 `slices.Delete`、`slices.Compact` 等函数会自动将被移除的尾部元素清零。

## 2.4 map

**map 不会缩容**

Go 的 map 在 delete 后**不会收缩底层存储**。对于流量突增后回落的场景（例如连接表、会话表），map 会保持在峰值大小。

缓解方式：

- value 存指针而不是大结构体，删除后至少 value 本身可被回收；
- 定期重建 map：把有效数据拷贝到新 map，丢弃旧 map；
- 使用分片 map 或带淘汰策略的缓存库。

**无上限缓存**

当使用 map 但是只增不减时，map 的内存容量会越来越大，若 key 空间不可控（例如以用户 ID、请求 URL 作 key），这就是一个必然的泄漏。

修复：使用带容量上限和 TTL 的缓存，例如 LRU（`hashicorp/golang-lru`）、`ristretto`、`bigcache` 等。

## 2.5 资源关闭

**HTTP Response Body 未关闭**

未关闭的 Body 会导致底层连接无法回到连接池，相关的读写 goroutine（`readLoop`/`writeLoop`）也会一直存在。

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
data, _ := io.ReadAll(resp.Body)
```

修复：获取到响应结果后立刻声明 defer 关闭。

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

**HTTP Client 没有超时**

`http.DefaultClient` 没有超时。如果对端不响应，请求 goroutine 会一直挂着。生产环境应始终配置 `Timeout` 或使用带 deadline 的 context。

```go
client := &http.Client{Timeout: 10 * time.Second}
```

另外，不要每次请求都新建 `http.Client`/`http.Transport`，否则连接池无法复用，空闲连接和相关 goroutine 会持续累积。

**数据库 Rows、文件、Stmt 未关闭**

读取完必须关闭，否则连接不会归还连接池、文件句柄不会取消。

```go
rows, err := db.Query("SELECT ...")
if err != nil {
    return err
}
defer rows.Close()
```

## 2.6 defer

defer 在**函数返回时**才执行。循环处理大量文件时，如果在循环内 defer 关闭，文件句柄和相关缓冲会一直累积。

```go
func processFiles(paths []string) error {
    for _, p := range paths {
        f, err := os.Open(p)
        if err != nil {
            return err
        }
        defer f.Close() // 所有文件在函数返回前都不会关闭
        // ...
    }
    return nil
}
```

修复：将循环内的逻辑抽成独立函数，让 defer 在独立函数内声明。

## 2.7 sync.Pool

`sync.Pool` 本身会在 GC 时被清理，但如果放回了异常大的对象（例如某次请求用到了 100MB 的 buffer），后续这块大内存会在池中被反复复用、长期驻留。

修复：放回前检查大小，超过 64KB 不回收到池中。

## 2.8 CGO 与堆外内存

通过 cgo 调用 C 代码分配的内存（`C.malloc`）不受 Go GC 管理，必须手动 `C.free`。这类泄漏在 Go 的 heap profile 中**完全看不到**，只能表现为进程 RSS 持续增长。

同样，`mmap` 映射、某些第三方库（如 RocksDB 绑定、图像处理库）也属于堆外内存。

## 2.9 Finalizer

为对象设置了 `runtime.SetFinalizer`，而对象之间存在循环引用时，GC 无法确定执行顺序，这些对象将永远不会被回收。

Go 1.24 引入的 `runtime.AddCleanup` 在设计上规避了不少此类问题，新代码可优先考虑。

# 3. 定位分析

## 3.1 确认是否真的泄漏

内存上涨不一定是泄漏，先区分清楚现象及原因：

| 现象                            | 可能原因                              |
| ------------------------------- | ------------------------------------- |
| 内存持续线性/阶梯增长，从不回落 | 很可能泄漏                            |
| 内存随流量上涨，流量降低后回落  | 正常                                  |
| Go heap 稳定，但 RSS 不下降     | 运行时未及时将内存归还 OS，或堆外内存 |
| goroutine 数持续增长            | goroutine 泄漏                        |

**关注核心指标**

如果服务接入了 Prometheus（`client_golang` 默认暴露 Go 运行时指标），重点观察：

- `go_goroutines`：goroutine 数量，是否持续增长；
- `go_memstats_heap_inuse_bytes`：堆上正在使用的内存；
- `go_memstats_heap_objects`：堆上对象数量；
- `process_resident_memory_bytes`：进程 RSS。

也可以在代码中直接读取内存大小和协程数量：

```go
var m runtime.MemStats
runtime.ReadMemStats(&m)
log.Printf("HeapInuse=%dMB HeapObjects=%d Goroutines=%d",
    m.HeapInuse>>20, m.HeapObjects, runtime.NumGoroutine())
```

**Heap 稳定但 RSS 很高**

- Go 1.12 ~ 1.15 在 Linux 上默认使用 `MADV_FREE` 归还内存，RSS 不会立刻下降。可设置 `GODEBUG=madvdontneed=1`。Go 1.16 起已改回默认 `MADV_DONTNEED`。
- Go 运行时的 scavenger 是渐进式归还内存的，存在延迟。
- 设置 `GOMEMLIMIT`（Go 1.19+）可以让运行时在接近上限时更积极地 GC 和归还内存，在容器环境中非常有用。
- 如果 heap profile 显示的内存远小于 RSS，则重点怀疑 cgo / 堆外内存 或大量 goroutine 栈。

**观察 GC 日志**

```
GODEBUG=gctrace=1 ./your-app
```

输出格式：

```
gc 15 @12.345s 2%: 0.02+3.1+0.05 ms clock, ... 120->125->80 MB, 160 MB goal, 8 P
```

其中 `120->125->80 MB` 分别表示 GC 开始时堆大小、GC 结束时堆大小、存活堆大小。

如果存活堆在多轮 GC 后持续增长，基本可以确认存在泄漏。

## 3.2 使用 pprof 定位

程序开启 pprof 运行。

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
    }()
}
```

**查看 goroutine 栈**

```bash
curl -s "http://127.0.0.1:6060/debug/pprof/goroutine?debug=1" > goroutine.txt
```

`debug=1` 会把相同栈的 goroutine 聚合并显示数量，`debug=2` 会输出每个 goroutine 的完整栈和**阻塞时长**。

通过分析 goroutine 的异常数量、异常阻塞时长来定位问题。

**使用 pprof 交互分析**

```go
go tool pprof -http=:8080 http://127.0.0.1:6060/debug/pprof/goroutine
```

在 Web 界面中查看火焰图（Flame Graph）或 Top 视图，找出数量最多的调用栈。

**定位堆内存泄漏**

```go
go tool pprof -http=:8080 http://127.0.0.1:6060/debug/pprof/heap
```

heap profile 包含四种采样指标，在 Web 界面右上角 `SAMPLE` 中切换：

| 指标            | 含义                       | 用途                  |
| --------------- | -------------------------- | --------------------- |
| `inuse_space`   | 当前仍在使用的内存大小     | **排查泄漏首选**      |
| `inuse_objects` | 当前仍在使用的对象数量     | 排查大量小对象泄漏    |
| `alloc_space`   | 程序启动以来累计分配的大小 | 排查分配热点、GC 压力 |
| `alloc_objects` | 累计分配的对象数量         | 同上                  |

命令行模式下常用命令：

```bash
(pprof) top 20               # 占用最多的前 20 个函数
(pprof) top -cum             # 按累计值排序
(pprof) list funcName        # 查看函数逐行的内存分配
(pprof) traces               # 打印调用栈
```

**两个时间点对比**

单次快照很难分辨“正常占用”和“泄漏”。**对比两个时间点的 profile 或 goroutine 差异**，是定位泄漏最有效的手段。

```bash
# 时间点 A
curl -s http://127.0.0.1:6060/debug/pprof/heap > heap_a.pb.gz
# 等待一段时间，让泄漏累积（例如 30 分钟）
curl -s http://127.0.0.1:6060/debug/pprof/heap > heap_b.pb.gz

# 以 A 为基准，查看 B 相对 A 的增量
go tool pprof -http=:8080 -base heap_a.pb.gz heap_b.pb.gz
```

可以给 heap 端点加 `?gc=1` 参数，在采样前先触发一次 GC，使 `inuse` 数据更准确。

## 3.3 测试阶段预防

Uber 开源的 [goleak](https://github.com/uber-go/goleak) 可以在单元测试结束时检查是否有遗留的 goroutine，把泄漏扼杀在开发阶段。

```go
import "go.uber.org/goleak"

// 对整个包生效
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// 或针对单个测试
func TestQuery(t *testing.T) {
    defer goleak.VerifyNone(t)
    queryWithTimeout()
}
```

若存在泄漏，测试会失败并打印残留 goroutine 的栈：

```
found unexpected goroutines:
[Goroutine 21 in state chan send, with main.queryWithTimeout.func1 on top of the stack: ...]
```

## 3.4 堆外内存排查

当 heap profile 显示的内存远小于 RSS 时：

1. 比较 `runtime/metrics` 中 Go 运行时管理的总内存（`/memory/classes/total:bytes`）与 RSS；
2. 检查 goroutine 数量和栈内存（`/memory/classes/heap/stacks:bytes`）；
3. 使用 `pmap -x <pid>` 或 `/proc/<pid>/smaps` 查看内存映射分布；
4. 若怀疑 cgo 泄漏，可以使用 `valgrind`、`jemalloc` 的 profiling 功能（`MALLOC_CONF=prof:true`）或 `bcc` 工具集中的 `memleak` 跟踪 malloc/free。

# 4. 防范清单

**Goroutine**

-  每个 `go` 语句都要明确：它**何时退出、由谁让它退出**？
-  长期运行的 goroutine 接收 `context.Context` 并监听 `ctx.Done()`
-  超时模式中的结果 channel 使用缓冲（容量 1）
-  `context.WithCancel/WithTimeout` 后立刻 `defer cancel()`
-  使用 `errgroup` 等结构化并发工具管理 goroutine 生命周期

**资源**

-  `resp.Body`、`rows`、文件、`Ticker` 获取后立即 `defer Close()/Stop()`
-  HTTP Client 复用并设置超时
-  不在循环体中 defer

**数据结构**

-  从大切片/大字符串中截取小片段长期保存时，使用拷贝（`slices.Clone`/`strings.Clone`）
-  删除指针切片元素时置 nil
-  缓存必须有容量上限和淘汰策略
-  注意 map 不缩容的特性

**工具链**

-  开启 `go vet`（检查 `lostcancel` 等）、`golangci-lint`（`bodyclose`、`sqlclosecheck`、`contextcheck` 等）
-  测试中接入 `goleak`
-  生产环境暴露 pprof（注意访问控制），监控 goroutine 数与 heap 指标
-  容器环境设置 `GOMEMLIMIT`
-  有条件时接入持续剖析平台

