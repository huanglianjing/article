# 1. 概述

`errgroup` 是 Go 官方扩展库 `golang.org/x/sync` 中的一个包，用于管理一组协同工作的 goroutine。它在 `sync.WaitGroup` 的基础上提供了以下能力：

| 能力         | 说明                                            |
| ------------ | ----------------------------------------------- |
| **等待**     | 等待所有 goroutine 执行完毕                     |
| **错误传播** | 收集第一个返回的非 nil 错误，并由 `Wait()` 返回 |
| **取消**     | 配合 `context`，任一任务出错时通知其他任务停止  |
| **并发控制** | 通过 `SetLimit` 限制同时运行的 goroutine 数量   |

典型场景有：并发请求多个下游服务、并行处理文件、扇出/扇入（fan-out/fan-in）流水线，以及任一子任务失败即整体失败的批处理。

# 2. 使用

errgroup 需要使用 Go 官方扩展库：

```bash
go get golang.org/x/sync/errgroup
```

## 2.1 API

它的 API 如下：

```go
type Group struct { /* 不导出字段 */ }

// 创建一个带派生 context 的 Group
func WithContext(ctx context.Context) (*Group, context.Context)

// 启动一个 goroutine 执行 f
func (g *Group) Go(f func() error)

// 尝试启动 goroutine，达到并发上限时立即返回 false
func (g *Group) TryGo(f func() error) bool

// 设置最大并发数，n < 0 表示不限制
func (g *Group) SetLimit(n int)

// 阻塞等待所有 goroutine 结束，返回第一个非 nil 错误
func (g *Group) Wait() error
```

**创建对象**

可以创建一个对象后直接使用，零值**没有取消能力**，某个任务出错后，其他任务仍会继续执行到结束，`Wait()` 返回第一个错误。

```go
var g errgroup.Group
```

创建一个**有取消能力**的对象。

```go
g, ctx := errgroup.WithContext(parentCtx)
```

返回的 `ctx` 会在以下两种情况中**先发生的那一种**出现时被取消：

1. 任意一个通过 `Go` 启动的函数**首次**返回非 nil 错误；
2. `Wait()` 返回。

`Wait` 返回之后，`err` 字段不会被重置，派生 ctx 也已经取消。不要复用 Group，**每批任务都应该新建一个 Group**。

**创建任务**

```go
func (g *Group) Go(f func() error)
```

- 在新 goroutine 中执行 `f`。
- 如果设置了并发上限且已经达到上限，`Go` 会**阻塞**，直到有空位。
- 只记录**第一个**非 nil 错误，之后的错误会被丢弃。

```go
func (g *Group) TryGo(f func() error) bool
```

功能与 `Go` 相同，区别在于不阻塞：当并发数已满时直接返回 `false`，`f` 不会被执行。

**控制并发**

```go
func (g *Group) SetLimit(n int)
```

- `n < 0`：不限制并发数（默认行为）。
- `n == 0`：任何 `Go` 调用都会永久阻塞，**不要这样使用**。
- `n > 0`：最多同时运行 n 个 goroutine。
- 在仍有活跃 goroutine 时调用 `SetLimit` 会 **panic**，因此 `SetLimit` 必须在调用任何 `Go` 之前，或者在所有任务结束之后调用。

**等待**

```go
func (g *Group) Wait() error
```

阻塞直到所有通过 `Go`/`TryGo` 启动的函数返回，然后返回第一个非 nil 错误；如果全部成功，则返回 `nil`。

## 2.2 示例

并发执行任务，然后在所有任务结束后获取任一 err。

```go
package main

import (
	"fmt"
	"net/http"

	"golang.org/x/sync/errgroup"
)

func main() {
	var g errgroup.Group

	urls := []string{
		"https://go.dev/",
		"https://pkg.go.dev/",
		"https://example.invalid/",
	}

	for _, url := range urls {
		// 创建任务
		g.Go(func() error {
			resp, err := http.Get(url)
			if err != nil {
				return fmt.Errorf("fetch %s: %w", url, err)
			}
			defer resp.Body.Close()
			return nil
		})
	}

	// 等待所有任务结束，获取任一 err
	if err := g.Wait(); err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Println("all fetched successfully")
}
```

如果需要收集每个任务中的结果，`Go` 的签名只返回 `error`，结果需要通过闭包写入外部变量。

如果要将它们写入 map 或追加到同一个切片，必须使用 `sync.Mutex` 保护。推荐的做法是**预分配切片并按索引写入**，每个 goroutine 写不同的下标，所以不需要加锁：

```go
func fetchAll(ctx context.Context, urls []string) ([]string, error) {
	g, ctx := errgroup.WithContext(ctx)
	results := make([]string, len(urls)) // 预分配保存任务结果的切片，每个任务对应索引下标

	for i, url := range urls {
		g.Go(func() error {
			body, err := fetch(ctx, url)
			if err != nil {
				return err
			}
			results[i] = body // 各自写入独立下标，无数据竞争
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		return nil, err
	}
	return results, nil
}
```

使用 `WithContext` 创建对象以实现快速失败，在某一个任务失败后取消其他任务。

errgroup 只负责**取消 ctx**，不会强制终止 goroutine。子任务必须主动检查 `ctx.Done()`，或者把 `ctx` 传给支持 context 的 API（如 `http.NewRequestWithContext`、`db.QueryContext`），取消才能真正生效。

```go
func process(ctx context.Context) error {
	g, ctx := errgroup.WithContext(ctx)

	g.Go(func() error {
		return errors.New("task A failed") // 先返回 err，触发 errgroup 返回的 ctx 被取消
	})

	g.Go(func() error {
		select {
		case <-time.After(5 * time.Second):
			fmt.Println("task B done")
			return nil
		case <-ctx.Done(): // 收到 ctx 被取消，该任务提前退出
			fmt.Println("task B canceled:", context.Cause(ctx))
			return ctx.Err()
		}
	})

	return g.Wait() // 返回 "task A failed"
}
```

使用 `SetLimit` 限制并发数。

```go
func processFiles(ctx context.Context, files []string) error {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8) // 最多 8 个并发

	for _, f := range files {
		if ctx.Err() != nil {
			break // 如果已有任务失败，则不再提交，中断退出
		}
		// 达到上限时，g.Go 会阻塞在这里
		g.Go(func() error {
			return processFile(ctx, f)
		})
	}
	return g.Wait()
}
```

