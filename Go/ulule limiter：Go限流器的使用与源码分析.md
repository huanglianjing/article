# 1. 简介

github仓库地址：https://github.com/ulule/limiter/

文档地址：https://pkg.go.dev/github.com/ulule/limiter/v3

ulule/limiter 是一个 Go 语言实现的限流库，支持基于内存和基于 Redis 存储的限流。适用于服务端进行基于 Redis 的分布式限流，也支持将限流器注册到服务中间件中。

# 2. 使用

安装：

```bash
go get github.com/ulule/limiter/v3
```

创建 limiter.Rate 对象，定义周期和次数，可以直接创建对象并给成员赋值，也可以通过格式串创建对象。

```go
// 直接创建对象
// 定义周期和次数
rate := limiter.Rate{
    Period: 1 * time.Minute,
    Limit:  1000,
}

// 从格式串创建对象
// 格式串的格式为 <limit>-<period>
// limit 为次数
// period 为周期，S、M、H、D 分别表示秒、分钟、小时、天
rate, err := limiter.NewRateFromFormatted("1000-M")
```

创建 Store 对象，保存频次信息，支持基于内存和基于 Redis 两种存储。

```go
// 基于内存
import "github.com/ulule/limiter/v3/drivers/store/memory"

store := memory.NewStore()

// 基于 Redis
import "github.com/redis/go-redis/v9"
import redisLimiter "github.com/ulule/limiter/v3/drivers/store/redis"

rdb := redis.NewClient(&redis.Options{
	Addr:     "localhost:6379",
	Password: "", // no password set
	DB:       0,  // use default DB
})
store, _ := redisLimiter.NewStore(rdb)
```

创建 limiter.Limiter 对象。

```go
instance := limiter.New(store, rate)

// 传入配置
instance := limiter.New(store, rate, limiter.WithClientIPHeader("True-Client-IP"), limiter.WithIPv6Mask(mask))
```

获取频次 token 分为同步和异步两种方式，同步会等待至成功获取返回，异步则直接返回是否获取成功。

```go
// SyncGet 同步获取
func SyncGet(ctx context.Context, l *limiter.Limiter, key string) (limiter.Context, error) {
	for {
		c, err := l.Get(ctx, key)
		if err != nil {
			return limiter.Context{}, err
		}
		if c.Reached { // 达到频控上限
			resetTime := time.Unix(c.Reset, 0)
			duration := resetTime.Sub(time.Now())
			if duration > 0 {
				time.Sleep(duration)
			}
			continue
		}
		return c, nil
	}
}

// AsyncGet 异步获取
func AsyncGet(ctx context.Context, l *limiter.Limiter, key string) (limiter.Context, error) {
	return l.Get(ctx, key)
}
```

# 3. 源码分析

## 3.1 结构体和方法

limiter.Rate 结构体记录了设定的周期和次数。

```go
// rate.go
// Rate 频率
type Rate struct {
	Formatted string
	Period    time.Duration
	Limit     int64
}

// NewRateFromFormatted 从格式串创建对象
func NewRateFromFormatted(formatted string) (Rate, error) {
	rate := Rate{}

	values := strings.Split(formatted, "-")
	if len(values) != 2 {
		return rate, errors.Errorf("incorrect format '%s'", formatted)
	}

	periods := map[string]time.Duration{
		"S": time.Second,    // Second
		"M": time.Minute,    // Minute
		"H": time.Hour,      // Hour
		"D": time.Hour * 24, // Day
	}

	limit, period := values[0], strings.ToUpper(values[1])

	p, ok := periods[period]
	if !ok {
		return rate, errors.Errorf("incorrect period '%s'", period)
	}

	l, err := strconv.ParseInt(limit, 10, 64)
	if err != nil {
		return rate, errors.Errorf("incorrect limit '%s'", limit)
	}

	rate = Rate{
		Formatted: formatted,
		Period:    p,
		Limit:     l,
	}

	return rate, nil
}
```

