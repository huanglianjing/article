# 1. 简介

singleflight 属于 Go 官方扩展库，它的核心作用是将重复请求合并，把并发的、相同 key 的重复调用合并成一次真实调用，singleflight 会保证只有一个请求真正去执行这个操作，而其他调用者都会等待这个唯一的请求完成，然后共享这个结果。

singleflight 最典型的用途是防止缓存击穿。当某个热点数据的缓存恰好失效过期的那一刻，如果突然有大量并发请求同时访问这个数据，由于缓存没有，这些请求会瞬间同时发起，给被调用的下游造成瞬时压力。而 singleflight 可以确保大量本应发起的请求，只有一个会真正调用到下游，其他请求都等着共享它的结果，避免对下游造成的巨大压力。

# 2. 使用

singleflight 需要使用 Go 官方扩展库：

```bash
go get golang.org/x/sync/singleflight
```

它的核心 API 有三个：

```go
func (g *Group) Do(key string, fn func() (any, error)) (v any, err error, shared bool)
func (g *Group) DoChan(key string, fn func() (any, error)) <-chan Result
func (g *Group) Forget(key string)
```

## 2.1 多次请求只执行一次

模拟一个慢查询，发起 100 次调用，实际只执行了 1 次，另外 99 次的返回值都是被分享的（shared = true）。

这里搭配 sync.WaitGroup 来使用，等待 100 次调用全部结束。

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"

	"golang.org/x/sync/singleflight"
)

var (
	g        singleflight.Group
	callCnt  int64 // 统计真实执行次数
)

// 模拟一次很慢的数据库查询
func queryDB(id string) (string, error) {
	atomic.AddInt64(&callCnt, 1)
	time.Sleep(500 * time.Millisecond)
	return id, nil
}

func getUser(id string) (string, error) {
	v, err, shared := g.Do(id, func() (interface{}, error) { // 获取值、err、是否被分享值
		return queryDB(id)
	})
	if err != nil {
		return "", err
	}
	fmt.Printf("结果=%v shared=%v\n", v, shared)
	return v.(string), nil
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			getUser("1001")
		}()
	}
	wg.Wait()
	fmt.Println("queryDB 实际被调用次数:", atomic.LoadInt64(&callCnt)) // 1
}
```

## 2.2 防止缓存击穿

通过 Do 方法，将实际查询操作包起来，后续查询请求也调用 Do 方法时，如果是查询同一个 key，会等待第一个调用完成，然后共享结果。

```go
type UserService struct {
	cache map[string]string // 缓存
	mu    sync.RWMutex
	sf    singleflight.Group
}

func (s *UserService) GetUser(id string) (string, error) {
	// 1. 先查缓存
	s.mu.RLock()
	if v, ok := s.cache[id]; ok {
		s.mu.RUnlock()
		return v, nil
	}
	s.mu.RUnlock()

	// 2. 缓存未命中：用 singleflight 保证只有一个 goroutine 去打 DB
	v, err, _ := s.sf.Do(id, func() (interface{}, error) {
		data, err := queryDB(id) // 查询数据库
		if err != nil {
			return nil, err
		}
		// 3. 回填缓存
		s.mu.Lock()
		s.cache[id] = data
		s.mu.Unlock()
		return data, nil
	})
	if err != nil {
		return "", err
	}
	return v.(string), nil
}
```

## 2.3 超时控制

Do 方法是阻塞的，如果第一个请求非常慢，所有等待者会被拖死，这时可以用 DoChan 配合 select 做超时取消的逻辑。

结果从通道中获取 Result 对象，该对象包含 Val、Err、Shared 三个成员。

使用 time.Duration 来控制超时时间：

```go
func GetUserWithTimeout(g *singleflight.Group, id string, timeout time.Duration) (string, error) {
	ch := g.DoChan(id, func() (interface{}, error) { // 得到一个通道，结果从通道获取
		return queryDB(id)
	})
	select {
	case res := <-ch: // 获取 Result 对象
		if res.Err != nil {
			return "", res.Err
		}
		return res.Val.(string), nil
	case <-time.After(timeout): // 超时
		g.Forget(id) // 关键：超时后把 key 忘掉，避免后续请求继续等这个"卡死"的调用
		return "", fmt.Errorf("timeout")
	}
}
```

使用 context 来控制超时时间：

```go
func GetUserCtx(ctx context.Context, g *singleflight.Group, id string) (string, error) {
	ch := g.DoChan(id, func() (interface{}, error) { // 得到一个通道，结果从通道获取
		return queryDB(id)
	})
	select {
	case res := <-ch: // 获取 Result 对象
		if res.Err != nil {
			return "", res.Err
		}
		return res.Val.(string), nil
	case <-ctx.Done(): // 超时
		return "", ctx.Err()
	}
}
```

# 3. 注意事项

Do/DoChan 方法在调用完成返回后，会自动删除 key，Forget 方法用于提前删除，让后续请求不再等待对应 key 的调用。

多个调用者获得的返回值是共享的同一个对象，应该只读，避免修改，必要时深拷贝再修改。

错误也是共享的，第一个调用报错后，其他等待的请求也同样会报错。

不要把某个调用者的 ctx 传进 fn，它取消会让所有共享者失败。

singleflight.Group 应该是长生命周期变量，不要每次新建。

