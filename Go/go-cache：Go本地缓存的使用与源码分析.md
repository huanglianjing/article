# 1. 简介

github仓库地址：https://github.com/patrickmn/go-cache

文档地址：https://pkg.go.dev/github.com/patrickmn/go-cache

go-cache 是一个 Go 语言实现的缓存库，用于在本地内存中保存 key/value 形式的缓存数据，适用于单机应用程序的缓存使用，支持删除、过期的功能。

go-cache 的主要优点在于，实现了线程安全的 map[string]interface{} 结构，以供多个协程安全地使用，并且缓存可以带有过期时间也可以长期存储。

go-cache 使用一个读写互斥锁来给一个KV缓存数据对象进行加锁，在大量 key 的情况下会造成锁竞争严重的情况。

# 2. 使用

安装方式：

使用 go get 将 go-cache 包下载到 GOPATH 指定的目录下。

```shell
go get github.com/patrickmn/go-cache
```

使用示例如下，引用自官方仓库 README 文件的使用示例。

```go
import (
	"time"
	
	"github.com/patrickmn/go-cache"
)

func main() {
	// 创建缓存，设置默认过期时间5分钟，每10分钟清除过期项
	c := cache.New(5*time.Minute, 10*time.Minute)

	// 设置key/value，过期时间使用缓存默认过期时间
	c.Set("foo", "bar", cache.DefaultExpiration)
	// 设置key/value，指定过期时间
	c.Set("a", "b", time.Hour)
	// 设置key/value，无过期时间
	c.Set("baz", 42, cache.NoExpiration)

	// 删除key
	c.Delete("baz")

	// 根据key获取value，并返回判断是否存在key，返回的value类型为interface{}，需要转换为对应格式
	if x, found := c.Get("foo"); found {
		foo := x.(string)
	}

	// 保存指针提升性能
	c.Set("foo", &MyStruct, cache.DefaultExpiration)
	if x, found := c.Get("foo"); found {
		foo := x.(*MyStruct)
	}
}
```

接下来对每一段调用进行说明。

在 import 引入该包后，调用 New 函数创建一个缓存对象，并返回一个缓存指针变量。这里需要传递两个参数，第一个参数是指定 key/value 的默认过期时间，后面在设置 key/value 数据的时候可以指定过期时间，可以指定为使用缓存默认过期时间，也就是这里参数设置的时间。第二个参数表示清除过期的缓存数据的时间间隔，创建缓存对象后，会新建一个协程，定期对过期的数据进行清除，这里参数指定的就是这个定时清除的时间间隔。

```go
	// 创建缓存，设置默认过期时间5分钟，每10分钟清除过期项
	c := cache.New(5*time.Minute, 10*time.Minute)
```

创建了缓存对象之后，就可以设置 key/value 数据保存到缓存对象中，以供需要的时候读取。这里的第三个参数的数据类型为 time.Duration，表示过期时间。下面三个方法调用分别对应三种用法，第一种是使用缓存对象创建时设置的默认过期时间，第二种是指定特定过期时间，第三种则无过期时间，也就是永不过期。

```go
	// 设置key/value，过期时间使用缓存默认过期时间
	c.Set("foo", "bar", cache.DefaultExpiration)
	// 设置key/value，指定过期时间
	c.Set("a", "b", time.Hour)
	// 设置key/value，无过期时间
	c.Set("baz", 42, cache.NoExpiration)
```

删除方法非常简单，传入 key 的值即可，类似于对 map 对象的 delete 操作。

```go
	// 删除key
	c.Delete("baz")
```

获取方法传入 key 的值，返回两个值，一个是 interface{} 类型的 value，一个是布尔类型的变量，用于表示是否存在 key/value，类似于对 map 对象的下标操作。当第二个返回值为 true 时，可以对第一个返回值进行类型转换，获得需要的数据。需要注意的是，类型转换有可能会失败，建议也加上能否转换成功的判断。

```go
	// 根据key获取value，并返回判断是否存在key，返回的value类型为interface{}，需要转换为对应格式
	if x, found := c.Get("foo"); found {
		foo := x.(string)
	}
```

下面进行源码分析的时候我们将会看到，对缓存对象设置进去的 key/value，数据会被保存到一个 map 中，对于 value 大小比较大的对象，将会使得缓存对象占用比较大的内存。因此，go-cache 的作者在说明文档中建议我们，可以保存对象的指针而非对象本身，小小的调整将可以减小内存占用，提高性能。

