# 1. 安装

## 1.1 在Linux安装

安装 MySQL：

```bash
apt install mysql-server
```

启动服务：

```bash
systemctl start mysql
```

## 1.2 在macOS安装

可以通过官网下载 MySQL 安装包安装，然后在系统设置的 MySQL 中启动服务。

或者通过 brew 安装：

```bash
brew install mysql

# 安装特定版本
brew install mysql@8.0
```

启动服务：

```bash
brew services start mysql
brew services start mysql@8.0
```

## 1.3 安装目录

不同系统可以通过安装命令安装、官网下载安装包、源码编辑的方式安装 MySQL ，它们的默认安装路径如下：

```
# Linux
/var/lib/mysql

# macOS
/usr/local/mysql

# Windows
C:\Program Files\MySQL\MySQL Server 5.7
```

安装目录下文件结构：

```
├── bin // 可执行文件
│   │── mysql       // 客户端
│   │── mysqld      // 服务器程序
│   │── mysqld_safe // 服务器启动脚本，调用 mysqld 并持续监控运行状态，出错时进行重启
│   │── mysqladmin  // 管理服务器配置和状态
│   │── mysqlshow   // 显示数据库、表、索引、列信息
│   │── mysqlcheck  // 表的维护、检查、分析
│   └── mysqldump   // 数据库备份工具
├── data // 数据目录
│   │── <database>       // 数据库名
│   │   │── <table>.frm  // 表结构文件
│   │   └── <table>.ibd  // 表数据文件
│   │── ibdata1          // 系统表空间
│   │── mysqld.local.err // 错误日志
│   └── mysqld.local.pid // 服务器进程id
├── docs    // 文档和示例
├── include // 头文件
├── lib     // 库文件
├── man     // 帮助文件
└── share   // 字符集、时区文件
```

# 2. 配置

服务器、客户端启动的时候会按优先级查找一个配置文件并读取其中的配置，MySQL 的配置文件一般为 `/etc/my.cnf`、`/etc/mysql/my.cnf`、`~/.my.cnf` 之一。

配置格式如下：

```ini
[group]
key
key=value
```

配置有多个分组如 server、mysqld、mysql_safe、mysql.server、client 等，服务器、客户端、备份工具等程序分别会读取部份分组。每个分组中都有一些单纯键的形式或者键值对形式的配置值。

也可以在启动命令中以参数的方式指定某些配置：

```bash
mysqld --key=value
```

还可以通过客户端登陆后通过命令设置系统变量，通过可选的 GLOBAL 或 SESSION 来区分设置的是服务器整体操作还是某个客户端连接，不带该参数默认为 SESSION。

```mysql
SET [GLOBAL|SESSION] key = value;
```

客户端登陆后，可以查看系统变量：

```mysql
SHOW [GLOBAL|SESSION] VARIABLES [LIKE <pattern>];

show variables;
show variables like '%timeout';
```

状态变量由服务器设置，用来表示程序运行状态，无法手动设置。查看状态变量：

```mysql
SHOW [GLOBAL|SESSION] STATUS [LIKE <pattern>];

show status;
show status like '%timeout';
```

## 2.1 连接

max-connextions 表示允许的最大客户端连接数。

prompt 设置命令提示符，如 \d 为当前数据库，\h 为服务器地址，\u 为用户名。

```ini
[mysql]
prompt='MySQL [\d]> '
```

## 2.2 日志

innodb_flush_log_at_trx_commit 设置为 1 表示每次事务的 redo log 都持久化到磁盘，建议设置成 1，保证 MySQL 异常重启后数据不丢失。

sync_binlog 设置为 1 表示每次事务的 binlog 都持久化到磁盘，建议设置成 1，保证 MySQL 异常重启后 binlog 不丢失。

**慢查询**

slow_query_log 表示慢查询日志开关，默认关闭，值为 OFF 或 ON。

slow_query_log_file 设置慢查询日志路径。

long_query_time 表示慢查询的阈值，单位为秒，默认是 10 秒，支持小数。

log_output 表示慢查询输出格式，默认为文件方式，值为 FILE 或 TABLE，输出为表时对应表为 mysql.slow_log。

```ini
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
log_output = FILE
```

可以用 mysqldumpslow 汇总和分析慢查询工具。

