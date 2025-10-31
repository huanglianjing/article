# 1. 简介

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/redis_logo.png)

Redis 全称是 REmote Dictionary Server，是一个基于键值对（key - value）的 NoSQL 数据库。它将所有数据存放在内存中，所以它的读写性能非常惊人。

Redis 官网：[Redis](https://redis.io/)

Redis 源码：[redis/redis](https://github.com/redis/redis)

Redis 中的值包含 string（字符串）、hash（哈希）、list（列表）、set（集合）、zset（有序集合）五种基础的数据结构，另外还支持 Bitmaps（位图）、HyperLogLog、GEO（地理信息定位）等数据结构。此外，Redis 还提供了键过期、发布订阅、事务、流水线、Lua 脚本等功能。

**性能优势**

Redis 官方宣称读写性能可以达到 10 万/秒，执行读写命令的速度非常快，主要是因为以下几点：

* 所有的数据都存放在内存中，访问速度比硬盘快很多倍
* 使用 C 语言实现，代码简介高效
* 使用单线程模型处理命令，避免了线程切换和竞态产生的消耗
* 使用 epoll 实现非阻塞 I/O 多路复用模型
* 使用全局散列表保存所有 key-value，可以快速获取指定 key，提供了多种高效的数据结构，可以根据需求选择合适的使用

**持久化和高可用**

Redis 提供了 RDB 和 AOF 两种持久化方式，将内存的数据保存到硬盘中，避免断电和重启导致的数据丢失。

Redis 提供了复制功能，实现了多个相同数据的副本，提升了可用性。

Redis 从 3.0 版本开始提供分布式实现的 Redis Cluster，提供了高可用，扩展了读写性能和容量。

**使用场景**

Redis 典型的使用场景：

* 缓存：加快数据的访问速度，降低数据库的压力，通过键过期时间和内存淘汰策略更合理地对数据进行缓存
* 排行榜系统：列表和有序集合数据结构可以用于构建排行榜系统
* 计数器：如视频播放数、电商网页浏览数的统计
* 消息队列：提供了基础的发布订阅功能和阻塞队列功能

由于 Redis 的数据是存放在内存中的，成本相较于硬盘更高，所以不适合用于存储大规模的数据。数据根据是否频繁操作，分为热数据与冷数据，Redis 更适合存储热数据而非冷数据。

**事务ACID**

事务的 ACID 特性中，Redis 支持的情况：

* 原子性：Redis 串行执行命令和 Lua 脚本，但是不支持回滚机制，运行时错误也不会中断事务，因此只是部份支持原子性；
* 一致性：事务执行前后有数据约束保持一致，支持一致性；
* 隔离性：Redis 使用单线程执行命令，天然保证事务间的隔离，支持隔离性；
* 持久性：Redis 支持持久化，但是存在宕机时短暂数据未写入 AOF 文件而丢失，支持大部分情况下的持久性；

# 2. 数据结构与命令

## 2.1 **全局命令**

### 2.1.1 服务

停止服务。

建议使用 shutdown 命令，会在关闭时优雅关闭，刷新缓冲区资源，而不是直接 kill 掉进程。

```bash
shutdown

# 关闭前生成持久化文件
shutdown save

# 关闭前不生成持久化文件
shutdown nosave
```

### 2.1.2 状态

Redis 实例状态，包含服务器、客户端、CPU、内存等信息。

```bash
info
```

### 2.1.3 键

列出键，参数支持匹配规则，* 表示任意数量字符，? 表示任意一个字符，[] 表示匹配部分字符，\ 表示转义字符。线上环境可能保存了大量的键，不要直接查看所有键。

```bash
# 符合匹配规则的键
keys <format>

# 所有键
keys *

# 列出部分键
# cursor 是游标，从 0 开始，每次会返回当前游标，到 0 表示遍历结束
# count 是每次遍历的键数量，默认值为 10
scan <cursor> [<format>] [<count>]

# 针对哈希、集合、有序集合的键扫描命令有：hscan、sscan、zscan，用法类似
```

当前键的总数量。

```bash
dbsize
```

检查键是否存在，不支持匹配规则。

```bash
exists <key>
```

删除一到多个键，返回成功删除的键数量。

```bash
del <keys>
```

重命名键。

```bash
# 如果新名字存在键，则覆盖
rename <key> <newkey>

# 如果新名字存在键，则会失败
renamenx <key> <newkey>
```

随机返回一个键。

```bash
randomkey
```

对一个键设置过期时间，过期后键会自动删除，过期时间为负会被立刻删除。

```bash
# 指定有效时长，单位为秒
expire <key> <second>

# 指定过期秒级时间戳
expireat <key> <timestamp>

# 指定有效时长，单位为毫秒
pexpire <key> <millisecond>

# 指定过期毫秒级时间戳
pexpireat <key> <timestamp>

# 清除过期时间，字符串用 set 也会去除过期时间，其他类型不会
persist <key>
```

查看键的剩余过期时间。正整数表示剩余的过期时间，-1 表示未设置过期时间，-2 表示键不存在。

```bash
# 秒级精度
ttl <key>

# 毫秒级精度
pttl <key>
```

查看键的数据结构类型，键不存在则返回 none。

```bash
type <key>
```

查看键的内部编码实现。

```bash
object encoding <key>
```

对列表、集合、有序集合进行排序、分页，还可以返回指定字段信息，指定升降序，是否按字母排序，存到指定键。

```bash
sort <key> [by <pattern>] [limit <offset> <count>] [GET <pattern>] [ASC|DESC] [ALPHA] [STORE <destination>]
```

### 2.1.4 数据库

Redis 默认配置有 16 个数据库，各个数据库之间的数据是分隔无关联的，键名称不会冲突。默认数据库为 0 号。

由于 Redis 是单线程的，多个数据库使用一个 CPU，彼此之间会相互影响，一个数据库的慢查询会影响其他数据库。因此从 Redis 3.0 开始将逐渐弱化该功能，如果需要部署多个 Redis 实例，可以用端口号来进行区分。

切换数据库。

```bash
select <dbindex>
```

清除数据库。

```bash
# 清除当前数据库
flushdb

# 清除所有数据库
flushall
```

### 2.1.5 配置

Redis 从配置文件启动客户端后，可以通过命令获取配置以及修改配置。

获取配置。

```bash
config get <config>
```

修改配置。

```bash
config set <config> <value>
```

将配置改动持久化到配置文件。

```bash
config rewrite
```

## 2.2 基础数据结构

Redis 中有五种基础的数据结构类型，分别是 string（字符串）、hash（哈希）、list（列表）、set（集合）、zset（有序集合）。

每种数据结构都有多种内部编码实现，Redis 根据实际的数据自动选择使用哪种内部编码实现。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/redis_type_encoding.png)

## 2.3 字符串

字符串（string）类型的值实际可以是字符串、数字、或是二进制的图片视频，最大长度为 512 MB。

字符串的内部编码有 3 种：

* int：8 字节的长整形
* embstr：小于等于 39 字节的字符串
* raw：大于 39 字节的字符串

**命令**

设置值。

可选选项：

* ex seconds：设置秒级过期时间
* px milliseconds：设置毫秒级过期时间
* nx：键不存在时才设置
* xx：键存在时才设置

```bash
set <key> <value> [ex seconds] [px milliseconds] [nx|xx]

# 设置键，会去除过期时间
set key value

# 设置键 10 秒过期
set key value ex 10

# 键不存在时才设置
set key value nx
```

这两个命令和 ex nx 选项等同。

```bash
setex <key> <seconds> <value>
setnx <key> <value>
```

获取值，如果键不存在，则返回 nil（空）。

```bash
get <key>

# 设置值并返回之前的值
getset <key> <value>
```

批量设置和获取值。相比多个单次设置和获取值，批量操作只有一次网络时间。

```bash
mset <key> <value> [<key> <value>...]
mget <keys>
```

自增和自减。如果键不存在则从 0 开始计算。

```bash
# 对整数自增
incr <key>
incrby <key> <increment>

# 对整数自减
decr <key>
decrby <key> <increment>

# 对浮点数自增
incrbyfloat <key> <increment>
```

字符串操作。

```bash
# 字符串尾部追加
append <key> <value>

# 字符串长度
strlen <key>

# 获取部分字符串
getrange <key> <start> <end>

# 设置指定位置的字符
setrange <key> <offset> <value>
```

**使用场景**

字符串比较常用的使用场景是用作缓存，通过将一些数据以字符串的形式保存在 Redis，可以提高读写速度，同时降低读写底层存储的数据库的压力。如分布式 Web 服务将用户的 Session 信息存储在 Redis 中集中管理。

还可以作为计数的基础工具，实现快速计数功能，以及使用 incrby 原子操作生成全局唯一 ID。

使用键过期的功能，实现限制用户操作频率的功能。如限制一个手机号每分钟接收验证码次数，限制短信接口不被频繁调用，或者网站限制同一个 IP 地址每秒访问次数。

使用整数实现限流功能，每次请求对于访问次数加一，当值超过一定阈值后，则拦截限制该请求。

通过 setnx 和 expire 实现分布式锁功能，成功创建键的请求将获得锁，同时设置锁的过期时间，主动释放锁只需要删除该键，或者等待键过期后自动删除，其他节点就可以争取获取锁了。

## 2.4 哈希

哈希（hash）类型又叫字典、关联数组，是一个键值对（field - value）的结构。

哈希的内部编码有 2 种：

* ziplist：压缩列表，当元素个数和所有的值都小于配置值（默认为 512 和 64 字节）时使用该实现，结构更紧凑，节省内存
* hashtable：哈希表，不满足使用压缩列表的条件时，读写效率会变差，使用该实现，内存占用更大但读写效率更优

**命令**

设置哈希的键值对。

```bash
hset <key> <field> <value>
hmset <key> <field> <value> [<field> <value>...]
```

获取哈希的对应 field 的 value 值。

```bash
hget <key> <field>
hmget <key> <fields>
```

删除 field，返回成功删除的个数。

```bash
hdel <key> <fields>
```

计算 field 的个数。

```bash
hlen <key>
```

判断 field 是否存在。

```bash
hexists <key> <fields>
```

获取哈希的内容所有 field。

```bash
# 获取所有 field
hkeys <key>

# 获取所有 value
hvals <key>

# 获取所有 field - value
hgetall <key>
```

对 value 自增。

```bash
# 对整数自增
hincrby <key> <field> <increment>

# 对浮点数自增
hincrbyfloat <key> <field> <increment>
```

计算 value 字符串长度。

```bash
hstrlen <key> <field>
```

**使用场景**

对于关系型数据库保存的数据，如果很多字段为空时，保存的数据较为稀疏，可以通过哈希的结构来保存必要的 field - value 内容。无法实现关系型数据库的复杂查询，但是对于简单的读写效率更优。

## 2.5 列表

列表（list）类型是一个双向链表，用于存储多个有序的字符串，列表中的每个字符串成为元素（element），列表存储元素上限为 2^32-1 个。可以对列表两段进行插入和弹出，获取指定下标或指定范围的元素。

对于一个长度为 n 的列表，它的索引下标从左到右为 0 到 n-1，也可以从右到左用 -1 到 -n 来表示。

列表的内部编码有 3 种：

* ziplist：压缩列表，当元素个数和所有的值都小于配置值（默认为 512 和 64 字节）时使用该实现，结构更紧凑，节省内存
* linkedlist：链表，不满足使用压缩列表的条件时，读写效率会变差，使用该实现，内存占用更大但读写效率更优
* quicklist：以 ziplist 为节点的 linkedlist，结合了二者的优势

**命令**

插入元素。

```bash
# 在右边插入元素
rpush <key> <values>

# 在左边插入元素
lpush <key> <values>

# 在某个元素前或后插入元素，只在从左向右查找到第一个时插入一次
linsert <key> before|after <oldvalue> <value>
```

获取元素。

```bash
# 获取指定范围元素，这里的下标范围包含 start 和 end
lrange <key> <start> <end>

# 获取指定下标元素
lindex <key> <index>
```

获取列表长度。

```bash
llen <key>
```

修改元素。

```bash
lset <key> <index> <value>
```

删除元素。

```bash
# 从左侧弹出元素
lpop <key>

# 从右侧弹出元素
rpop <key>

# 删除指定元素，匹配等于 value 的元素
# count > 0 从左向右查找并删除，最多删除 count 个
# count < 0 从右向左查找并删除，最多删除 count 个
# count = 0 删除所有匹配的元素
lrem <key> <count> <value>

# 修剪列表，只保留索引范围内的元素
ltrim <key> <start> <end>
```

阻塞等待的方式弹出一个元素。

以该方式等待元素弹出，在队列没有数据时，将会阻塞等待，进入休眠状态，一旦有数据插入，则会立刻醒过来。

timeout 指阻塞时间，单位为秒。如果列表有元素则立刻返回，如果列表为空，则会最多阻塞指定时间，当插入了数据会立刻返回。timeout = 0 表示永远阻塞直到有值返回。

```bash
# 从左侧弹出元素
blpop <keys> timeout

# 从右侧弹出元素
brpop <keys> timeout
```

**使用场景**

列表可以用作消息队列，使用 lpush 和 brpop 命令组合实现阻塞队列，使用 lpush 和 rpop 命令组合实现队列。

队列也可以用于实现栈，使用 lpush 和 lpop 命令组合实现。

## 2.6 集合

集合（set）类型用来保存多个字符串元素的集合，集合中的元素是去重的和无序的。一个集合最多可以存储 2^32-1 个元素。

集合的内部编码有 2 种：

* intset：整数集合，所有元素都是整数切元素个数小于配置值（默认 512）时使用，可以减少内存使用
* hashtable：哈希表

**命令**

添加元素，返回成功添加的元素个数。

```bash
sadd <key> <elements>
```

判断元素是否属于集合。

```bash
sismember <key> <element>
```

获取集合内容。

```bash
# 元素数量
scard <key>

# 获取所有元素
smembers <key>
```

删除元素，返回成功删除的元素个数。

```bash
srem <key> <elements>
```

随机获取元素。

```bash
# 获取元素，不删除，count 指元素数量，不传默认为 1
srandmember <key> [<count>]

# 获取元素，并从集合删除，count 指元素数量，不传默认为 1
spop <key> [<count>]
```

集合操作。

```bash
# 交集
sinter <keys>

# 并集
sunion <keys>

# 差集，第一个集合减去后面集合的重复元素
sdiff <keys>

# 将集合操作结果保存到目标集合
sinterstore <dstkey> <keys>
sunionstore <dstkey> <keys>
sdiffstore <dstkey> <keys>
```

**使用场景**

集合的典型使用场景是保存标签，对于不同用户的多个标签分别用集合来保存，方便计算用户间标签的交集、并集和差集。

## 2.7 有序集合

有序集合（zset）也是集合，用于保存一系列去重的元素，但是元素是有序的，对于每个元素要设置一个分数（score）作为排序依据，分数可以是整型或浮点数。

集合的内部编码有 2 种：

* ziplist：压缩列表，元素个数和分数较小时的实现，可以减少内存使用
* skiplist：跳跃表

**命令**

添加元素，返回成功添加的元素个数。

有序集合为了保持有序，添加元素的时间复杂度为 O(logn)，而集合的时间复杂度为 O(1)。

可选选项：

* nx：元素不存在时才设置
* xx：元素存在时才设置
* ch：命令返回元素和分数发生变化的数量
* incr：对 score 做增加

```bash
zadd <key> <score> <element> [<score> <element>...]
```

元素数量。

```bash
zcard <key>
```

获得某个元素的分数。

```bash
zscore <key> <element>
```

增加元素的分数。

```bash
zincrby <key> <increment> <element>
```

计算元素排名，排名从 0 开始算。

```bash
# 分数从低到高排名
zrank <key> <element>

# 分数从高到低排名
zrevrank <key> <element>
```

返回指定分数范围成员个数。

```bash
zcount <key> <min> <max>
```

返回指定排名范围的元素。withscores 可选，表示同时返回元素的分数。

```bash
# 排名从低到高
zrange <key> <start> <end> [withscores]

# 排名从高到低
zrevrange <key> <start> <end> [withscores]
```

返回指定分数范围的元素。可以用 limit 限制返回的下标范围。min 和 max 还支持开区间和闭区间，如 (100 和 [100，-inf 和 +inf 表示无限小和无限大。

```bash
# 分数从低到高
zrangebyscore <key> <min> <max> [withscores] [limit <offset> <count>]

# 分数从高到低
zrevrangebyscore <key> <max> <min> [withscores] [limit <offset> <count>]
```

删除元素，返回成功删除的元素个数。

```bash
# 删除指定元素
zrem <key> <elements>

# 删除指定排名的升序元素
zremrangebyrank <key> <start> <end>

# 删除指定分数范围元素
zremrangebyscore <key> <min> <max>
```

集合操作。keynumber 指明接下来有几个有序集合，weights 列出每个有序集合的权重，aggregate 表示成员计算求和、最小值或最大值。

```bash
# 交集
zinterstore <dstkey> <keynumber> <keys> [weights <weights>] [aggregate sum｜min|max]

# 交集
zunionstore <dstkey> <keynumber> <keys> [weights <weights>] [aggregate sum｜min|max]
```

**使用场景**

有序集合的典型使用场景是排行榜系统，如视频网站对每个视频按照播放数、点赞数等维度的排行榜。

## 2.8 Bitmaps

Bitmaps 是一个以二进制位为单位的数组，实际上保存在一个字符串中，根据使用到的位的下标来分配字符串的长度。数组的下标在 Bitmaps 中叫做偏移量，支持按偏移量设置某一位的值为 0 或 1。

也可以将一个字符串类型的键看作 Bitmaps，使用它的命令。

**命令**

设置值，可以设置为 0 或 1。

```bash
setbit <key> <offset> <value>
```

获取某一位的值。

```bash
getbit <key> <offset>
```

获取指定范围值为 1 的数量，可以指定查找范围。

```bash
bitcount <key> [<start> <end>]
```

位运算，将多个的运算结果保存到目标键中。op 运算可以是 and（交集）、or（并集）、not（非）、xor（异或）。

```bash
bitop <op> <dstkey> <keys>
```

返回第一个值为 0 或 1 的偏移量。bit 可以是 0 或 1，可以指定查找范围。

```bash
bitpos <key> <bit> [<start> <end>]
```

**使用场景**

Bitmaps 适用于保存大量用户的状态，相比每个用户用一个字节来保存，Bitmaps 对于每个用户只用一个字节来保存状态，只需要使用 1/8 大小的内存。

但当设置为 1 的位较少时，会有大量的内存占用浪费，这时候使用集合反而比 Bitmaps 更节省内存。

## 2.9 HyperLogLog

HyperLogLog 是一种基于概率的统计算法，使用相对少的内存空间来完成总数的统计，但是它的统计结果是不完全准确存在误差的。一个 HyperLogLog 对象需要 12K 内存占用，就可以统计 2^64 个数据，且计数的标准误差仅为 0.81%。

这个算法是由 Philippe Flajolet 提出的，因此它的命令都以 pf 为开头。

**命令**

添加元素。

```bash
pfadd <key> <elements>
```

计算元素数量，如果有多个键则求它们的并集。

```bash
pfcount <keys>
```

合并，将多个 HyperLogLog 的并集复制给一个键。

```bash
pfmerge <dstkey> <keys>
```

**使用场景**

使用 HyperLogLog 可以减少内存的占用率，但是需要满足以下几点要求：

* 只计算独立的元素总数，不需要获取单个元素
* 元素只能增加不能删减
* 容忍一定的误差率

HyperLogLog 可以用于统计页面的总访问量、视频的总播放量，容忍一定的误差率，但是可以大大减少内存消耗。

## 2.10 GEO

GEO 用于表示地理信息定位，存储地理位置的信息，用来实现附近位置的功能。

GEO 是将信息存放在 zset 中来实现它的功能的。

**命令**

增加地理位置，指定经纬度和成员。

```bash
geoadd <key> <longitude> <latitude> <member>
```

获取成员位置。

```bash
geopos <key> <members>
```

获取两个成员间距离。unit 表示单位，可以是m（米）、km（千米）、mi（英里）、ft（尺）。

```bash
geodist <key> <member> <member> [<unit>]
```

找出一个位置一定半径内的成员集合。

```bash
georadius <key> <longitude> <latitude> <unit>
georadiusbymember <key> <member> <unit>
```

删除成员，需要借助 zset 的命令来实现。

```bash
zrem <key> <member>
```

# 3. 功能

## 3.1 慢查询分析

一条命令的执行包含了发送命令、命令在队列中排队等待、命令执行、返回结果四部分，慢查询只针对命令执行统计时间。

慢查询相关的配置参数为：

* slowlog-log-slower-than：慢查询判断的执行时间阈值，单位为微秒
* slowlog-max-len：保存的慢查询日志数量，超过时会讲最旧的删除

获取慢查询日志。可以指定条数。

```bash
slowlog get [<count>]
```

获取慢查询日志数量。

```bash
slowlog len
```

清理慢查询日志。

```bash
slowlog reset
```

## 3.2 Pipeline

一条命令的执行包含了发送命令、命令在队列中排队等待、命令执行、返回结果四部分。Redis 提供了一些批量操作命令如 mget、mset 等，将多个命令合并为一次命令，但对于其他命令，每次请求都需要经历命令的发送和返回。

Pipeline（流水线）可以将一组 Redis 命令进行组装，一次过传输给 Redis，最后将他们的执行结果按顺序返回给客户端。多条命令只需要一次发送和返回，节省了网络传输时间。

原生的批量命令是原子操作，而 Pipeline 是非原子操作。

当封装进 Pipeline 的请求过多时，可能会增加网络阻塞和客户端等待时间，因此需要适当控制 Pipeline 的大小。

## 3.3 事务

在 Redis 客户端中可以通过命令来实现事务功能，实现多条命令全部执行或全部不执行。

**multi 和 exec**

multi 命令表示事务开始，然后每个命令返回结构都是 QUEUED，表示命令被添加到待执行队列中，exec 命令表示事务结束。

```bash
multi
set a 1
set b 2
exec
```

**discard**

discard 命令表示事务停止执行。

对于语法错误，整个事务都会无法执行，而对于运行时错误，将会有部分命令执行了，而部分命令执行错误。Redis 针对事务不支持回滚，需要开发者自己修复该错误。

**watch**

watch 命令可以监控一到多个键，一旦其中有一个键被修改或删除，之后的事务就不会执行，监控一直持续到 exec 命令。

事务是在 exec 后才执行的，所以 multi 命令之后可以修改 watch 监控的键。

利用 watch 实现 incr：

```go
watch mykey
val = get mykey
val = val + 1
multi
set mykey $val
exec
```

**Lua 脚本**

Redis 支持执行 Lua 语言脚本，对于 Lua 脚本的执行是原子执行的，开发人员可以通过 Lua 实现复杂的操作，定制自己的命令。整个 Lua 脚本会一次性发送到服务器，减少了网络消耗。

通过客户端执行 Lua 脚本文件。

```bash
redis-cli --eval a.lua <参数列表>
```

通过命令执行 Lua 脚本。

```bash
eval <lua脚本> <参数数量> <参数列表>
```

evalsha 命令则是将 Lua 脚本加载到 Redis 服务器，并得到该脚本的 SHA1 校验值，

在 Lua 脚本中，KEYS[1]、KEYS[2] 等表示传递的参数列表。

Lua 脚本中也可以使用 redis.call 函数实现对 Redis 的访问。

```bash
redis.call("get", "a")
```

Redis 支持将 Lua 脚本加载到服务器，会返回脚本的 SHA1 校验值，然后通过校验值来执行已加载的脚本。

```bash
# 加载脚本到服务器
script load <script>

# 启动客户端加载脚本到服务器
redis-cli script load "$(cat a.lua)"

# 执行已加载脚本
evalsha <lua脚本SHA1值> <参数数量> <参数列表>

# 判断脚本是否加载
script exists <SHA1>

# 清除已加载脚本
script flush

# 杀掉执行中的脚本
script kill
```

## 3.4 发布订阅

Redis 提供了基于发布订阅模式的消息机制，生产者客户端向指定频道（channel）发布消息，订阅该频道的每个客户端都可以收到消息。

**命令**

发布消息。

```bash
publish <channel> <message>
```

订阅消息。

```bash
subscribe <channels>

# 按照模式订阅
psubscribe <channels>
```

取消订阅。

```bash
unsubscribe <channels>

# 按照模式取消订阅
punsubscribe <channels>
```

查看活跃的频道，即至少有一个订阅者的频道，可以指定频道名称的模式。

```bash
pubsub channels [<format>]
```

查看频道订阅数。

```bash
pubsub numsub <channels>
```

查看按模式订阅数。

```bash
pubsub numpat
```

**和 Kafka 对比**

相比于 Kafka，Redis 的发布订阅模式功能比较简单，暂时还无法做到以下功能：

* 不保证数据的持久化，一旦信息被发送，则会被丢弃，除非显示配置持久化；
* 不提供内置的 ack 消息确认机制，消息被发送后假定所有订阅者已经接收到消息；
* 不支持多分区或分片功能；
* 所有消息都是广播给所有订阅者，不支持主题的匹配、消费者组；

**使用场景**

发布订阅模式通常应用于聊天室、公告牌，以及服务之间的消息传递解耦。

# 4. 通信协议

Redis 服务器和客户端之间通过 TCP 协议通信，制定了它的通信协议 RESP（REdis Serialization Protocal，Redis 序列化协议）。这种协议以明文传输，简单高效，人可以直接读明白。

发送一条命令的格式如下，CRLF 表示 "\r\n" 换行符：

```
*<参数数量> CRLF
$<参数1字节数> CRLF
<参数1> CRLF
$<参数2字节数> CRLF
<参数2> CRLF
...
$<参数n字节数> CRLF
<参数n> CRLF
```

例如以下命令：

```bash
set a hello
```

转化为 RESP 协议发送的格式为：

```
*3
$3
SET
$1
a
$5
hello
```

返回结果的格式有五种情况：

* 状态回复：第一个字节为 +
* 错误回复：第一个字节为 -
* 整数回复：第一个字节为 :
* 字符串回复：第一个字节为 $
* 多条字符串回复：第一个字节为 *

# 4. 持久化

Redis 支持 RDB 和 AOF 两种持久化机制，避免因服务器进程退出造成的数据丢失问题，下次重启时可以利用之前持久化的文件实现数据恢复。

## 4.1 RDB

RDB（Redis Database）持久化是把当前进程的整个数据集生成快照，以二进制压缩的形式，保存到硬盘中的 dump.rdb 文件。

RDB 文件将会保存在配置 dir 指定的目录下，文件名则由配置 dbfilename 指定，Redis 默认采用 LZF 算法对生成的 RDB 文件做压缩处理。

RDB 是在某个时间点的全量数据快照，适用于数据备份、全量复制的场景，且加载 RDB 文件回复数据远快于 AOF 方式。但 RDB 方式执行成本过高，没办法做到实时或秒级持久化，并且存在不同版本的 Redis 格式兼容的问题。

**手动触发**

可以通过命令手动触发 RDB 持久化。

```bash
# 阻塞服务器执行持久化，直到完成，线上环境应避免使用
save

# fork子进程进行持久化，完成后结束子进程
bgsave
```

Redis 在 fork 子进程进行持久化时，通过写时复制（Copy-On-Write，COW）技术，即使进程内存非常大，也不会阻塞父进程，使得服务能够正常运行。

**自动触发**

Redis 还支持自动触发持久化的机制，需要在配置中配置。

以下配置表示在 m 秒内数据集存在 n 次修改时，自动触发 bgsave。

```
save <m> <n>
```

服务器关闭时，如果没有开启 AOF 持久化功能，也会自动触发 bgsave。

## 4.2 AOF

AOF（Append Only File）持久化是以独立日志的方式记录每条写命令，重启时再依次重新执行命令来恢复数据，是目前主流的持久化方式。

开启 AOF 需要配置为 appendonly yes，默认是不开启的。AOF 持久化文件名由配置 appendfilename 指定。AOF 命令写入的内容是依照 RESP 协议来转化的。

AOF 的工作流程为：将所有写入命令追加到缓冲区 aof_buf，根据对应策略从缓冲区写入硬盘。

AOF 缓冲区写入硬盘策略由配置 appendfsync 决定：

* always：每条命令都会触发 fsync 写入操作
* everysec：每秒触发 fsync 写入操作，默认配置，最多会丢失一秒的数据
* no：不对 AOF 文件做 fsync 同步，同步操作由操作系统负责

随着 AOF 文件越来越大，Redis 会定期对 AOF 文件进行重写，方式是直接把进程中的数据传化为写命令，同步到 AOF 文件中。

还有一个命令可以手动触发重写 AOF 文件。

```
bgrewriteaof
```

## 4.3 混合持久化

Redis 在 4.0 版本引入了混合持久化机制，结合 RDB 和 AOF 两种方式，拥有更高的数据安全性，提升了重启恢复速度，文件大小比纯 AOF 更小，但是比纯 RDB 的性能开销稍高一些。

需要开启以下两个配置：

```
# 开启AOF
appendonly yes

# 开启混合持久化
aof-use-rdb-preamble yes
```

当触发 AOF 重写时，Redis 会 fork 一个子进程，将当前内存的数据以 RDB 格式写到新的 AOF 文件开头，作为全量快照。生成快照期间的其他写命令会记录到 AOF 重写缓冲区，然后追加到 AOF 文件的后面，作为增量快照。最后用这个新的 AOF 混合快照文件替换旧的快照文件。

重启恢复时，读取混合快照文件，先将文件开头的 RDB 部份加载，然后按顺序重放文件后面的 AOF 部份命令。

# 5. 复制

Redis 为了解决单点问题，将实例分为主节点（master）和从节点（slave），每个主节点可以有 0 到多个从节点，将数据从主节点复制到多个从节点作为副本，满足故障恢复和负载均衡的需求。数据复制是从主节点到从节点单向流动的。

当主节点的读写压力过大时，可以对其配置多个从节点，构成一主一从结构或一主多从（星形）结构。将读写分离，写命令都对主节点操作，而读命令可以分摊至各个从节点，从而降低 Redis 压力。

配置方法有：

```
# 配置文件
slaveof <masterhost> <masterport>

# 服务器启动参数
redis-server --slaveof <masterhost> <masterport>

# 执行命令
slaveof <masterhost> <masterport>
```

从节点必须配置为 slave-read-only，避免从库写入数据导致主从数据不一致。

从节点也可以断开复制，断开与主节点的复制关系后，从节点晋升为主节点，构成树状主从结构。之前同步过来的数据仍然保留，只是不会再同步新的数据过来了。

需要执行以下命令：

```bash
slaveof no one
```

也可以配置为另一个主节点的从节点，称为切主。

**原理**

从节点执行 slaveof 命令后，复制过程便开始运作，分为以下几步：

* 从节点保存主节点的地址信息；
* 从节点通过定时任务维护复制逻辑，发现新的主节点后，尝试建立一个 socket 套接字，与该主节点建立网络连接；
* 连接建立后，从节点发送 ping 请求进行通信，如果未收到主节点的 pong 回复或超市，从节点会断开连接，下次定时任务发起重连；
* 如果主节点设置了密码验证，从节点需要配置对应参数，进行权限验证；
* 通信建立后主节点把持有的全部数据发给从节点；
* 后续持续进行命令复制，保持主从数据的一致性；

配合 Redis 的主从复制功能，可以通过使用 Redis Sentinel（哨兵）或 Redis Cluster（集群），来实现 Redis 服务的高可用性。

# 6. 参考

* [《Redis开发与运维》](https://book.douban.com/subject/26971561/)
* [《Redis深度历险：核心原理与应用实践》](https://book.douban.com/subject/30386804/)

