# 1. 概述

SQL 包含以下指令，指令不区分大小写。

```
CREATE
DROP
ALTER
SELECT
INSERT
UPDATE
DELETE
COMMIT
ROLLBACK
GRANT
REVOKE
```

SQL 语句分为三类：

* DDL（Data Definition Language，数据定义语言）：用于创建或者删除数据库或表，包含 CREATE、DROP、ALTER
* DML（Data Manipulation Language，数据操纵语言）：用于查询或变更表中的记录，包含 SELECT、INSERT、UPDATE、DELETE
* DCL（Data Control Language，数据控制语言）：用于确认或取消对数据的变更，以及对操作权限进行设定，包含 COMMIT、ROLLBACK、GRANT、REVOKE

SQL 以分号 ; 为结尾，以 \G 为结尾。

SQL 的注释可以是这几种方式：

```sql
#comment
-- comment # --后需要加空格
/*comment*/ #可以是多行注释
```

当 SQL 语句数量较多或者较长时，可以将要执行的 SQL 语句保存在一个文件中，然后在客户端中调用：

```sql
source a.sql
```

# 2. 数据库操作

查看数据库列表：

```sql
SHOW DATABASES;
```

创建数据库：

```sql
CREATE DATABASE <数据库>;
```

查看建数据库语句：

```sql
SHOW CREATE DATABASE <数据库>;
```

选择数据库：

```sql
USE <数据库>;
```

查看数据库中的表：

```sql
SHOW TABLES;
SHOW TABLES [FROM <数据库>]; # 指定数据库
SHOW TABLES LIKE <模式>; # 根据模式查询表
```

# 3. 表操作

查看表状态：

```sql
SHOW TABLE STATUS;
```

查看建表语句：

```sql
SHOW CREATE TABLE <表>;
SHOW CREATE TABLE <表>\G # 因为只有一行返回结果，这种方式更清楚
```

查看表结构：

```sql
DESC <表>;
SHOW COLUMNS FROM <表>;
```

查看索引：

```sql
SHOW INDEX FROM <表>;
```

删除表：

```sql
DROP TABLE <表>;
DROP TABLE IF EXISTS <表>;
```

修改表名：

```sql
RENAME TABLE <表> TO <表>;
ALTER TABLE <表> RENAME TO <表>;
```

添加列：

```sql
ALTER TABLE <表> ADD COLUMN <列> <类型>;
ALTER TABLE <表> ADD COLUMN <列> <类型> AFTER <列>;
```

删除列：

```sql
ALTER TABLE <表> DROP COLUMN <列>;
```

修改列：

```sql
ALTER TABLE <表> MODIFY COLUMN <列> <调整点>;
```

根据已有表创建同样结构的表：

```sql
CREATE TABLE <表> LIKE <表>;
```

创建表：

```sql
CREATE TABLE <表>
(
  <列> <类型> [<属性>],
  <列> <类型> [<属性>],
  ....
  <索引>,
  <索引>,
  ....
) <表信息>;

# 示例
CREATE TABLE IF NOT EXISTS `user` (
  `id`          bigint(20)  NOT NULL auto_increment,
  `uid`         bigint(20)  NOT NULL            COMMENT '用户id',
  `name`        varchar(64) NOT NULL DEFAULT '' COMMENT '用户名称',
  `create_time` timestamp   NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` timestamp   NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_user` (`uid`),
  KEY `name_index` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

列要求英文字母开头，使用字母数字下划线。

数据类型包含：

* 整数：tinyint 为 1 字节，smallint 为 2 字节，mediumint 为 3 字节，int 为 4 字节，bigint 为 8 字节。int(m) 中 m 为显示最小的显示宽度，不影响字段范围。
* 浮点数：float 为单精度，4 字节，double 为双精度，8 字节，float(m,d) 中 m 为总位数，d 为小数位数。
* 定点数：decimal，decimal(m,d) 中 m 为总位数，d 为小数位数。
* 字符串：char(n) 为固定长度，n 为字符数，最多 255 个字符，后面用空格补足，varchar(n) 为可变长度，n 为字符数，最多 65535 个字符，text、mediumtext、longtext 为可变长度，最多 65535、2^24-1、2^32-1 个字符。
* 日期：date 为日期，time 为时间，datetime 为日期时间，timestamp 为时间戳。

数据属性：

* NULL：可包含 NULL 值
* NOT NULL：不允许包含 NULL 值
* PRIMARY KEY：主键
* DEFAULT <默认值>：默认值
* AUTO_INCREMENT：自增，只能用于一列，插入不必指定该列的值
* UNSIGNED：无符号整数
* CHARACTER SET <字符集>：指定字符集

索引：

* 主键：PRIMARY KEY (<列列表>)
* 外键：FOREIGN KEY (<列列表>) REFERENCES <表>(<列>) [<级联操作>]，必须对应另一张表的主键或唯一键
  * 级联操作：
  * ON DELETE CASCADE：对应行删除则本表级联删除
  * ON UPDATE CASCADE：对应行更新则本表级联更新
  * ON DELETE SET NULL：设为NULL
  * ON DELETE SET DEFAULT：设为默认值
* 唯一索引：UNIQUE KEY <索引> (<列列表>)
* 普通索引：KEY <索引> (<列列表>)

添加索引：

```sql
CREATE INDEX <索引> ON <表>(<列>);
ALTER TABLE <表> ADD INDEX <索引>(<列>);

