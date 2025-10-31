# 1. 系统

```mysql
# 数据库版本
select version();

# 当前用户
select user();

# 当前数据库
select database();
```

# 2. 时间日期

时间转换

```mysql
# 时间 -> 字符串
select date_format(now(), '%Y-%m-%d %H:%i:%s');

# 时间 -> 时间戳
select unix_timestamp();
select unix_timestamp(now());

# 时间戳 -> 字符串
select from_unixtime(1738901130, '%Y-%m-%d %H:%i:%s');

# 时间戳 -> 时间
select from_unixtime(1738901130);

# 字符串 -> 时间
select str_to_date('2025-02-07 12:05:30', '%Y-%m-%d %H:%i:%s');

# 字符串 -> 时间戳
select unix_timestamp('2025-02-07 12:05:30');
```

获取当前时间

```mysql
# 当前日期
select curdate();

# 当前时间
select curtime();

# 当前日期+时间，执行开始时得到
select now();

# 当前日期+时间，执行时得到
select sysdate();

# 当前时间戳
select current_timestamp();
```

# 3. 数字

```mysql
# 绝对值
select abs(x);

# 符号，负数/0/整数分别返回 -1/0/1
select sign(x);

# 向上取整
select ceil(x);

# 向下取整
select floor(x);

# 最大值
select max(x);

# 最小值
select min(x);

# 平均值
select avg(x);

# 求和
select sum(x);

# 数量
select count(*);

# 0 - 1 的随机数
select rand();
```

# 4. 字符串

```mysql
# 长度
select length(s);

# 字符串拼接
select concat(s1, s2);

# 转小写字母
select lower(s);

# 转大写字母
select upper(s);

# 子字符串
select substr(s, start, length);

# 字符串反转
select reverse(s);

# 去除首尾空格
select trim(s);

# 字符串比较
select strcmp(s1, s2);
```

# 5. 加密解密

```mysql
# 加密
select encode(str, passwd);

# 解密
select decode(str, passwd);

# aes加密
select aes_encrypt(str, key);

# aes解密
select aes_decrypt(str, key);

# MD5
select md5(s);
```

# 6. 控制流

```mysql
# 如果 expr1 为真，返回 expr2，否则返回 expr3
select if(expr1, expr2, expr3);

# 如果 expr1 非 NULL 则返回，否则返回 expr2
select ifnull(expr1, expr2);

# 如果 expr1 expr2 相等则返回 NULL，否则返回 expr1
select nullif(expr1, expr2);

# 多种情况判断
select case when <condition> then <result> when <condition> then <result> else <result> end

# 变量多种值判断
select case <value> when <value> then <result> when <value> then <result> else <result> end
```

# 7. 类型转换

```mysql
# 将值转为指定数据类型
select cast(expr as type);
```