## 2.3 事务

transaction_isolation 设置事务的隔离级别，可以设置为 READ-UNCOMMITTED、READ-COMMITTED、REPEATABLE-READ、SERIALIZABLE。

## 2.4 存储引擎

default-storage-engine 表示默认存储引擎，默认为 InnoDB。

## 2.5 数据

datadir 指定数据目录。MySQL 的数据目录一般在安装目录的 data 目录。

## 2.6 操作

如果在 MySQL 客户端中 ctrl + w 删除了一行的字符而不是一个单词，在 ~/.editrc 添加：

```ini
bind "^W" ed-delete-prev-word
```

# 3. 客户端

客户端登陆

```bash
# 输入命令后手动输入密码
mysql -h<ip> -P<port> -u<user> -p

# 在命令中带上密码，不建议使用
mysql -h<ip> -P<port> -u<user> -p<passwd>
```

参数：

* -h / --host=：IP 地址，默认使用 localhost 或 127.0.0.1；
* -P / --port=：端口号，默认使用 3306；
* -u / --user=：用户名；
* -p / --password=：密码，参数和密码内容间不能有空格，建议参数后不带内容，在登陆后输入实际密码；
* -S / --socket=：使用套接字文件登陆，不用输入 IP 和端口号，仅限于登陆本地服务器；
* -e \<SQL\>：执行指定 SQL 语句；

查看套接字文件的路径：

```mysql
show variables like 'socket';
```

执行了语句之后，返回的最后除了说明有多少行返回和耗时，有时还会带有警告的数量：

```mysql
1 row in set, 1 warning (0.01 sec)
```

可以查看这次执行语句产生的警告信息：

```mysql
show warnings;
```

**执行 SQL 文件**

通过 mysqldump 可以导出某个表的表结构和数据到一个 sql 文件，我们也可以将要执行的 SQL 语句写在一个文件中。

登陆客户端后，通过 SOURCE 命令可以执行包含 SQL 语句的脚本文件。

```mysql
SOURCE a.sql
\. a.sql
```

也可以通过客户端携带参数的方式执行该脚本。

```bash
mysql -e "SOURCE a.sql"
mysql < a.sql
```

# 4. 服务器

服务器启动

```bash
mysqld -P<port>
```

参数：

* -P / --port=：端口号，默认使用 3306；
* --socket=：使用套接字文件通信；
* --default-storage-engine=MyISAM：设置默认存储引擎，默认为 InnoDB；
* --defaults-file=：指定配置文件；
* --datadir=：指定数据目录；

# 5. 工具

mysqldump

导出数据至一个 sql 文件。

```bash
# 导出数据库
mysqldump -h$host -P$port -u$user -p ---single-transaction  <database> > a.sql

# 导出表
mysqldump -h$host -P$port -u$user -p ---single-transaction  <database> <table> > a.sql
mysqldump -h$host -P$port -u$user -p ---single-transaction  <database> <table> --where="age>=40" > a.sql
```

# 6. 服务状态

查看正在运行的线程。

```mysql
show processlist;
```

查看系统变量值。

```mysql
show variables;

# 查看指定变量或符合正则匹配的变量
show variables like 'transaction_isolation';
show variables like 'transaction_%';
```

查看指定系统变量。

```mysql
select @@transaction_isolation;
```

# 7. 用户与权限

查看所有用户：

```mysql
SELECT user,host FROM mysql.user;
```

root 用户：

```mysql
# 修改 root 用户密码
SET PASSWORD = PASSWORD(<passwd>);
```

普通用户：

```mysql
# 创建用户
CREATE USER <user>@<ip> IDENTIFIED BY <passwd>;

# 创建用户并授权
GRANT <权限> ON <数据库>.<表> TO <user>@<ip> IDENTIFIED BY <passwd>;

# 授权
GRANT <权限> ON <数据库>.<表> TO <user>@<ip>;

# 收回权限
REVOKE <权限> ON <数据库>.<表> FROM <user>@<ip>;

# 查看用户的权限
SHOW GRANTS FOR <user>@<ip>;

# 刷新权限
FLUSH PRIVILEGES;

# 删除用户
DROP USER <user>@<ip>;

# 修改密码
SET PASSWORD FOR <user>@<ip>=<passwd>;
```

