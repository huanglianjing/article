# 1. 简介

在 MongoDB 早期版本，默认的存储引擎为 MMapV1，索引使用 B树，从 MongoDB 3.0 开始引入了 WiredTiger（WT）存储引擎，并在 MongoDB 3.2 开始将其作为默认的存储引擎。

功能特性：

* 使用 B+树组织数据和索引，叶子节点使用双向链表；
* 支持文档级锁，修改不同文档可并发进行；
* 支持多版本并发控制 (MVCC)；
* 使用 Buffer Pool 缓存频繁访问的数据页，采用类似 LRU 的算法淘汰最近较少访问的页；
* 持久化机制，通过预写日志（WAL）保障原子性和持久性，通过检查点（Checkpoint）将内存脏页以一致性快照方式刷入数据文件；
* 支持对集合和索引进行压缩，减少磁盘空间消耗；
* 支持多文档 ACID 事务，提供读未提交、读提交、快照隔离这几种隔离级别；

WiredTiger 默认的隔离级别为快照隔离，也是 MongoDB 对事物采用的隔离级别。

# 2. 文件结构

WiredTiger 数据目录下的文件结构大致如下：

```
├── journal
│   ├── WiredTigerLog.0000000003
│   └── WiredTigerPreplog.0000000001
├── WiredTiger
├── WiredTiger.lock
├── WiredTiger.turtle
├── WiredTiger.wt
├── collection-*.wt
└── index-*.wt
```

文件内容：

* journal：存储 write ahead log
* WiredTiger：基本配置信息
* WiredTiger.lock：存储进程ID，用于防止多个进程连接同一个 WiredTiger 数据库
* WiredTiger.turtle：存储 WiredTiger.wt 的元数据
* WiredTiger.wt：存储所有其他集合的元数据信息
* collection-*.wt：存储集合的数据
* index-*.wt：存储索引的数据

# 3. 配置参数

WiredTiger 一些常用参数：

* storage.journal.commitIntervalMs：Journal 日志刷新周期，默认为 100ms；
* storage.syncPeriodSecs：CheckPoint 刷新周期，默认为 60s；
* storage.wiredTiger.collectionConfig.blockCompressor：压缩算法，默认为 Snappy；
* storage.wiredTiger.indexConfig.prefixCompression：是否启用索引前缀压缩，默认为 true；

# 4. 压缩

WiredTiger 默认会对集合使用块压缩，对索引使用前缀压缩，压缩后写入磁盘，从磁盘中读取出来时再进行解压。对于 Journal 日志则采用 Snappy 压缩算法压缩。通过压缩可以减少磁盘 I/O 和磁盘占用。

而在 WiredTiger 内存缓存中则是未经压缩的，可以直接读写。

# 5. 缓存

## 5.1 读缓存

WiredTiger 实现数据的二级缓存，第一层是操作系统层级的页面缓存，第二层则是存储引擎的内部缓存。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/wiredtiger_read_cache.jpg)

在读取数据时：

1. 数据库发起 buffer I/O 读操作，操作系统将磁盘数据页加载到页缓存区；
2. 引擎层读取页缓存区数据，进行解压后放到内部缓存出；
3. 在内存完成匹配查询，将结果返回给应用；

如果数据已经存储在内部缓存中， MongoDB 可以直接从内存中返回数据。内部缓存的默认大小是机器内存的一半，通过参数 wiredTigerCacheSize 指定。

## 5.2 写缓冲

WiredTiger 使用 MVCC（多版本并发控制），在操作开始时，会向该操作提供数据在该时间点的快照。

当数据发生写入时，会先在内存记录这些变更，随后通过 CheckPoint 机制将变化的数据写入磁盘，完成持久化。

WiredTiger 对数据的持久化分为两块：

* CheckPouint 机制：建立 CheckPoint 时，会在内存建立所有数据的一致性快照，然后将快照覆盖的所有数据变化通过 fsync 持久化到数据文件，默认每 60s 建立一次 CheckPoint；
* Journal 日志：通过预写日志（write ahead log）机制，将每个写操作的日志写入 Journal 缓冲区，该缓冲区会频繁将日志持久化到磁盘上，默认每 100ms 执行一次持久化；

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/wiredtiger_checkpoint_journal.jpg)

当 MongoDB 发生宕机重启时，首先会恢复到上一个 CheckPoint，然后根据 Journal 日志恢复增量的变化。Journal 日志持久化时间间隔很短，极大减少了数据丢失的情况。

## 5.3 缓存页管理

WiredTiger 使用页（Page）作为数据存取的单元，页块在内存中的结构类似 B+树，区别在于叶子节点还拥有父级指针，以实现范围遍历操作。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/wiredtiger_tree.jpg)

读取数据时，先通过 B+树索引找到对应的叶子节点，然后在页内使用二分查找来查找记录。

当叶子节点产生数据写入时，该节点会被标记为脏页，脏页通过 insert 和 update 的跳表（skiplist）来保存。

在进行 CheckPoint 时，WiredTiger 会找到所有的脏页进行持久化，为了不阻塞读，采用了 copy-on-write 的方式。

