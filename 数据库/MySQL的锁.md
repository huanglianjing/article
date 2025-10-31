# 1. 锁

MySQL 通过不同等级的锁来处理资源并发访问问题，大致可以分为全局锁、表锁、行锁三类。

InnoDB 支持表锁和行锁，而 MyISAM 只支持表锁。

# 2. 全局锁

全局锁对整个数据库实例进行加锁。

MySQL 提供一个加全局锁的命令 FTWRL，使整个库处于只读状态，此时建表和修改表结构、数据增删改语句、更新类事务的提交将被阻塞。

```mysql
FLUSH TABLES WITH READ LOCK;
```

全局锁的典型使用场景是做全库逻辑备份。对于 InnoDB 来说，可以使用 mysqldump 备份带上参数 –single-transaction，启动一个事务获得一致性视图，期间数据库可以正常更新不需要锁表，但是对于不支持事务的引擎如 MyISAM，无法通过这种方式保持备份的一致性，数据备份恢复可能会造成数据错误。

# 3. 表锁

表级别的锁有表锁、元数据锁、意向锁。

## 3.1 表锁

表锁是读写锁，可以给指定表加读锁或写锁。如果给一个表加了读锁，本线程和其他线程可以读该表，但写表操作会被阻塞。如果给一个表加了写锁，本线程能读写该表，其他线程读写该表会被阻塞。

执行命令：

```mysql
# 加读/写锁
lock tables … read/write;
lock tables t1 read, t2 write;

# 释放锁
unlock tables;
```

## 3.2 元数据锁

元数据锁（meta data lock，MDL）是读写锁，不需要显式使用，在访问一个表时会被自动加上，用于保证读写的正确性。

当对一个表做增删改查操作时会加 MDL 读锁，对表做结构变更操作时会加 MDL 写锁，避免一个线程查询更新表数据时，另一个线程修改表结构，造成表结构对不上。

修改表结构需要 MDL 写锁，对于频繁有数据查询或更新的表，可能会一直获取不到锁。做法是避免长事务，查询并 kill 掉或等到没有长事务，再修改表结构，并在修改语句中加上等待时间：

```mysql
ALTER TABLE <tbl_name> NOWAIT add column ...
ALTER TABLE <tbl_name> WAIT N add column ... 
```

查看元数据锁的加锁情况：

```mysql
select object_type,object_schema,object_name,lock_type,lock_duration from performance_schema.metadata_locks;
```

## 3.3 意向锁

意向锁（intention locks）指未来的某个时刻，事务可能要对表的某些行加共享锁或独占锁，提前声明一个意向，避免数据库为了判断能否加表锁而逐行检查行锁，从而大幅提升性能。它是 InnoDB 为了支持多粒度锁机制而引入的，即表级锁和行级锁共存。

意向共享锁（intention shared lock, IS）表示事务有意向对表中某些行加共享锁，意向独占锁（intention exclusive lock, IX）表示事务有意向对表中某些行加独占锁。

# 4. 行锁

行锁是由各个存储引擎自己实现的，并非所有存储引擎都支持行锁。

MyISAM 不支持行锁，并发控制只能使用表锁，并发性能较差。而 InnoDB 是支持行锁的。

在 InnoDB 事务中，行锁在需要的时候才加上，然后等到事务结束时才释放，这叫两阶段锁协议。因此在事务中，应当将最可能造成锁冲突、最可能影响并发度的锁尽量放在后面。

InnoDB 中的行级锁分为行锁（Record Lock）、间隙锁（Gap Lock）、临键锁（Next-Key Lock）三种。

查看意向锁和行锁的加锁情况：

```mysql
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;
```

LOCK_TYPE 的值：

* TABLE：意向锁；
* RECORD：行级锁；

LOCK_MODE 的值：

* X：临键锁；
* X, REC_NOT_GAP：行锁；
* X, GAP：间隙锁；

## 4.1 行锁