Store 接口如下，包含了 4 个接口。

```go
// store.go
// Store 储存频次信息接口
type Store interface {
	// Peek 获取某个 key 的频次信息
	Peek(ctx context.Context, key string, rate Rate) (Context, error)
	// Get 对某个 key 频次加 1，返回更新的频次信息
	Get(ctx context.Context, key string, rate Rate) (Context, error)
	// Increment 对某个 key 频次加 count，返回更新的频次信息
	Increment(ctx context.Context, key string, count int64, rate Rate) (Context, error)
	// Reset 设置某个 key 频次为 0
	Reset(ctx context.Context, key string, rate Rate) (Context, error)
}
```

ulule/limiter 库实现了基于内存的 memory.Store 和基于 Redis 的 redis.Store，通过选择创建对象调用的函数来区分使用哪个实现。

```go
// drivers/store/memory/store.go
// NewStore 创建基于内存的 Store 对象
func NewStore() limiter.Store {
	return NewStoreWithOptions(limiter.StoreOptions{
		Prefix:          limiter.DefaultPrefix,
		CleanUpInterval: limiter.DefaultCleanUpInterval,
	})
}

// drivers/store/redis/store.go
// NewStore 创建基于 Redis 的 Store 对象
func NewStore(client Client) (limiter.Store, error) {
	return NewStoreWithOptions(client, limiter.StoreOptions{
		Prefix:          limiter.DefaultPrefix,
		CleanUpInterval: limiter.DefaultCleanUpInterval,
		MaxRetry:        limiter.DefaultMaxRetry,
	})
}
```

limiter.Limiter 结构体与方法：

```go
// limiter.go
// Limiter 限流对象
type Limiter struct {
	Store   Store   // 储存频次信息，基于内存和基于 Redis 存储分别实现了它们
	Rate    Rate    // 周期和次数
	Options Options // 配置
}

// Context 当前频次信息
type Context struct {
	Limit     int64
	Remaining int64
	Reset     int64
	Reached   bool
}

// Peek 获取某个 key 的频次信息
func (limiter *Limiter) Peek(ctx context.Context, key string) (Context, error)

// Get 对某个 key 频次加 1，返回更新的频次信息
func (limiter *Limiter) Get(ctx context.Context, key string) (Context, error)

// Increment 对某个 key 频次加 count，返回更新的频次信息
func (limiter *Limiter) Increment(ctx context.Context, key string, count int64) (Context, error)

// Reset 设置某个 key 频次为 0
func (limiter *Limiter) Reset(ctx context.Context, key string) (Context, error)
```

## 3.2 基于内存

基于内存的 Store 在内存中创建缓存，主要是封装了一个 CacheWrapper 结构体来保存限流信息。

```go
// drivers/store/memory/store.go
// Store
type Store struct {
	Prefix string       // key 前缀
	cache *CacheWrapper // 内存缓存对象
}

// Peek 获取某个 key 的频次信息
func (store *Store) Peek(ctx context.Context, key string, rate limiter.Rate) (limiter.Context, error) {
	count, expiration := store.cache.Get(store.getCacheKey(key), rate.Period)

	lctx := common.GetContextFromState(time.Now(), rate, expiration, count)
	return lctx, nil
}

// Get 对某个 key 频次加 1，返回更新的频次信息
func (store *Store) Get(ctx context.Context, key string, rate limiter.Rate) (limiter.Context, error) {
	count, expiration := store.cache.Increment(store.getCacheKey(key), 1, rate.Period)

	lctx := common.GetContextFromState(time.Now(), rate, expiration, count)
	return lctx, nil
}

// Increment 对某个 key 频次加 count，返回更新的频次信息
func (store *Store) Increment(ctx context.Context, key string, count int64, rate limiter.Rate) (limiter.Context, error) {
	newCount, expiration := store.cache.Increment(store.getCacheKey(key), count, rate.Period)

	lctx := common.GetContextFromState(time.Now(), rate, expiration, newCount)
	return lctx, nil
}

// Reset 设置某个 key 频次为 0
func (store *Store) Reset(ctx context.Context, key string, rate limiter.Rate) (limiter.Context, error) {
	count, expiration := store.cache.Reset(store.getCacheKey(key), rate.Period)

	lctx := common.GetContextFromState(time.Now(), rate, expiration, count)
	return lctx, nil
}
```

