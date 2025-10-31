# 1. 简介

github仓库地址：https://github.com/google/uuid

文档地址：https://pkg.go.dev/github.com/google/uuid

uuid 是 Go 语言中用于生成和解析 UUID（Universally Unique Identifier，通用唯一标识符）的库，主要是对 RFC4122 协议的实现。UUID 通常用于在分布式环境中作为唯一标识，如用于唯一标识区分不同的 HTTP 请求、不同的任务id、区分分布式节点和事务、以及数据库主键等。

UUID 是一个 128位长的标识符，一般通过 32 个十六进制表示，被连字符分为五段。

UUID 经过多个版本的演化，每个版本有不同的算法，其标准格式如下：

```
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
    M: 表示版本号，只会是 1 2 3 4 5
    N: 只会是 8 9 a b
```

各版本的实现原理也不同：

1. V1：基于时间，通过当前时间戳、机器 MAC 地址生成，因为 MAC 地址是全球唯一的，从而间接的可以保证 UUID 全球唯一，不过它暴露了电脑的 MAC 地址和生成这个 UUID 的时间；
2. V2：DCE安全，也是通过当前时间戳、机器 MAC 地址生成，但会把时间戳的前 4 位置换为 POSIX 的 UID 或 GID，不过这个版本在 UUID 规范中没有明确指定，基本不会实现；
3. V3：基于命名空间，由用户指定一个命名空间和一个具体的字符串，然后通过 MD5 散列来生成 UUID，主要是为了向后兼容，所以很少用到；
4. V4：基于随机数，根据随机数或者伪随机数生成 UUID，这个版本是最常使用的；
5. V5:基于名字空间，和 V3 类似，不过散列函数换成了 SHA1；

# 2. 使用

使用 go get 将 uuid 包下载到 GOPATH 指定的目录下。

```bash
go get github.com/google/uuid
```

使用 V1 版本生成一个新的 UUID。

```go
u, _ := uuid.NewUUID()
```

使用 V4 版本生成一个新的 UUID。

我们将变量 u 打印出来可以看到，它的内容包含 128 位二进制位。

```go
u := uuid.New()
// uuid.UUID{0x74, 0xcc, 0x35, 0x95, 0xde, 0xfe, 0x45, 0x7c, 0xb4, 0x4a, 0xa3, 0xf6, 0x86, 0xce, 0x28, 0xb1}
```

转化为字符串。

它的内容就是 UUID 的十六进制表示，中间多了四个减号分隔。

```go
u.String()
// 74cc3595-defe-457c-b44a-a3f686ce28b1
```

将字符串形式的 UUID 解析为 UUID 对象。

```go
u, err := uuid.Parse("74cc3595-defe-457c-b44a-a3f686ce28b1")
// uuid.UUID{0x74, 0xcc, 0x35, 0x95, 0xde, 0xfe, 0x45, 0x7c, 0xb4, 0x4a, 0xa3, 0xf6, 0x86, 0xce, 0x28, 0xb1}
```

# 3. 参考

* [UUID简介及Golang实现 - Go语言中文网 - Golang中文社区](https://studygolang.com/articles/28126)

