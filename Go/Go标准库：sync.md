# 1. 介绍

文档地址：https://pkg.go.dev/sync

Go 标准库 sync 用于提供基础的同步原语，如互斥锁、Once、WaitGroup，而更高层级的同步更推荐使用通道来完成。

使用 Once 来定义只会执行一次的操作：

```go
var once sync.Once
onceFunc := func() {
	fmt.Println("Only once")
}
for i := 0; i < 10; i++ {
	once.Do(onceFunc)
}
```

使用 WaitGroup 来等待多个协程的完成：

```go
var wg sync.WaitGroup
var urls = []string{
	"http://www.golang.org/",
	"http://www.google.com/",
	"http://www.example.com/",
}
for _, url := range urls {
	wg.Add(1)
	go func(url string) {
		defer wg.Done()
		http.Get(url)
	}(url)
}
wg.Wait()
```

使用 Mutex 来添加互斥锁，防止不同协程间同时修改一个变量的值：

```go
var wg sync.WaitGroup
var m sync.Mutex

func Add(a *int) {
	m.Lock()
	defer m.Unlock()
	defer wg.Done()
	*a++
}

func main() {
	var a int
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go Add(&a)
	}
	wg.Wait()
	fmt.Printf("a=%d\n", a)
}
```

# 2. 类型

## 2.1 Once

一个 Once 对象只会执行最多一次 Do 方法的函数，即使多次调用 Do 方法的参数是不同的函数。该结构体一般涌来做只做一次的初始化工作。

类型定义：

```go
type Once struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Do 执行最多一次参数函数
func (o *Once) Do(f func())
```

## 2.2 WaitGroup

WaitGroup 用于等待多个协程的完成。每创建一个协程时调用 Add 方法来添加计数，每个协程完成后调用 Done 方法减少计数，Wait 方法则会阻塞直至所有计数清零。

类型定义：

```go
type WaitGroup struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Add 添加计数
func (wg *WaitGroup) Add(delta int)

// Done 计数减1
func (wg *WaitGroup) Done()

// Wait 阻塞直至计数为0
func (wg *WaitGroup) Wait()
```

## 2.3 Mutex

互斥锁，任何时间最多只支持一个地方获得锁。不与任何协程绑定，可以在不同协程间上锁和解锁，来控制程序执行流程。

类型定义：

```go
type Mutex struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Lock 上锁，如果对象已经上锁，则阻塞至可用
func (m *Mutex) Lock()

// TryLock 尝试上锁，返回是否成功
func (m *Mutex) TryLock() bool

// Unlock 解锁，对未上锁的互斥锁解锁会报错
func (m *Mutex) Unlock()
```

## 2.4 RWMutex

读写互斥锁，支持任意数量的读锁或者一个写锁。

类型定义：

```go
type RWMutex struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Lock 上写锁，若已经有读锁或写锁则阻塞
func (rw *RWMutex) Lock()

// TryLock 尝试上写锁，返回是否成功
func (rw *RWMutex) TryLock() bool

// RLock 上读锁
func (rw *RWMutex) RLock()

// TryRLock 尝试上读锁，返回是否成功
func (rw *RWMutex) TryRLock() bool

// Unlock 解写锁
func (rw *RWMutex) Unlock()

// RUnlock 解读锁
func (rw *RWMutex) RUnlock()

// RLocker 返回一个 Locker 接口对象
func (rw *RWMutex) RLocker() Locker
```

## 2.5 Pool

Pool 是一系列临时对象的读取和保存，对于多协程同步使用是安全的，主要用于缓存申请了而未使用的临时对象，以便后续使用，减轻垃圾回收的压力。成员变量 New 是一个返回 any 类型的函数，应该返回指针类型，这样作为 interface 值返回的时候就不需要申请内存了。

类型定义：

```go
type Pool struct {
	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() any
	// contains filtered or unexported fields
}
```

方法：

```go
// Put 将变量加到池子
func (p *Pool) Put(x any)

// Get 从池子移除任意一个变量并返回，当池子空时，如果定义了 New 方法则调用方法返回一个对象，否则返回 nil
func (p *Pool) Get() any
```

## 2.6 Locker

Locker 是一个接口，表示可以被上锁和解锁的对象。

类型定义：

```go
type Locker interface {
	Lock()
	Unlock()
}
```

## 2.7 Cond

Cond 是一个可供多个协程等待和触发事件的条件变量。关联的 Locker 变量通常是一个 Mutex 或 RWMutex。Cond 能够做到控制协程让出处理器使用权、唤醒协程、广播所有等待的协程。

类型定义：

```go
type Cond struct {

	// L is held while observing or changing the condition
	L Locker
	// contains filtered or unexported fields
}
```

方法：

```go
// NewCond 创建对象
func NewCond(l Locker) *Cond

// Broadcast 唤醒所有等待变量的协程
func (c *Cond) Broadcast()

// Signal 唤醒一个等待变量的协程
func (c *Cond) Signal()

// Wait 解锁 L 并阻塞等待当前协程
func (c *Cond) Wait()
```

## 2.8 Map

协程安全的 Map。

类型定义：

```go
type Map struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// CompareAndDelete 如果 key 对应的 value 等于 old，则删除对应 key
func (m *Map) CompareAndDelete(key, old any) (deleted bool)

// CompareAndSwap 如果 key 对应的 value 等于 old，则设置为 new
func (m *Map) CompareAndSwap(key, old, new any) bool

// Store 设置 key/value
func (m *Map) Store(key, value any)

// Load 返回 key 对应的 value
func (m *Map) Load(key any) (value any, ok bool)

// Swap 将 key 对应的 value 替换为参数的，并返回之前的 value
func (m *Map) Swap(key, value any) (previous any, loaded bool)

// LoadOrStore 如果 key 存在则返回对应的 value，否则设置参数的 value 并返回
func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool)

// LoadAndDelete 删除 key 并返回对应的 value
func (m *Map) LoadAndDelete(key any) (value any, loaded bool)

// Delete 删除 key
func (m *Map) Delete(key any)

// Range 对每个 key/value 调用指定函数
func (m *Map) Range(f func(key, value any) bool)
```

# 3. 函数

```go
// OnceFunc 返回一个函数，该函数调用多少次都只会调用参数函数一次
func OnceFunc(f func()) func()

// OnceValue 返回一个返回某变量的函数，该函数调用多少次都只会调用参数函数一次
func OnceValue[T any](f func() T) func() T

// OnceValues 返回一个返回某两变量的函数，该函数调用多少次都只会调用参数函数一次
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2)
```

