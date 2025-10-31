# 1. 简介

github仓库地址：https://github.com/bwmarrin/snowflake

文档地址：https://pkg.go.dev/github.com/bwmarrin/snowflake

snowflake 是 Go 语言中雪花算法（snowflake）的实现。雪花算法是一种生成分布式全局唯一ID的算法，生成的 ID 称为 Snowflake IDs 或 snowflakes。

一个 snowflake id 有 64 bit，包含 1 bit 符号位（始终是 0）、41 bit 毫秒时间戳（最长可以记录 69 年）、10 bit 机器 id（最多 1024 台机器）、12 bit 序列号（最多每毫秒 4096 个序列号）。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_snowflake.jpg)

雪花算法生成的 ID 能满足在高并发分布式环境系统环境下的 ID 不重复，且不依赖第三方库和中间件，生成效率很快。基于时间戳能保证基本有序递增，因此也可以按时间排序。可以根据自己的实际需求和业务场景，调整各部分的位数。

它也存在一些缺点：如强依赖于机器时钟，如果机器的时钟回拨，会导致 ID 重复的问题。生成的 ID 只能递增。

# 2. 使用

使用 go get 将 snowflake 包下载到 GOPATH 指定的目录下。

```bash
go get github.com/bwmarrin/snowflake
```

根据节点 ID 创建一个 snowflake.Node 对象。

```go 
node, _ := snowflake.NewNode(nodeID)
```

生成 snowflake ID，这个 ID 实际上是一个 64 位整数 int64。

```go
id := node.Generate()

// ID
type ID int64
```

可以将 snowflake ID 转化为不同格式。

```go
// 字符串
func (f ID) String() string

// 二进制位
func (f ID) Base2() string

// base64 编码
func (f ID) Base64() string

// 毫秒时间戳
func (f ID) Time() int64

// 节点 id
func (f ID) Node() int64

// 序列号
func (f ID) Step() int64
```