# 唯一索引
CREATE UNIQUE INDEX <索引> ON <表>(<列>);
ALTER TABLE <表> ADD UNIQUE INDEX <索引>(<列>);
```

删除索引：

```sql
DROP INDEX <索引> ON <表>;
ALTER TABLE <表> DROP INDEX <索引>;
```

# 4. 表查询

查询表使用 SELECT 语句，通用的形式如下：

```sql
SELECT <列列表> FROM <表列表> [WHERE <条件>] [GROUP BY <列> [HAVING <条件>]] [ORDER BY <列列表>] [<限制行数>];
```

执行顺序：FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY

可以调用子查询，子查询括号后面必须带有名称：

```sql
SELECT <列列表> FROM (<SELECT>) <子查询>;
```

列可以是以下形式：

* *：所有列
* \<列\> AS \<key\>：为列或表设定别名，可被引用
* \<字符串/数字/日期常量\> AS \<key\>：常量列
* DISTINCT \<列\>：结果去重，必须写在 select 的字段中的最前面
* 运算表达式：+、-、*、/、%，含 NULL 列计算结果也是 NULL
* CASE 表达式：CASE WHEN <表达式> THEN <表达式> WHEN <表达式> THEN <表达式> ... [ELSE <表达式>] END
* 函数

函数包含：

| 函数类别   | 函数                 | 含义           |
| ---------- | -------------------- | -------------- |
| 聚合函数   | COUNT(<列>)          | 行数           |
|            | COUNT(DISTINCT <列>) | 去重行数       |
|            | SUM(<列>)            | 求和           |
|            | AVG(<列>)            | 平均值         |
|            | MAX(<列>)            | 最大值         |
|            | MIN(<列>)            | 最小值         |
| 算数函数   | ABS(i)               | 绝对值         |
|            | MOD(i,j)             | 余数           |
|            | ROUND(i,j)           | 保留小数位数   |
| 字符串函数 | CONCAT(i,j)          | 字符串拼接     |
|            | LENGTH(i)            | 长度           |
|            | LOWER(i)             | 转换小写       |
|            | UPPER(i)             | 转换大写       |
|            | REPLACE(i,j,k)       | 字符串匹配替换 |
| 日期函数   | CURRENT_DATE         | 日期           |
|            | CURRENT_TIME         | 时间           |
|            | CURRENT_TIMESTAMP    | 时间戳         |
| 转换函数   | CAST(<值> AS <类型>) | 类型转换       |

查询条件：

* 运算表达式：+ - * /
* 比较运算符：= <> != > >= < <=，字符串的比较基于字典序
* 逻辑运算符：AND OR NOT，结果为 TRUE 或 FALSE，用在含 NULL 的值上结果可能为 UNKNOWN
* <列> IS [NOT] NULL：判断是否为NULL
* <列> [NOT] LIKE '<模式>'：字符串模式匹配，% 表示 0 至多个任意字符，_ 表示一个任意字符，\ 表示转义
* <列> [NOT] BETWEEN i AND j：范围查询，包含 i 和 j，需要 i<=j
* <列> [NOT] IN (<值列表>|\<SELECT\>)：是否属于集合的一个
* [NOT] EXIST (\<SELECT\>)：子查询是否有内容
* [NOT] UNIQUE (\<SELECT\>)：子查询是否有重复行

分组：

* GROUP BY <列>：以列分组，SELECT时分组的列可以直接列出，其他列需要用聚合函数
* GROUP BY <列> HAVING <条件>：对分组后的结果进行条件筛选
* GROUP BY <列> WITH ROLLUP：添加一列统计聚合函数列的和

排序：

* ORDER BY <列列表>：按列排序，默认升序，有多列则按顺序定优先级
* ORDER BY <列> ASC：升序，可为每个列指定顺序
* ORDER BY <列> DESC：降序
* ORDER BY \<n\>：使用SELECT中的第n列排序，从1开始算

限制行数：

* LIMIT n：取结果的前 n 行
* LIMIT m,n：从结果的第 m 行开始取 n 行

表连接有如下几种：

* 内连接：把两张表所有两两符合条件的行组合，不含条件时等于交叉连接
* 左外连接：内连接加上左表那些匹配不到右表的行，右表对应列填NULL
* 右外连接：内连接加上右表那些匹配不到左表的行，左表对应列填NULL
* 交叉连接：两张表所有记录两两组合，不添加连接条件则叫笛卡尔积
* 自然连接：同名列的值相同时组合

```sql
SELECT <列列表> FROM <表1> [INNER] JOIN <表2> [ON <条件>] [WHERE <条件>]; # 内连接
SELECT <列列表> FROM <表1> LEFT [OUTER] JOIN <表2> ON <条件> [WHERE <条件>]; # 左外连接
SELECT <列列表> FROM <表1> RIGHT [OUTER] JOIN <表2> ON <条件> [WHERE <条件>]; # 右外连接
SELECT <列列表> FROM <表1> CROSS JOIN <表2> [WHERE <条件>]; # 交叉连接
SELECT <列列表> FROM <表1> NATURAL JOIN <表2> [WHERE <条件>]; # 自然连接
```

多个语句结果合并，它们的列数要相等：

```sql
<SELECT> UNION <SELECT> # 去除重复行
<SELECT> UNION ALL <SELECT> # 不去除重复行
```

通过 EXPLAIN 可以用于分析查询语句的执行计划，不会实际执行语句。

```sql
EXPLAIN <SQL>;
```

# 5. 表更新

插入行：

```sql
INSERT INTO <表> VALUES (<值列表>);
INSERT INTO <表> (<列列表>) values(<值列表>) # 指定对应列的插入值
INSERT INTO <表> <SELECT>; # 用查询的数据插入
INSERT INTO <表> (<列列表>) values(<值列表>) ON DUPLICATE KEY UPDATE col1=v1, col2=v2; # 主键或唯一键冲突时更新指定列
```

更新行，该操作十分危险，要记得核对查询条件：

```sql
UPDATE <表> SET <列> = <表达式> [WHERE <条件>];
UPDATE <表> SET <列> = <表达式>, <列> = <表达式> [WHERE <条件>]; # 更新多列
```

删除行，该操作十分危险，要记得核对查询条件：

```sql
DELETE FROM <表> [WHERE <条件>];
```

全表删除：

```sql
# 删除数据保留空表，一行行删除，可回滚
DELETE FROM <表>;

# 删除数据保留空表，将表数据文件大小重置为初始大小，速度快，不记录log
TRUNCATE TABLE <表>;

# 删除表的所有数据和表结构，速度快，但消耗大量IO资源
DROP TABLE <表>;
```

# 6. 事务

开始事务，可以用二者之一：

```sql
START TRANSACTION;
BEGIN;
```

提交事务：

```sql
COMMIT;
```

设置保存点，以便回滚到保存点：

```sql
SAVEPOINT savepoint_name;
```

回滚：

```sql
ROLLBACK;

# 回滚至保存点
ROLLBACK TO SAVEPOINT savepoint_name;
```

# 7. 视图

创建视图：

```sql
CREATE VIEW <视图> [(<列列表>)] AS <SELECT>;
```

删除视图：

```sql
DROP VIEW <视图>;
```

# 8. 参考

* [SQL基础教程](https://book.douban.com/subject/27055712/)