```go
	// 保存指针提升性能
	c.Set("foo", &MyStruct, cache.DefaultExpiration)
	if x, found := c.Get("foo"); found {
		foo := x.(*MyStruct)
	}
```

# 3. 开发文档

## 3.1 常量

这是 go-cache 库包含的两个常量，NoExpiration 用 -1 表示不过期，也就是永久有效，DefaultExpiration 用 0 表示使用缓存创建时设置的默认过期时间。

在设置具体 key 的过期时间时，代码将会判断传入的值，除了这两个设定好的过期时间，大于 0 的过期时间才是设置有效的过期时间。

```go
const (
	// 不过期
	NoExpiration time.Duration = -1
	// 使用创建cache对象时设置的默认过期时间
	DefaultExpiration time.Duration = 0
)
```

## 3.2 Cache

Cache 结构体表示一个缓存对象，其中存储了许多 key/value 的映射关系，并且每个 key 都可以设置相应的过期时间。

类型定义：

```go
type Cache struct {
	// contains filtered or unexported fields
}
```

下面简要筛选出比较重要常用的方法的函数说明，包含创建、获取、设置、删除等操作。

```go
// New 创建cache对象，指定默认过期时间和清理过期item的时间间隔
func New(defaultExpiration, cleanupInterval time.Duration) *Cache
// NewFrom 从map创建cache对象
func NewFrom(defaultExpiration, cleanupInterval time.Duration, items map[string]Item) *Cache

// Get 获取key对应的value
func (c Cache) Get(k string) (interface{}, bool)
func (c Cache) GetWithExpiration(k string) (interface{}, time.Time, bool)

// Set 设置key/value，如果已存在则替换
func (c Cache) Set(k string, x interface{}, d time.Duration)
// SetDefault 设置key/value，使用默认过期时间
func (c Cache) SetDefault(k string, x interface{})
// Add 添加key/value到缓存，仅当key不存在或已过期
func (c Cache) Add(k string, x interface{}, d time.Duration) error
// Replace 设置key对应的value，仅当key已存在且未过期
func (c Cache) Replace(k string, x interface{}, d time.Duration) error

// Delete 删除key
func (c Cache) Delete(k string)

// ItemCount item的数量，包含过期的key
func (c Cache) ItemCount() int
// Items 返回所有未过期的item
func (c Cache) Items() map[string]Item

// DeleteExpired 从缓存删除所有过期的key
func (c Cache) DeleteExpired()
// Flush 从缓存清楚所有key
func (c Cache) Flush()

// 将value加上n
func (c Cache) Increment(k string, n int64) error
func (c Cache) IncrementFloat64(k string, n float64) (float64, error)
func (c Cache) IncrementInt(k string, n int) (int, error)
func (c Cache) IncrementInt64(k string, n int64) (int64, error)

// 将value减去n
func (c Cache) Decrement(k string, n int64) error
func (c Cache) DecrementFloat64(k string, n float64) (float64, error)
func (c Cache) DecrementInt(k string, n int) (int, error)
func (c Cache) DecrementInt64(k string, n int64) (int64, error)
```

## 3.3 Item

Item 结构体表示具体存储的 value 以及过期时间，这两个数据打包起来，被存储在 Cache 结构体。

类型定义：

```go
type Item struct {
	Object     interface{}
	Expiration int64
}
```

Item 结构体只有一个方法，判断是否过期。

```go
// Expired 是否已过期
func (item Item) Expired() bool
```

# 4. 实现原理

## 4.1 数据结构

Item 数据结构定义包含一个 interface{} 类型成员，用于保存缓存的对象。还有一个 int64 类型的过期时间成员，用来保存纳秒级的时间戳，如果是 0 则表示永不过期。

判断 Item 是否过期，需要判断是否为永不过期，再将设置的过期时间和当前时间做比较。

```go
type Item struct {
	Object     interface{}
	Expiration int64
}

// Returns true if the item has expired.
func (item Item) Expired() bool {
	if item.Expiration == 0 {
		return false
	}
	return time.Now().UnixNano() > item.Expiration
}
```

Cache 数据结构中包含一个对外不可见的结构体 cache 的指针。cache 结构体包含默认过期时间，也就是创建对象时设置的参数，一个类型为 map[string]Item 的 key/value 映射关系表，一个读写互斥锁，防止出现多协程读写造成的数据问题，一个删除key时的回调函数，一个定期清理器，定期删除过期元素。

```go
type Cache struct {
	*cache
}

type cache struct {
	defaultExpiration time.Duration             // 默认过期时间
	items             map[string]Item           // 保存key/value
	mu                sync.RWMutex              // 读写互斥锁
	onEvicted         func(string, interface{}) // 删除key时的回调函数
	janitor           *janitor                  // 定期清理器
}
```