CacheWrapper 结构体是一个缓存封装器，包含了 Cache 结构体，对频次记录的增加和查询主要是对 Cache 的修改。

```go
// CacheWrapper 缓存封装器
type CacheWrapper struct {
        *Cache
}

// Cache 频次计数缓存，还会定时清理过期的 key
type Cache struct {
	counters sync.Map // 计数器
	cleaner  *cleaner // 清理器
}

// cleaner 清理器
type cleaner struct {
	interval time.Duration
	stop     chan bool
}

// Run 清理器定时从计数器清理过期 key，直至收到结束信号
func (cleaner *cleaner) Run(cache *Cache) {
	ticker := time.NewTicker(cleaner.interval)
	for {
		select {
		case <-ticker.C: // 清理过期 key
			cache.Clean()
		case <-cleaner.stop: // 收到信号停止运行
			ticker.Stop()
			return
		}
	}
}
```

以下是缓存 Cache 各个方法的主要逻辑。

```go
// Get 获取当前信息
func (cache *Cache) Get(key string, duration time.Duration) (int64, time.Time) {
	expiration := time.Now().Add(duration).UnixNano()

	counter, ok := cache.Load(key) // 直接从 cache 成员读取
	if !ok {
		return 0, time.Unix(0, expiration)
	}

	value, expiration := counter.Load(expiration)
	return value, time.Unix(0, expiration)
}

// Increment 增加计数
func (cache *Cache) Increment(key string, value int64, duration time.Duration) (int64, time.Time) {
	expiration := time.Now().Add(duration).UnixNano()

	// 在 cache 中时读取
	counter, loaded := cache.Load(key)
	if loaded {
		value, expiration = counter.Increment(value, expiration) // 增加计数，带有过期时间
		return value, time.Unix(0, expiration)
	}

	// 不在 cache 中时创建
	counter, loaded = cache.LoadOrStore(key, &Counter{
		mutex:      sync.RWMutex{},
		value:      value,
		expiration: expiration,
	})
	if loaded {
		value, expiration = counter.Increment(value, expiration) // 增加计数，带有过期时间
		return value, time.Unix(0, expiration)
	}

	// 已经被创建，直接返回
	return value, time.Unix(0, expiration)
}

// Reset 清空
func (cache *Cache) Reset(key string, duration time.Duration) (int64, time.Time) {
	cache.Delete(key) // cache 删除 key

	expiration := time.Now().Add(duration).UnixNano()
	return 0, time.Unix(0, expiration)
}
```

## 3.3 基于 Redis

基于 Redis 的 Store 包括两个 lua 脚本，会在创建 Store 的时候加载到 Redis 中，然后在需要执行命令时调用 evalsha 来执行脚本。

将频次信息设置到一个数字类型的 key 中来统计频次，通过给 key 设置过期时间的方式，实现过期 key 的自动删除。

```lua
-- luaPeekScript 读取 key
local key = KEYS[1]
local v = redis.call("get", key) -- 获取 key
if v == false then
	return {0, 0}
end
local ttl = redis.call("pttl", key) -- 设置过期时间
return {tonumber(v), ttl}

-- luaIncrScript 增加 key 并返回
local key = KEYS[1]
local count = tonumber(ARGV[1])
local ttl = tonumber(ARGV[2])
local ret = redis.call("incrby", key, ARGV[1]) -- 增加 key
if ret == count then -- 第一次 incrby 创建了 key
	if ttl > 0 then
		redis.call("pexpire", key, ARGV[2]) -- 设置过期时间
	end
	return {ret, ttl}
end
ttl = redis.call("pttl", key) -- 获取过期时间
return {ret, ttl}
```

