# 1. 数据库性能排查

查看执行情况：

```mysql
# 查看正在执行的所有线程状态信息
SHOW PROCESSLIST;

# 查看完整查询语句
SHOW FULL PROCESSLIST;

# 查询当前运行的执行时间最久的sql
SELECT * FROM information_schema.PROCESSLIST WHERE COMMAND != 'Sleep' ORDER BY time DESC LIMIT 10;

# 当前的事务
SELECT * FROM information_schema.INNODB_TRX;

# 当前的锁
SELECT * FROM information_schema.INNODB_LOCKS;

# 当前的锁等待信息
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

# 2. 复制表数据

如果表的行数不多，直接用 insert select 语句复制并插入。

```mysql
INSERT INTO TABLE2 (col1, col2, col3)
SELECT col1, col2, col3 FROM TABLE1 Where a > 100;
```

如果表比较大，可以用 mysqldump 导出表的全部或部份数据。

```bash
mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction  --set-gtid-purged=OFF <database> <table> --where="a>900" --result-file=a.sql
```

然后再通过 sql 文件导入数据库。

```mysql
SOURCE a.sql
```

也可以导出为 csv 文件。

```mysql
select * from db1.t where a>900 into outfile 'a.csv';
```

然后导入 csv 文件。

```mysql
load data infile 'a.csv' into table db2.t;
```

# 3. 删除表的所有数据

如果表的数据量不大，可以用 delete 语句直接删除。对于非常大的表，DELETE 会非常慢，因为这种方式会一行行地删除数据，并记录每行的更改到事务日志中。

```mysql
DELETE FROM user;
```

使用 TRUNCATE 更快，因为它不记录每行的删除操作，它会删除表并重新创建回来，而不是一行行删除。这将会重置表的自增计数器，而且不能回滚。

```mysql
TRUNCATE TABLE user;
```

DROP 将会删除表结构和数据，执行速度也比较快，也是不可逆操作。

```bash
DROP TABLE user;
```

TRUNCATE 和 DROP 虽然执行速度快，但是持续从磁盘上删除表的数据文件，会消耗大量的 I/O 资源。对服务器影响更小的方式是分批每次删除一些数据，直至表不存在数据。这种方式每次占用的锁范围更小时间也更短，串行化执行不会对服务器占用很多资源，也不会影响到其他客户端的工作。

```mysql
DELETE FROM user limit 100;
```

# 4. 主备同步

主库配置：

```ini
[mysqld]
server-id=1       # 主库的唯一 ID
log-bin=mysql-bin # 启用二进制日志
binlog-format=ROW # 推荐使用 ROW 格式
```

重启 MySQL 服务：

```bash
systemctl restart mysql
```

创建用于复制的用户：

```mysql
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

查看主库状态，获得主库的 File 和 Position 值。

```mysql
SHOW MASTER STATUS;
```

接着配置备库：

```ini
[mysqld]
server-id=2               # 备库的唯一 ID，必须与主库不同
relay-log=mysql-relay-bin # 启用中继日志
read-only=1               # 备库设置为只读模式（可选）
```

重启 MySQL 服务：

```bash
systemctl restart mysql
```

配置连接主库的信息：

```mysql
CHANGE MASTER TO
MASTER_HOST='主库IP',
MASTER_USER='repl',
MASTER_PASSWORD='password',
MASTER_LOG_FILE='mysql-bin.000001',  -- 主库的 File 值
MASTER_LOG_POS=123;                  -- 主库的 Position 值
```

启动备库复制：

```mysql
START SLAVE;
```

检查复制状态：

```mysql
SHOW SLAVE STATUS;
```

# 5. 主从不一致

问题排查：

```mysql
# 主库查看
SHOW PROCESSLIST;
SHOW MASTER STATUS;

# 从库查看
SHOW SLAVE STATUS;
```

对主库锁表并查看同步点文件和位置：

```mysql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

备份数据：

```bash
mysqldump -uroot -p -hlocalhost > mysql.sql
```

把备份文件传到从库。

停止从库：

```mysql
STOP SLAVE;
```

从库导入数据备份：

```mysql
SOURCE mysql.sql
```

设置从库同步，这里设置同步点文件和位置：

```mysql
CHANG MASTER TO master_host='', master_user='',master_port=, master_password='', master_log_file='', master_log_pos=;
```

重新开启同步，查看同步状态：

```mysql
START SLAVE;
SHOW SLAVE STATUS;
```

主库解除表锁定：

```mysql
UNLOCK TABLES;
```

