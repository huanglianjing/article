# 1. 架构

Redis Sentinel（哨兵）是 Redis 官方提供的高可用性解决方案，主要用于监控、管理、主动恢复 Redis 主从集群，确保服务在节点故障时仍能持续运行。

**主从复制**

Redis 的主从复制模式可以将主节点的数据同步给从节点。当主节点出现故障不可访问时，从节点作为后备，保证数据尽量不丢失。从节点还可以扩展主节点的读性能，分担主节点在大并发量时的读压力。

但这个模式也带来了一些问题：

* 一旦主节点出现故障，需要手动做很多操作：将一个从节点提升为主节点，修改客户端的主节点地址，配置其他从节点的主节点；
* 主节点的写能力受到单机的限制；
* 主节点的存储能力收到单机的限制；

**Sentinel 架构**

当主节点出现故障时，Redis Sentinel 能自动完成故障发现和故障转移，并通知客户端，实现高可用。

Sentinel 是一个分布式架构，包含若干个 Sentinel 节点和 Redis 数据节点，每个 Sentinel 节点会对数据节点和其余 Sentinel 节点进行监控。Sentinel 节点也是独立的 Redis 节点，但它不存储数据，只支持部分命令。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/database/redis_sentinel_arch.jpg)

当 Sentinel 节点发现某个数据节点不可达时，对其做下线标识。当发现主节点不可达时，会和其他 Sentinel 节点进行协商，当它们大多数认为不可达时，推选一个 Sentinel 节点来完成故障转移工作，选出新的主节点，让其他从节点指向新的主节点，然后将变动通知给客户端。

Sentinel 节点之间使用 gossip 协议交换节点状态，通过 Raft 算法来选择领导者执行故障转移。

**不足**

Sentinel 无法保证故障转移时的零数据丢失，因为主从同步可能存在延迟。

Sentinel 的监控和通信会带来额外的开销，需评估可用性和性能的平衡。

# 2. 部署

Sentinel 节点的默认端口是 26379。

建议部署至少为 3 的奇数个 Sentinel 节点，避免脑裂问题。生产环境的各个 Sentinel 应该分布在不同的物理机上，下面例子使用不同端口号仅用于测试。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/database/redis_sentinel_deploy.jpg)

**主节点**

首先启动一个主节点。

配置文件：

```
# 6379.conf
port 6379
daemonize yes
logfile "6379.log"
dbfilename "dump-6379.rdb"
dir "/opt/redis/data/"
```

启动主节点：

```bash
redis-server 6379.conf
```

**从节点**

另外启动两个从节点。

配置文件：

```
# 6380.conf
port 6380
daemonize yes
logfile "6380.log"
dbfilename "dump-6380.rdb"
dir "/opt/redis/data/"
slaveof 127.0.0.1 6379

# 6381.conf
port 6381
daemonize yes
logfile "6381.log"
dbfilename "dump-6381.rdb"
dir "/opt/redis/data/"
slaveof 127.0.0.1 6379
```

启动从节点：

```bash
redis-server 6380.conf
redis-server 6381.conf
```

在主节点或从节点查看主从关系：

```bash
redis-cli 127.0.0.1 -p 6379 info replication
```

**Sentinel 节点**

启动三个 Sentinel 节点。

配置文件，三个节点的端口号不一样，其他的一样，mymaster 是主节点的别名。

```
# 26379.conf
port 26379
daemonize yes
logfile "26379.log"
dir /opt/redis/data
sentinel monitor mymaster 127.0.0.1 6379 2      # 主节点地址，判断主节点失败的节点数量
sentinel down-after-milliseconds mymaster 30000 # 主节点判定不可达的超时时间
sentinel parallel-syncs mymaster 1              # 故障转移时，复制操作的并行数
sentinel failover-timeout mymaster 180000       # 故障转移超时时间
```

启动 Sentinel 节点，有两种方式：

```bash
redis-sentinel 26379.conf
redis-server 26379.conf --sentinel
```