Store 结构体以及方法。

```go
// drivers/store/redis/store.go
// Store
type Store struct {
	Prefix string         // key 前缀
	MaxRetry int          // 最大重试次数，所有操作都是原子操作，所以已经不再需要
	client Client         // Redis 客户端
	luaMutex sync.RWMutex // lua 脚本读写锁
	luaLoaded uint32      // CAS 锁
	luaIncrSHA string     // lua 脚本 SHA：增加
	luaPeekSHA string     // lua 脚本 SHA：读取
}

// Peek 获取某个 key 的频次信息
func (store *Store) Peek(ctx context.Context, key string, rate limiter.Rate) (limiter.Context, error) {
	cmd := store.evalSHA(ctx, store.getLuaPeekSHA, []string{store.getCacheKey(key)}) // 执行 lua 脚本
	count, ttl, err := parseCountAndTTL(cmd)
	if err != nil {
		return limiter.Context{}, err
	}

	now := time.Now()
	expiration := now.Add(rate.Period)
	if ttl > 0 {
		expiration = now.Add(time.Duration(ttl) * time.Millisecond)
	}

	return common.GetContextFromState(now, rate, expiration, count), nil
}

// Get 对某个 key 频次加 1，返回更新的频次信息
func (store *Store) Get(ctx context.Context, key string, rate limiter.Rate) (limiter.Context, error) {
	cmd := store.evalSHA(ctx, store.getLuaIncrSHA, []string{store.getCacheKey(key)}, 1, rate.Period.Milliseconds()) // 执行 lua 脚本
	return currentContext(cmd, rate)
}

// Increment 对某个 key 频次加 count，返回更新的频次信息
func (store *Store) Increment(ctx context.Context, key string, count int64, rate limiter.Rate) (limiter.Context, error) {
	cmd := store.evalSHA(ctx, store.getLuaIncrSHA, []string{store.getCacheKey(key)}, count, rate.Period.Milliseconds()) // 执行 lua 脚本
	return currentContext(cmd, rate)
}

// Reset 设置某个 key 频次为 0
func (store *Store) Reset(ctx context.Context, key string, rate limiter.Rate) (limiter.Context, error) {
	_, err := store.client.Del(ctx, store.getCacheKey(key)).Result() // 调用 Redis Del 删除 key
	if err != nil {
		return limiter.Context{}, err
	}

	count := int64(0)
	now := time.Now()
	expiration := now.Add(rate.Period)

	return common.GetContextFromState(now, rate, expiration, count), nil
}
```

## 3.4 获取频次返回

调用 Store 接口的各个方法，如获取当前频次信息、增加频次操作，会返回一个 limiter.Context 对象，记录了频次上限、可用次数、是否到达最大限制、下次重置时间。

这些信息将被调用方用于判断频次是否成功获取，以及还要等多久才能再次尝试获取频次。

```go
// limiter.go
// Context 频次结果
type Context struct {
	Limit     int64 // 限制量
	Remaining int64 // 可用次数
	Reset     int64 // 重置次数的时间戳
	Reached   bool  // 是否到达最大限制
}

// GetContextFromState 创建 limiter.Context 对象
func GetContextFromState(now time.Time, rate limiter.Rate, expiration time.Time, count int64) limiter.Context {
	limit := rate.Limit
	remaining := int64(0)
	reached := true

	if count <= limit { // 判断可用次数、是否到达最大限制
		remaining = limit - count
		reached = false
	}

	reset := expiration.Unix()

	return limiter.Context{
		Limit:     limit,
		Remaining: remaining,
		Reset:     reset,
		Reached:   reached,
	}
}
```

