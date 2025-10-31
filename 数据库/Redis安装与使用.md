# 1. 安装

## 1.1 在Linux安装

安装 Redis：

```bash
apt install redis-server
```

后台启动 Redis：

```bash
systemctl start redis-server
```

## 1.2 在macOS安装

通过 brew 安装 Redis：

```bash
brew install redis
```

前台启动 Redis：

```bash
redis-server
```

后台启动 Redis：

```bash
brew services start redis
```

后台关闭 Redis：

```bash
brew services stop redis
```

# 2. 配置

在 Linux 中，Redis 的配置文件是 /etc/redis.conf

在 macOS 中，Redis 的配置文件是 /usr/local/etc/redis.conf

服务端启动可以有几种方式：

```bash
# 使用默认配置文件
redis-server

# 指定配置文件
redis-server ~/redis.conf

# 指定选项，其他配置使用默认或指定的配置文件
redis-server --port 6380
```

配置文件的一些基础配置：

* port：端口
* logfile：日志文件
* dir：工作目录，存放持久化文件和日志文件
* daemonize：是否以守护进程的方式启动

Redis 的默认端口是 6379。

# 3. 客户端

启动 Redis 客户端，进入命令行模式。客户端默认连接的 IP 是 127.0.0.1，默认端口是 6379。

```bash
redis-cli
```

通过参数 -h 和 -p 可以指定特定的 IP 地址和端口。以上命令等同于：

```bash
redis-cli -h 127.0.0.1 -p 6379
```

如果带有 Redis 命令，则执行该命令，返回结果，然后就会退出结束。

```bash
redis-cli <命令>

# 示例
redis-cli PING
redis-cli set a 123
```

## 3.1 参数

-r 表示重复执行多次某个命令。

```bash
# 执行 3 次 PING
redis-cli -r 3 PING
```

-i 表示每隔几秒执行一次命令。单位为秒，可以用浮点数表示毫秒。

```bash
# 每 10 秒执行一次 PING
redis-cli -i 10 PING
```

-a 在 Redis 配置了密码时使用，不用再输入 auth 命令。

-c 用于连接 Redis Cluster 节点。

-x 从标准输入读取数据作为最后一个参数。

--eval 执行 Lua 脚本。

--slave 将当前客户端模拟为从节点。

--rdb 保存 RDB 持久化文件。

--pipe 将 Redis 通信协议的数据格式发送给 Redis 执行。

--latency 检测网络延迟。

# 4. 基准测试

redis-benchmark 用于对 Redis 做基准性能测试。

常用参数：

-c 客户端并发数量，默认是 50。

-n 客户端请求总量，默认是 100000。

-r 向 Redis 插入更多随机的键。

-t 对指定命令进行基准测试。

--csv 将结果以 csv 格式输出。

# 5. 参考

* [《Redis开发与运维》](https://book.douban.com/subject/26971561/)