行锁（Record Lock）直接锁住索引的一条记录。

如果语句没有匹配到索引，则无法使用行锁，将会使用表锁。

**读锁和写锁**

读锁（共享锁，Shared Locks，S 锁）用于事务读取一条记录，写锁（独占锁，Exclusive Locks，X 锁）用于事务改动一条记录。

多个事务可以同时获取一条记录的多个读锁，但是不能同时获取一条记录的读锁和写锁、多个写锁。

insert、update、delete 语句会给记录加写锁：

* insert 语句通过间隙锁来保护新插入的记录，在本事务提交前不被别的事务访问到；
* update 语句根据该记录是否被更新、列的存储空间是否发生变化，来决定申请 X 锁和间隙锁，如果未修改键值且不发生存储空间变化，则只需要申请 X 锁；
* delete 语句先在 B+ 树中定位到记录位置，然后获取这条记录的 X 锁，再执行 delete mark 操作；

普通的 select 语句不会加锁，除非在语句中主动声明：

```mysql
SELECT ... LOCK IN SHARE MODE; # 读锁
SELECT ... FOR UPDATE; # 写锁
```

select for update 针对不同的过滤条件，加锁的情况：

* 主键、唯一索引的等值查询：加行锁；
* 普通索引或范围查询：加临键锁；
* 未命中索引：加表锁；

## 4.2 间隙锁和临键锁

间隙锁（Gap Lock）锁住索引记录的间隙，确保索引记录的间隙不变，它是一个前开后开区间。临键锁（Next-Key Lock）是行锁和间隙锁的组合，它是一个前开后闭区间，如行锁锁住 10，间隙锁锁住 (5,10)，它们组成了区间 (5,10]。

这两个锁只针对可重复读或以上隔离级别，它们保护了一个区间间隙不允许插入值，可以防止幻读（Phantom Read）的发生，但是会稍微降低性能和并发度。在更低的隔离级别如读提交，MySQL 会使用临时的意向锁来避免并发问题，而非生成间隙锁。

间隙锁的加锁规则：

* 加锁的基本单位是临键锁，左开右闭区间；
* 查询过程中访问到的对象才会加锁；
* 唯一索引的范围查询会上锁到不满足条件的第一个值为止；
* 唯一索引等值查询，当记录存在，临键锁会退化为行锁；
* 索引等值查询，会将距离最近的左边界和右边界作为锁定范围，如果不是唯一索引则会继续向右匹配，直到遇到第一个不满足条件的值，如果最后一个值不等于查询条件，临键锁退化为间隙锁；

间隙锁之间不是互斥的，两个事务可以同时获取一个间隙锁，可能会因为后续对间隙锁范围内插入数据而造成死锁。

## 4.3 死锁

当并发系统中不同事务出现循环资源依赖，涉及的事务都在等待别的事务释放资源，就会导致它们进入无限等待的状态，即死锁。如事务 A 持有 id=1 的行锁时请求 id=2 的行锁，同时事务 B 持有 id=2 的行锁时请求 id=1 的行锁，它们就会陷入死锁。

处理死锁策略：

* 进入等待直到超时，超时时间通过参数 innodb_lock_wait_timeout 设置，它的默认值为 50s；
* 发起死锁检测，发现死锁后主动回滚其中一个事务，使其他事务得以继续执行，通过参数 innodb_deadlock_detect=on 开启功能，它的默认值为 on；

# 5. 参考

* [06 | 全局锁和表锁 ：给表加个字段怎么有这么多阻碍？-MySQL 实战 45 讲-极客时间](https://time.geekbang.org/column/article/69862)
* [07 | 行锁功过：怎么减少行锁对性能的影响？-MySQL 实战 45 讲-极客时间](https://time.geekbang.org/column/article/70215)
* [MySQL八股-深入理解间隙锁（GAP-LOCK）_gap lock-CSDN博客](https://blog.csdn.net/weixin_46028606/article/details/144471986)

