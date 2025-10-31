# 1. 简介

github仓库地址：https://github.com/redis/go-redis

文档地址：https://pkg.go.dev/github.com/redis/go-redis/v9

go-redis 是 Go 语言的 Redis 客户端。

# 2. 使用

## 2.1 安装

使用 go get 将 go-redis 包下载到 GOPATH 指定的目录下。
```bash
go get github.com/redis/go-redis/v9
```

## 2.2 连接

通过 redis.NewClient 设置配置连接 Redis。

```go
rdb := redis.NewClient(&redis.Options{
	Addr:     "localhost:6379",
	Password: "", // no password set
	DB:       0,  // use default DB
})
```

也可以通过 Redis URI 来连接。

```go
// 指定用户名密码
url := "redis://user:password@localhost:6379/0?"
opts, err := redis.ParseURL(url)
if err != nil {
	panic(err)
}
rdb := redis.NewClient(opts)

// 不指定用户名密码
url := "redis://@localhost:6379/0?"
opts, err := redis.ParseURL(url)
if err != nil {
	panic(err)
}
rdb := redis.NewClient(opts)
```

## 2.3 执行命令

连接后返回的 *redis.Client 对象包含各种方法，对应 Redis 的不同命令，通过调用它的方法来执行 Redis 命令。

```go
// set 命令
err := rdb.Set(ctx, "key", "value", 0).Err()

// get 命令
val, err := rdb.Get(ctx, "a").Result()
```

Do 方法可以执行任意命令，包括 Redis 支持了而 go-redis 客户端仍未支持的命令。

```go
val, err := rdb.Do(ctx, "get", "key").Result()
val.(string)

// Text 方法相当于 get.Val().(string)，直接获取字符串类型结果
val, err := rdb.Do(ctx, "get", "key").Text()
```

Do 方法返回类型为 *redis.Cmd，返回变量可以转换为其它类型。

```go
s, err := cmd.Text()
flag, err := cmd.Bool()

num, err := cmd.Int()
num, err := cmd.Int64()
num, err := cmd.Uint64()
num, err := cmd.Float32()
num, err := cmd.Float64()

ss, err := cmd.StringSlice()
ns, err := cmd.Int64Slice()
ns, err := cmd.Uint64Slice()
fs, err := cmd.Float32Slice()
fs, err := cmd.Float64Slice()
bs, err := cmd.BoolSlice()
```

## 2.4 Redis功能

**集群**

go-redis 支持 Redis Cluster 客户端。

redis.ClusterClient 表示集群对象，对集群内每个 redis 节点使用 redis.Client 对象进行通信，每个 redis.Client 会拥有单独的连接池。

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
})
```

ForEachShard 方法遍历节点，ForEachMaster 方法遍历主节点，ForEachSlave 方法遍历从节点。

```go
err := rdb.ForEachShard(ctx, func(ctx context.Context, shard *redis.Client) error {
    return shard.Ping(ctx).Err()
})
```

**哨兵**

连接到哨兵模式管理的服务器。

```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "master-name",
    SentinelAddrs: []string{":9126", ":9127", ":9128"},
})
```

连接到哨兵服务器。

```go
sentinel := redis.NewSentinelClient(&redis.Options{
    Addr: ":9126",
})

// 获取哨兵管理的服务器信息
addr, err := sentinel.GetMasterAddrByName(ctx, "master-name").Result()
```

**分片**

创建一个由三个节点组成的 Ring 客户端。

```go
rdb := redis.NewRing(&redis.RingOptions{
    Addrs: map[string]string{
        // shardName => host:port
        "shard1": "localhost:7000",
        "shard2": "localhost:7001",
        "shard3": "localhost:7002",
    },
})
```

遍历节点。

```go
err := rdb.ForEachShard(ctx, func(ctx context.Context, shard *redis.Client) error {
    return shard.Ping(ctx).Err()
})
```

**管道**

通过 Pipeline 一次执行多个命令并读取返回值，需要调用 Exec 方法后获取返回值。

```go
pipe := rdb.Pipeline()

incr := pipe.Incr(ctx, "pipeline_counter")
pipe.Expire(ctx, "pipeline_counter", time.Hour)

cmds, err := pipe.Exec(ctx)
if err != nil {
	panic(err)
}
fmt.Println(incr.Val())
```

也可以使用 Pipelined 方法，它将自动调用 Exec 方法。

```go
var incr *redis.IntCmd
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	incr = pipe.Incr(ctx, "pipelined_counter")
	pipe.Expire(ctx, "pipelined_counter", time.Hour)
	return nil
})
fmt.Println(incr.Val())
```

调用多个命令时遍历结果集获取结果。

```go
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	for i := 0; i < 100; i++ {
		pipe.Get(ctx, fmt.Sprintf("key%d", i))
	}
	return nil
})
for _, cmd := range cmds {
    fmt.Println(cmd.(*redis.StringCmd).Val())
}
```

**事务**

使用 Watch 和事务管道，来实现 INCR 操作。

```go
WATCH mykey

val = GET mykey
val = val + 1

MULTI
SET mykey $val
EXEC
```

实现代码：

```go
const maxRetries = 1000

// increment 方法，使用 GET + SET + WATCH 来实现Key递增效果，类似命令 INCR
func increment(key string) error {
	// 事务函数
	txf := func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}

		n++

		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Set(ctx, key, n, 0)
			return nil
		})
		return err
	}

	for i := 0; i < maxRetries; i++ {
		err := rdb.Watch(ctx, txf, key)
		if err == nil { // Success.
			return nil
		}
		if err == redis.TxFailedErr { // 乐观锁失败
			continue
		}
		return err
	}
	return errors.New("increment reached maximum number of retries")
}
```

**发布订阅**

发布一条消息。

```go
err := rdb.Publish(ctx, "mychannel1", "payload").Err()
```

订阅一个 Channel，使用完后必须关闭它。

```go
pubsub := rdb.Subscribe(ctx, "mychannel1")
defer pubsub.Close()
```

读取消息。

```go
for {
	msg, err := pubsub.ReceiveMessage(ctx)
	if err != nil {
		panic(err)
	}
	fmt.Println(msg.Channel, msg.Payload)
}
```

或者以 Go 通道的方式读取。

```go
ch := pubsub.Channel()
for msg := range ch {
	fmt.Println(msg.Channel, msg.Payload)
}
```

## 2.5 其它

**redis.Nil**

redis.Nil 是一种特殊的错误，表示 Redis 的一种状态。如使用 get 命令获取不存在的 key 的值时，返回 redis.Nil。

在实际逻辑中判断：

```go
val, err := rdb.Get(ctx, "key").Result()
switch {
case err == redis.Nil:
	fmt.Println("key不存在")
case err != nil:
	fmt.Println("错误", err)
case val == "":
	fmt.Println("值是空字符串")
}
```

**redis.Conn**

redis.Conn 是从连接池取出来的单个连接。

除非有特殊要求，否则尽量不要使用它。使用完之后应该调用 Close 方法将其返回给 go-redis，否则连接池会永远丢失一个连接。

```go
cn := rdb.Conn(ctx)
defer cn.Close()
```

# 3. 参考

* [Golang Redis客户端](https://redis.uptrace.dev/zh/)

