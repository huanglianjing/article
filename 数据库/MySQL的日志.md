# 1. redo log

redo log 叫做重做日志，是 InnoDB 存储引擎独有的日志，用于记录数据库数据更新的记录，用于保证事务的持久性和恢复数据。

MySQL 使用了 WAL（write ahead loggin）技术，先写日志，再写磁盘。当有记录需要更新时，InnoDB 引擎会先把记录写到 redo log，并更新内存，此时更新就算完成了，在系统空闲的时候再将操作记录更新到磁盘里。这个技术避免了每次更新操作都写磁盘带来的很高的 IO 和查找成本。

InnoDB 的 redo log 是固定大小的，由配置决定文件数量和每个文件的大小，使用日志文件进行循环写入。当日志写满了，需要暂停任务处理日志，直至有空余空间。

InnoDB 通过 redo log 实现了 crash-safe 的能力，即使数据库发生异常重启，之前提交的记录都不会丢失。

**写入机制**

redo log 有三种状态：存在程序内存中的 redo log buffer 中，通过 write 写到磁盘的 page cache，通过 fsync 持久化到磁盘。

通过参数 innodb_flush_log_at_trx_commit 控制每次事务提交时的 redo log 写入策略：

* 0：只把 redo log 留在 redo log buffer 中；
* 1：将 redo log 持久化到磁盘；
* 2：把 redo log 写到 page cache；

# 2. binlog

binlog 叫做归档日志，是 MySQL 的 Server 层实现的，用于实现主从复制。

最开始 MySQL 里没有 InnoDB 引擎，默认是 MyISAM 引擎，并没有 crash-safe 能力， binlog 日志只能用于归档，后续 InnoDB 以插件形式引入后才通过 redo log 实现 crash-safe。

**写入机制**

事务执行时，MySQL 会先把日志写到内存中的 binlog cache，事务提交时再把 binlog cache 中该事务完整写到磁盘中的 binlog 文件，binlog cache 内存超过一定大小也会触发写入 binlog 文件。

每个线程有自己的 binlog cache，但公用同一份 binlog 文件。

写到 binlog cache 执行 write 操作，写到 binlog 文件执行 fsync 操作，它们的时机由参数 sync_binlog 控制。

* sync_binlog=0：每次提交事务都只 write 不 fsync；
* sync_binlog=1：每次提交事务都 fsync；
* sync_binlog=N：每次提交事务都 write，累积 N 歌事务后 fsync；

出现 IO 瓶颈时，可以将 sync_binlog 设置为较大的值，如 100 - 1000，但不建议设置为 0。这个方法可以提升性能，但代价是如果机器异常关闭会丢失最近 N 个事务的 binlog 日志。

## 2.1 redo log 和 binlog 对比

redo log 和 binlog 的区别：

* redo log 是 InnoDB 引擎特有的，binlog 是 Server 层实现的，所有引擎都可以使用；
* redo log 是物理日志，记录在某个数据页上做了什么修改，binlog 是逻辑日志，记录的是语句的原始逻辑，比如给 ID=2 这一行的 c 字段加 1；
* redo log 是在固定空间的循环写，binlog 可以追加写入，binlog 文件写到一定大小后切换下一个，不覆盖以前的日志；

以一个更新语句为例：

```mysql
update t set c = c+1 where ID = 2;
```

执行流程如下，浅色框在 InnoDB 内部执行，深色框在 Server 层执行器执行。

* 执行器先找引擎取 ID=2 这一行，引擎通过主键 ID 从索引找到这一行并返回，如果所在数据页在内存则直接返回，否则从磁盘读入内存后返回；
* 执行器拿到这行数据，将 c 的值加一，再调用引擎接口写入新数据。
* 引擎将新数据更新到内存中，同事讲更新记录写到 redo log，此时 redo log 处于 prepare 状态，然后告知执行器执行完成，可以提交事务；
* 执行器生成操作的 binlog，将 binlog 写入磁盘；
* 执行器调用引擎的提交事务接口，引擎将写入的 redo log 改成 commit 状态，更新完成；

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/mysql_2pc.jpg)

## 2.2 两阶段提交

将 redo log 的写入拆分为 prepare 和 commit 两个步骤，就是两阶段提交（2PC, Two-Phase Commit）。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/mysql_2pc_log.jpg)

redo log 和 binlog 都可以拆分为 write 和 fsync 两个步骤，MySQL 对这个流程做了拆分优化，使得 redo log 和 binlog 在 fsync 时（下图 3、4 步），可以使用组提交（group commit）机制，降低磁盘 IOPS。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/mysql_2pc_log_write_fsync.jpg)

当做了错误的数据操作，希望让数据库恢复到之前某时刻的数据，只要有定期做整库备份且保存了近期以来的 binlog，就可以先从备份恢复数据库，在将备份时间到指定时间期间的 binlog 取出来重放，就可以恢复到指定时刻的数据状态了。两阶段提交是为了让 redo log 和 binlog 之间的逻辑一致。

如果不用两阶段提交，而只是分别写 redo log 和 binlog，都会产生问题：

* 先写 redo log 后写 binlog，当写完 redo log 后未写完 binlog，MySQL 异常重启，此时可以用 redo log 将数据恢复回来，但 binlog 没有记录这次改动，之后备份日志时就没有这次更新。
* 先写 binlog 后写 redo log，当写完 binlog 后未写完 redo log，发生 crash 导致事务无效，之后用 binlog 恢复数据时就会多出一个事务，和原库值不同。

# 3. undo log

undo log 叫做撤销日志，用于记录事务执行过程中对数据的修改操作，以便在事务回滚或系统崩溃时，能够将数据恢复到事务开始前的状态，以保证原子性和隔离性。

在事务执行失败或显式调用 ROLLBACK 时，使用 undo log 撤销事务对数据的修改。数据库在提交事务前崩溃时，通过 undo log 恢复数据的一致状态。undo log 还为 MVCC 提供了历史版本的数据，实现了事务的隔离性。

当事务对数据进行插入、更新、删除时，MySQL 会将之前的数据值记录到 undo log 中。事务提交后 undo log 不会立即删除，而是会保留一段时间，历史版本会在没有任何事务需要访问时被清理。

# 4. 对比

| 特性     | Redo Log（重做日志）         | Undo Log（回滚日志）                         | Binlog（二进制日志）       |
| -------- | ---------------------------- | -------------------------------------------- | -------------------------- |
| 层级     | InnoDB 存储引擎层            | InnoDB 存储引擎层                            | MySQL Server 层            |
| 分层     | 物理持久化                   | 逻辑回滚                                     | 逻辑复制                   |
| 作用     | 崩溃恢复，保证事务的持久性   | 事务回滚，实现MVCC，保证事务的原子性和隔离性 | 主从复制，数据按时间点恢复 |
| 日志类型 | 物理日志（数据页的物理修改） | 逻辑日志（数据的旧版本）                     | 逻辑日志（SQL语句）        |
| 写入方式 | 固定文件大小，循环写入覆盖   | 存储在回滚段中，事务提交后异步清理           | 按序生成文件，追加写入     |

# 5. 参考

* [02 | 日志系统：一条SQL更新语句是如何执行的？-MySQL 实战 45 讲-极客时间](https://time.geekbang.org/column/article/68633)