## 4.2 函数和方法

go-cache 针对 Cache 类型的函数和方法有很多，这里只选出 New、Get、Set、Delete 这几个较为重要的函数和方法进行详细介绍说明。

### 4.2.1 New

New 函数会先进行初始化工作，初始化各个变量成员，然后会创建了一个 janitor 协程，这个协程将进行过期 item 的定时清理工作。

```go
func New(defaultExpiration, cleanupInterval time.Duration) *Cache {
	items := make(map[string]Item)
	return newCacheWithJanitor(defaultExpiration, cleanupInterval, items)
}

func newCacheWithJanitor(de time.Duration, ci time.Duration, m map[string]Item) *Cache {
	c := newCache(de, m)
	C := &Cache{c}
	if ci > 0 {
		runJanitor(c, ci) // 执行janitor协程，定时清理过期item
		runtime.SetFinalizer(C, stopJanitor) // 当进行垃圾回收时，发送信号以停止janitor协程
	}
	return C
}

func newCache(de time.Duration, m map[string]Item) *cache {
	if de == 0 {
		de = -1
	}
	c := &cache{
		defaultExpiration: de,
		items:             m,
	}
	return c
}
```

janitor 可以理解为一个定期清理器。创建 Cache 时将会连带启动一个 janitor 协程，这个协程会定期清理过期 item，并且会监听包含停止信号的通道，以便终结协程。

```go
type janitor struct {
	Interval time.Duration // 清理时间间隔
	stop     chan bool     // 是否停止
}

func runJanitor(c *cache, ci time.Duration) {
	j := &janitor{
		Interval: ci,
		stop:     make(chan bool),
	}
	c.janitor = j
	go j.Run(c) // 新建janitor协程
}

func (j *janitor) Run(c *cache) {
	ticker := time.NewTicker(j.Interval) // 启动定时器，时间间隔为创建Cache时参数设定的清理时间间隔
	for {
		select {
		case <-ticker.C: // 定时器触发
			c.DeleteExpired() // 清理过期item
		case <-j.stop: // 接收到停止信号返回
			ticker.Stop()
			return
		}
	}
}

func stopJanitor(c *Cache) {
	c.janitor.stop <- true // 对cache对象垃圾回收时，发送信号到通道
}
```

### 4.2.2 Get

Get 方法会先给缓存加上读锁，获取对应的 value，然后判断这个 value 是否有过期时间和是否已经过期，如果未过期则返回，最后解除读锁。

```go
func (c *cache) Get(k string) (interface{}, bool) {
	c.mu.RLock() // 加读锁
	item, found := c.items[k] // 读取value
	if !found {
		c.mu.RUnlock()
		return nil, false
	}
	if item.Expiration > 0 {
		if time.Now().UnixNano() > item.Expiration {
			c.mu.RUnlock()
			return nil, false
		}
	}
	c.mu.RUnlock() // 解锁
	return item.Object, true
}
```

### 4.2.3 Set

Set 方法先给缓存加上写锁，然后设置 key/value，最后解除写锁。

```go
func (c *cache) Set(k string, x interface{}, d time.Duration) {
	var e int64
	if d == DefaultExpiration {
		d = c.defaultExpiration
	}
	if d > 0 {
		e = time.Now().Add(d).UnixNano() // 设置过期时间
	}
	c.mu.Lock() // 加写锁
	c.items[k] = Item{ // 设置key/value
		Object:     x,
		Expiration: e,
	}
	c.mu.Unlock() // 解锁
}
```

### 4.2.4 Delete

Delete 方法先给缓存加上写锁，然后从映射表中删除 key，如果设置了回调函数则执行，最后解除写锁。

```go
func (c *cache) Delete(k string) {
	c.mu.Lock() // 加写锁
	v, evicted := c.delete(k) // 删除并获得对应的value
	c.mu.Unlock() // 解锁
	if evicted {
		c.onEvicted(k, v) // 删除key时的回调函数
	}
}

func (c *cache) delete(k string) (interface{}, bool) {
	if c.onEvicted != nil {
		if v, found := c.items[k]; found {
			delete(c.items, k)
			return v.Object, true
		}
	}
	delete(c.items, k) // 删除key
	return nil, false
}
```

# 5. 参考

* [Go 每日一库之 go-cache 缓存 - 技术颜良 - 博客园](https://www.cnblogs.com/cheyunhua/p/16802281.html)
