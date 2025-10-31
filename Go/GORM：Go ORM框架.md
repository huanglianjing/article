# 1. 简介

github仓库地址：https://github.com/go-gorm/gorm

文档地址：https://pkg.go.dev/gorm.io/gorm

GORM 是一个 Go 语言下的 ORM 库。ORM 全称 Object Relational Mapping（对象关系映射），用于实现面向对象编程语言里数据之间的转换，应用程序通过它可以连接关系型数据库并读写数据。GORM 支持的数据库类型包括： MySQL、PostreSQL、SQLite、SQL Server、TiDB。

## 1.1 模型

GORM 通过将 Go 结构体映射到数据库表来简化数据库交互，模型使用结构体定义，可以包含基本类型、指针类型、类型别名，也可以是实现了 database/sql 包中的 Scanner 和 Valuer 接口的类型。

以下示例模型中，包含基本类型 uint、string、uint8，也包含指针类型表示可空字段，sql.NullString 和 sql.NullTime 用于更多控制的可空字段，CreatedAt 和 UpdatedAt 是特殊字段，记录被创建或更新时自动填充。

```go
type User struct {
  ID           uint           // Standard field for the primary key
  Name         string         // 一个常规字符串字段
  Email        *string        // 一个指向字符串的指针, allowing for null values
  Age          uint8          // 一个未签名的8位整数
  Birthday     *time.Time     // A pointer to time.Time, can be null
  MemberNumber sql.NullString // Uses sql.NullString to handle nullable strings
  ActivatedAt  sql.NullTime   // Uses sql.NullTime for nullable time fields
  CreatedAt    time.Time      // 创建时间（由GORM自动管理）
  UpdatedAt    time.Time      // 最后一次更新时间（由GORM自动管理）
}
```

GORM 有如下默认的约定，遵循这些约定可以简化代码编写，也可以通过代码自定义这些设置。

* 将结构体名称转换为 snake_case 并加上复数形式作为表名，如 UserInfo 结构体默认表名为 user_infos
* 将字段名称转换为 snake_case 作为列名
* 使用一个名为 ID 的字段作为默认主键
* 使用 CreatedAt 和 UpdatedAt 自动跟踪记录的创建和更新时间

GORM 还提供了一个预定义结构体 gorm.Model，可以将其嵌入结构体以自动包含这些字段。

```go
type Model struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

## 1.2 标签

当 GORM 需要将数据库重的表记录的对象与程序中的结构体进行字段映射时，可以设置结构体成员变量的各种标签，以定义字段的对应关系和行为。

GORM 读取的标签为 `gorm:""`，具体设置放在双引号中，一个字段的标签可以定义多个属性，通过 `;` 分隔。

如下为一个例子，定义的结构体中有字段是主键，还有字段有索引，还有字段不对应数据库的任何字段。

```go
type People struct {
    ID      int    `gorm:"column:id;primary_key"`
    Name    string `gorm:"column:name;index"`
    Address string `gorm:"column:address;default:''"`
    Type    int    `gorm:"-"`
}
```

结构体成员的标签大小写不敏感，但建议使用 camelCase 风格。

如下标签指定了成员对应的列名，设置只允许读。

```go
Name string `gorm:"column:name;->"`
```

如下为 GORM 支持的标签。

| 标签名         | 说明                                                         | 用法                                                         |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| column         | 指定列名                                                     | gorm:"column:<column_name>"                                  |
| type           | 指定列的数据类型                                             | gorm:"type:<data_type>"                                      |
| serializer     | 指定将数据序列化或反序列化到数据库中的序列化器               |                                                              |
| size           | 列数据类型的大小或长度                                       | gorm:"size:\<size\>"                                         |
| primaryKey     | 将列定义为主键                                               | gorm:"primaryKey"                                            |
| unique         | 将列定义为唯一键                                             | gorm:"unique"                                                |
| default        | 定义列的默认值                                               | gorm:"default:<default_value>"                               |
| precision      | 指定列的精度                                                 |                                                              |
| scale          | 指定列大小                                                   |                                                              |
| not null       | 指定列为 NOT NULL                                            | gorm:"not null"                                              |
| embedded       | 嵌套字段                                                     |                                                              |
| embeddedPrefix | 嵌入字段的列名前缀                                           |                                                              |
| autoCreateTime | 创建时追踪当前时间                                           |                                                              |
| autoUpdateTime | 创建/更新时追踪当前时间                                      |                                                              |
| autoIncrement  | 自增字段                                                     | gorm:"autoIncrement"                                         |
| index          | 创建索引，在多个字段创建指定名称的索引为联合索引，一般按照字段顺序指定联合索引顺序，也可以通过 priority 区分指定顺序 | gorm:"index"<br />gorm:"index:\<name\>"<br />gorm:"index:\<name\>,priority:1" |
| uniqueIndex    | 创建唯一索引，在多个字段创建指定名称的索引为联合唯一索引     | gorm:"uniqueIndex"<br />gorm:"uniqueIndex:\<name\>"          |
| check          | 创建检查约束，例如 check:age > 13                            |                                                              |
| foreignKey     | 指定外键，该成员变量是以指定另一个成员变量为外键，从关联表读取的行 | gorm:"foreignKey:\<name\>"                                   |
| references     | 指定外键，该成员变量以指定变量，从关联表读取                 | gorm:"references:\<name\>"                                   |
| constraint     | 指定约束                                                     | gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"        |
| <-             | 设置字段写入的权限， <-:create 只创建、<-:update 只更新、<-:false 无写入权限、<- 创建和更新权限 |                                                              |
| ->             | 设置字段读的权限，->:false 无读权限                          |                                                              |
| -              | 忽略该字段，- 表示无读写，-:migration 表示无迁移权限，-:all 表示无读写迁移权限 | gorm:"-"                                                     |
| comment        | 迁移时为字段添加注释                                         | gorm:"comment:\<comment\>"                                   |

**字段权限控制**

可导出字段在使用 GORM 进行 CRUD 时拥有全部权限，可以使用标签来控制某个字段的权限，包含只读、只写、只创建、只更新或者被忽略。

```go
type User struct {
  Name string `gorm:"<-:create"` // 允许读和创建
  Name string `gorm:"<-:update"` // 允许读和更新
  Name string `gorm:"<-"`        // 允许读和写（创建和更新）
  Name string `gorm:"<-:false"`  // 允许读，禁止写
  Name string `gorm:"->"`        // 只读（除非有自定义配置，否则禁止写）
  Name string `gorm:"->;<-:create"` // 允许读和写
  Name string `gorm:"->:false;<-:create"` // 仅创建（禁止从 db 读）
  Name string `gorm:"-"`  // 通过 struct 读写会忽略该字段
  Name string `gorm:"-:all"`        // 通过 struct 读写、迁移会忽略该字段
  Name string `gorm:"-:migration"`  // 通过 struct 迁移会忽略该字段
}
```

**创建/更新时间追踪**

GORM 约定使用 CreatedAt、UpdatedAt 追踪创建/更新时间，要使用不同名称的字段，可以配置 autoCreateTime、autoUpdateTime 标签。

字段类型表示时间的精度。

```go
type User struct {
  CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
  UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
  Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳纳秒数填充更新时间
  Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
  Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳秒数填充创建时间
}
```

# 2. 使用

## 2.1 安装

使用 go get 将 GORM 包下载到 GOPATH 指定的目录下。

```bash
go get -u gorm.io/gorm
```

## 2.2 连接数据库

以连接 MySQL 为例，在变量 dsn 中指定 IP 和端口、用户名和密码、数据库名称、连接参数，这里使用了默认配置。

dsn 全称 Data Source Name，也就是数据库名称。

```go
dsn := "user:passwd@tcp(ip:port)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

可以在连接时指定一些配置。

```go
db, err := gorm.Open(mysql.New(mysql.Config{
	DSN: "gorm:gorm@tcp(127.0.0.1:3306)/gorm?charset=utf8&parseTime=True&loc=Local", // DSN data source name
	DefaultStringSize: 256, // string 类型字段的默认长度
	DisableDatetimePrecision: true, // 禁用 datetime 精度，MySQL 5.6 之前的数据库不支持
	DontSupportRenameIndex: true, // 重命名索引时采用删除并新建的方式，MySQL 5.7 之前的数据库和 MariaDB 不支持重命名索引
	DontSupportRenameColumn: true, // 用 `change` 重命名列，MySQL 8 之前的数据库和 MariaDB 不支持重命名列
	SkipInitializeWithVersion: false, // 根据当前 MySQL 版本自动配置
}), &gorm.Config{})
```

可以通过一个现有的数据库连接来初始化 *gorm.DB。

```go
gormDB, err := gorm.Open(mysql.New(mysql.Config{
	Conn: sqlDB,
}), &gorm.Config{})
```

GORM 使用 database/sql 来维护连接池。

```go
sqlDB, err := db.DB()
sqlDB.SetMaxIdleConns(10) // 设置最大空闲连接数
sqlDB.SetMaxOpenConns(100) // 设置最大打开连接数
sqlDB.SetConnMaxLifetime(time.Hour) // 设置最大连接时间
```

## 2.3 创建

Create 方法向表中插入一条记录，参数传入结构体的指针。

```go
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}

result := db.Create(&user) // 通过数据的指针来创建

user.ID             // 返回插入数据的主键
result.Error        // 返回 error
result.RowsAffected // 返回插入记录的条数
```

Create 方法还支持插入多条记录，参数传入结构体或结构体指针的切片。

```go
users := []User{
    {Name: "Jinzhu", Age: 18, Birthday: time.Now()},
    {Name: "Jackson", Age: 19, Birthday: time.Now()},
}

result := db.Create(users) // 传入切片

result.Error        // 返回 error
result.RowsAffected // 返回插入记录的条数
```

CreateInBatches 方法可以在插入多条记录时按照批次处理，并指定批次大小，防止一次插入太多数据而失败。

```go
db.CreateInBatches(users, 100)
```

Create 方法支持通过 map[string]interface{} 与 []map[string]interface{}{}来创建记录。

```go
db.Model(&User{}).Create(map[string]interface{}{
  "Name": "jinzhu", "Age": 18,
})

db.Model(&User{}).Create([]map[string]interface{}{
  {"Name": "jinzhu_1", "Age": 18},
  {"Name": "jinzhu_2", "Age": 20},
})
```

**指定和忽略对应字段**

创建记录并为指定字段赋值。

```go
db.Select("Name", "Age").Create(&user)
```

创建记录并忽略指定字段。

```go
db.Omit("Age", "CreatedAt").Create(&user)
```

**插入时更新**

在插入产生冲突时，可以指定接下来执行的动作。

```go
import "gorm.io/gorm/clause"

// 发生冲突时不执行
db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)

// 某字段发生冲突时，设置另外的字段为某个值
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"role": "user"}),
}).Create(&users)
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE ***;

// 某字段发生冲突时，设置另外的字段为 SQL 执行值
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"count": gorm.Expr("GREATEST(count, VALUES(count))")}),
}).Create(&users)
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `count`=GREATEST(count, VALUES(count));

// 某字段发生冲突时，用结构体对象成员更新指定字段的值
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
}).Create(&users)
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age);

// 发生冲突时更新所有值
db.Clauses(clause.OnConflict{
  UpdateAll: true,
}).Create(&users)
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age), ...;
```

## 2.4 查询

**查询单个对象**

通过 First、Take、Last 方法查询一个对象返回，如果没有找到记录会返回 ErrRecordNotFound 错误。

```go
// 获取第一条记录（主键升序）
db.First(&user)
// SELECT * FROM users ORDER BY id LIMIT 1;

// 获取一条记录，没有指定排序字段
db.Take(&user)
// SELECT * FROM users LIMIT 1;

// 获取最后一条记录（主键降序）
db.Last(&user)
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

result := db.First(&user)
result.RowsAffected // 返回找到的记录数
result.Error        // 返回错误

// 检查是否不存在记录的错误
if errors.Is(result.Error, gorm.ErrRecordNotFound) {}
```

如果不想产生 ErrRecordNotFound错误，可以使用 Find 方法，可以接受 struct 和 slice 的数据。如果不带 limit 则会查询整个表并返回第一个对象，性能不高且不确定。

```go
db.Limit(1).Find(&user)
```

**根据主键检索**

查询方法可以带有额外参数来按主键检索。

对于数字类型主键：

```go
db.First(&user, 10)
// SELECT * FROM users WHERE id = 10;

db.First(&user, "10")
// SELECT * FROM users WHERE id = 10;

db.Find(&users, []int{1,2,3})
// SELECT * FROM users WHERE id IN (1,2,3);
```

对于字符串类型主键：

```go
db.First(&user, "id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a")
// SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```

当目标对象有一个主键值时，将使用主键构建查询条件。

```go
var user = User{ID: 10}
db.First(&user)
// SELECT * FROM users WHERE id = 10;

var result User
db.Model(User{ID: 10}).First(&result)
// SELECT * FROM users WHERE id = 10;
```

**查询多个对象**

通过 Find 和 Scan 方法查询多个对象。它们的用法类似，区别在于 Find 支持更多的回调函数，Scan 使用时需要指定表名。

不带条件将会查询所有记录，通常加上一些条件来查询。

```go
// 查询所有记录
result := db.Find(&users)
// SELECT * FROM users;

// 根据条件查询记录
var result Result
db.Table("users").Select("name", "age").Where("name = ?", "Antonio").Scan(&result)
```

**条件**

通过字符串表示筛选条件。

```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;

// Get all matched records
db.Where("name <> ?", "jinzhu").Find(&users)
// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name IN ?", []string{"jinzhu", "jinzhu 2"}).Find(&users)
// SELECT * FROM users WHERE name IN ('jinzhu','jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
// SELECT * FROM users WHERE name LIKE '%jin%';

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```

如果对象设置了主键，条件查询又筛选了主键，将会将筛选加起来，有查不到结果的风险。

```go
var user = User{ID: 10}
db.Where("id = ?", 20).First(&user)
// SELECT * FROM users WHERE id = 10 and id = 20 ORDER BY id ASC LIMIT 1;
```

通过 struct 和 map 表示筛选条件。

当用 struct 来筛选条件时，除非指定字段，否则零值的成员将会被忽略，不会被用来查询，如 Age: 0 不会被用来查询 age = 0。

```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;

// Struct 指定字段
db.Where(&User{Name: "jinzhu"}, "name", "Age").Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;

// Struct 指定字段
db.Where(&User{Name: "jinzhu"}, "Age").Find(&users)
// SELECT * FROM users WHERE age = 0;

// Struct 指定查询条件
db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)
// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// Slice of primary keys
db.Where([]int64{20, 21, 22}).Find(&users)
// SELECT * FROM users WHERE id IN (20, 21, 22);
```

Not 方法表示 not 条件。

```go
db.Not("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE NOT name = "jinzhu" ORDER BY id LIMIT 1;

// Not In
db.Not(map[string]interface{}{"name": []string{"jinzhu", "jinzhu 2"}}).Find(&users)
// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");

// Struct
db.Not(User{Name: "jinzhu", Age: 18}).First(&user)
// SELECT * FROM users WHERE name <> "jinzhu" AND age <> 18 ORDER BY id LIMIT 1;

// Not In slice of primary keys
db.Not([]int64{1,2,3}).First(&user)
// SELECT * FROM users WHERE id NOT IN (1,2,3) ORDER BY id LIMIT 1;
```

Or 方法表示 or 条件。

```go
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';

// Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2", Age: 18}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);

// Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2", "age": 18}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);
```

**选择字段**

Select 方法能指定 select 的字段，不指定默认选择所有字段。

```go
db.Select("name", "age").Find(&users)
// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
// SELECT name, age FROM users;
```

**排序**

Order 方法进行排序。

```go
db.Order("age desc, name").Find(&users)
// SELECT * FROM users ORDER BY age desc, name;

// Multiple orders
db.Order("age desc").Order("name").Find(&users)
// SELECT * FROM users ORDER BY age desc, name;
```

**分页**

Limit 和 Offset 方法指定分页大小。

```go
db.Limit(3).Find(&users)
// SELECT * FROM users LIMIT 3;

db.Offset(3).Find(&users)
// SELECT * FROM users OFFSET 3;

db.Limit(10).Offset(5).Find(&users)
// SELECT * FROM users OFFSET 5 LIMIT 10;

// Cancel limit condition with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
// SELECT * FROM users LIMIT 10; (users1)
// SELECT * FROM users; (users2)

// Cancel offset condition with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
// SELECT * FROM users OFFSET 10; (users1)
// SELECT * FROM users; (users2)
```

**聚合**

```go
db.Model(&User{}).Select("name, sum(age) as total").Where("name LIKE ?", "group%").Group("name").First(&result)
// SELECT name, sum(age) as total FROM `users` WHERE name LIKE "group%" GROUP BY `name` LIMIT 1

db.Model(&User{}).Select("name, sum(age) as total").Group("name").Having("name = ?", "group").Find(&result)
// SELECT name, sum(age) as total FROM `users` GROUP BY `name` HAVING name = "group"
```

**Distinct**

```go
db.Distinct("name", "age").Order("name, age desc").Find(&results)
```

**表连接**

```go
db.Joins("Company").Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;

// inner join
db.InnerJoins("Company").Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` INNER JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;
```

**记数**

```go
// 计数
var count int64
db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)
// SELECT count(1) FROM users WHERE name = 'jinzhu'

// 为不同 name 计数
db.Model(&User{}).Distinct("name").Count(&count)
// SELECT COUNT(DISTINCT(`name`)) FROM `users`

// 按名称分组后计数
db.Model(&User{}).Group("name").Count(&count)
// SELECT COUNT(1) FROM `users` GROUP BY `name`
```

**子查询**

```go
// 简单的子查询
db.Where("amount > (?)", db.Table("orders").Select("AVG(amount)")).Find(&orders)
// SQL: SELECT * FROM "orders" WHERE amount > (SELECT AVG(amount) FROM "orders");

// 内嵌子查询
subQuery := db.Select("AVG(age)").Where("name LIKE ?", "name%").Table("users")
db.Select("AVG(age) as avgage").Group("name").Having("AVG(age) > (?)", subQuery).Find(&results)
// SQL: SELECT AVG(age) as avgage FROM `users` GROUP BY `name` HAVING AVG(age) > (SELECT AVG(age) FROM `users` WHERE name LIKE "name%")

// 在 FROM 子句中使用子查询
db.Table("(?) as u", db.Model(&User{}).Select("name", "age")).Where("age = ?", 18).Find(&User{})
// SQL: SELECT * FROM (SELECT `name`,`age` FROM `users`) as u WHERE `age` = 18
```

## 2.5 更新

**更新所有列**

Save 方法保存所有字段，即使字段是零值。如果保存值不包含主键，将执行 Create，否则 Update 所有字段。

```go
db.First(&user)
user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)
// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
```

**更新指定列**

Update 方法更新单列，需要指定一些条件，否则会引起 ErrMissingWhereClause 错误。

Updates 方法更新多列，支持 struct 和 map 参数，当使用 struct 更新是默认只更新非零值字段。

在更新时可以通过 Select 和 Omit 方法选择、忽略某些字段。

```go
// 更新单列
db.Model(&User{}).Where("active = ?", true).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true;

// 根据 struct 更新属性，只会更新非零值的字段
db.Model(&user).Updates(User{Name: "hello", Age: 18, Active: false})
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// 根据 map 更新属性
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

**使用表达式更新**

```go
db.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))
// UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3;

db.Model(&product).UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3;
```

**返回修改行数据**

```go
// 返回所有列
var users []User
db.Model(&users).Clauses(clause.Returning{}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
// UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING *

// 返回指定列
db.Model(&users).Clauses(clause.Returning{Columns: []clause.Column{{Name: "name"}, {Name: "salary"}}}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
// UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING `name`, `salary`
```

## 2.6 删除

**删除一条记录**

删除一条记录时，删除对象需要指定主键，否则会触发批量删除。

```go
// Email 的 ID 是 `10`
db.Delete(&email)
// DELETE from emails where id = 10;

// 带额外条件的删除
db.Where("name = ?", "jinzhu").Delete(&email)
// DELETE from emails where id = 10 AND name = "jinzhu";
```

**根据主键删除**

```go
db.Delete(&User{}, 10)
// DELETE FROM users WHERE id = 10;

db.Delete(&User{}, "10")
// DELETE FROM users WHERE id = 10;

db.Delete(&users, []int{1,2,3})
// DELETE FROM users WHERE id IN (1,2,3);
```

**批量删除**

如果对象不包含主键的值，则只会根据匹配条件删除。

```go
db.Where("email LIKE ?", "%jinzhu%").Delete(&Email{})
// DELETE from emails where email LIKE "%jinzhu%";

db.Delete(&Email{}, "email LIKE ?", "%jinzhu%")
// DELETE from emails where email LIKE "%jinzhu%";
```

如果主键设置了值，则还会判断主键的值。

```go
var users = []User{{ID: 1}, {ID: 2}, {ID: 3}}
db.Delete(&users)
// DELETE FROM users WHERE id IN (1,2,3);

db.Delete(&users, "name LIKE ?", "%jinzhu%")
// DELETE FROM users WHERE name LIKE "%jinzhu%" AND id IN (1,2,3); 
```

**返回删除行的数据**

```go
// 回写所有的列
var users []User
DB.Clauses(clause.Returning{}).Where("role = ?", "admin").Delete(&users)
// DELETE FROM `users` WHERE role = "admin" RETURNING *

// 回写指定的列
DB.Clauses(clause.Returning{Columns: []clause.Column{{Name: "name"}, {Name: "salary"}}}).Where("role = ?", "admin").Delete(&users)
// DELETE FROM `users` WHERE role = "admin" RETURNING `name`, `salary`
```

## 2.7 原生SQL

Raw 和 Scan 方法可以直接写完整的 SQL 语句并执行。

```go
type Result struct {
  ID   int
  Name string
  Age  int
}
db.Raw("SELECT name, age FROM users WHERE name = ?", "Antonio").Scan(&result)

var age int
db.Raw("SELECT SUM(age) FROM users WHERE role = ?", "admin").Scan(&age)
```

Exec 方法直接执行一个 SQL 语句。

```go
db.Exec("DROP TABLE users")

db.Exec("UPDATE orders SET shipped_at = ? WHERE id IN ?", time.Now(), []int64{1, 2, 3})
```

**命名参数**

GORM 支持不同形式的命名参数传入方式。

```go
// sql.Named
db.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
// SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu"

// map
db.Where("name1 = @name OR name2 = @name", map[string]interface{}{"name": "jinzhu2"}).First(&result3)
// SELECT * FROM `users` WHERE name1 = "jinzhu2" OR name2 = "jinzhu2" ORDER BY `users`.`id` LIMIT 1

// struct
db.Raw("SELECT * FROM users WHERE (name1 = @Name AND name3 = @Name) AND name2 = @Name2", NamedArgument{Name: "jinzhu", Name2: "jinzhu2"}).Find(&user)
// SELECT * FROM users WHERE (name1 = "jinzhu" AND name3 = "jinzhu") AND name2 = "jinzhu2"
```

**只生成 SQL**

GORM 使用 database/sql 的参数占位符来构建 SQL 语句，可以使用 ToSQL 语句只生成 SQL 而不执行。

```go
sql := db.ToSQL(func(tx *gorm.DB) *gorm.DB {
  return tx.Model(&User{}).Where("id = ?", 100).Limit(10).Order("age desc").Find(&[]User{})
})
```

**获取返回结果**

将结果返回为 *sql.Row 或 *sql.Rows，再通过 Scan 方法解析其中的信息，可以在解析的时候做其他的业务逻辑处理。

```go
// 一行
row := db.Table("users").Where("name = ?", "jinzhu").Select("name", "age").Row()
row.Scan(&name, &age)

// 多行
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows()
defer rows.Close()
for rows.Next() {
  // 将 row 扫描至成员变量
  rows.Scan(&name, &age, &email)

  // 将 row 扫描至 struct 变量
  db.ScanRows(rows, &user)

  // 业务逻辑...
}
```

# 3. 功能

## 3.1 上下文

GORM 的上下文支持由 WithContext 方法启用，可以增强 Go 应用程序中数据库操作的灵活性和控制力。 它允许在不同的操作模式、超时设置以及甚至集成到钩子/回调和中间件中进行上下文管理。

```go
db.WithContext(ctx).Find(&users)

// 在多个操作中保留 context
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```

通过 context 来设置超时时间。

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

db.WithContext(ctx).Find(&users)
```

可以在 hooks 或 callbacks 中使用 context 获取信息。

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  ctx := tx.Statement.Context
  // ... use context
  return
}
```

## 3.2 错误处理

GORM 将错误处理集成到其可链式方法语法中。 *gorm.DB 实例包含一个 Error 字段，当发生错误时会被设置。

通常的做法是在执行数据库操作后，特别是在完成方法后，检查这个字段。

```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
  // 处理错误...
}

if result := db.Where("name = ?", "jinzhu").First(&user); result.Error != nil {
  // 处理错误...
}
```

当使用 First、Last、Take 等方法未找到记录时，GORM 会返回 ErrRecordNotFound。

```go
err := db.First(&user, 100).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
  // 处理未找到记录的错误...
}
```

解析数据库的特定代码错误。

```go
import (
    "github.com/go-sql-driver/mysql"
    "gorm.io/gorm"
)

result := db.Create(&newRecord)
if result.Error != nil {
    if mysqlErr, ok := result.Error.(*mysql.MySQLError); ok {
        switch mysqlErr.Number {
        case 1062: // MySQL中表示重复条目的代码
            // 处理重复条目
        // 为其他特定错误代码添加案例
        default:
            // 处理其他错误
        }
    } else {
        // 处理非MySQL错误或未知错误
    }
}
```

## 3.3 钩子

Hook（钩子）是在创建、查询、更新、删除等操作之前、之后调用的函数。如果为模型定义了指定的方法，它会在创建、更新、查询、删除时自动被调用。如果任何回调返回错误，GORM 将停止后续的操作并回滚事务。

钩子方法的函数签名应该是 `func(*gorm.DB) error`，基于参数 *gorm.DB 的操作会在同一个事务中。

**创建**

创建时可用的 hook：

```go
// 开始事务
BeforeSave
BeforeCreate
// 关联前的 save
// 插入记录至 db
// 关联后的 save
AfterCreate
AfterSave
// 提交或回滚事务
```

方法定义示例：

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  u.UUID = uuid.New()

  if !u.IsValid() {
    err = errors.New("can't save invalid data")
  }
  return
}

func (u *User) AfterCreate(tx *gorm.DB) (err error) {
  if u.ID == 1 {
    tx.Model(u).Update("role", "admin")
  }
  return
}
```

**更新**

更新时可用的 hook：

```go
// 开始事务
BeforeSave
BeforeUpdate
// 关联前的 save
// 更新 db
// 关联后的 save
AfterUpdate
AfterSave
// 提交或回滚事务
```

方法定义示例：

```go
func (u *User) BeforeUpdate(tx *gorm.DB) (err error) {
  if u.readonly() {
    err = errors.New("read only user")
  }
  return
}

func (u *User) AfterUpdate(tx *gorm.DB) (err error) {
  if u.Confirmed {
    tx.Model(&Address{}).Where("user_id = ?", u.ID).Update("verfied", true)
  }
  return
}
```

**删除**

删除时可用的 hook：

```go
// 开始事务
BeforeDelete
// 删除 db 中的数据
AfterDelete
// 提交或回滚事务
```

方法定义示例：

```go
func (u *User) AfterDelete(tx *gorm.DB) (err error) {
  if u.Confirmed {
    tx.Model(&Address{}).Where("user_id = ?", u.ID).Update("invalid", false)
  }
  return
}
```

**查询**

查询时可用的 hook：

```go
// 从 db 中加载数据
// Preloading (eager loading)
AfterFind
```

方法定义示例：

```go
func (u *User) AfterFind(tx *gorm.DB) (err error) {
  if u.MemberShip == "" {
    u.MemberShip = "user"
  }
  return
}
```

## 3.4 事务

GORM 默认在事务中执行写入操作（创建、更新、删除）。

**禁用事务**

在初始化时禁用它将获得大约 30%+ 性能提升。

```go
// 全局禁用
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})

// 当前会话禁用
tx := db.Session(&Session{SkipDefaultTransaction: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)
```

**使用事务**

定义事务函数 `func(tx *gorm.DB) error`，然后通过 Transaction 方法调用它，函数体内执行一系列操作，将会以事务进行。

```go
// 定义事务函数
txF := func(tx *gorm.DB) error {
	if err := tx.Create(&User{Uid: 123, Name: "name1"}).Error; err != nil {
		// 返回任何错误都会回滚事务
		return err
	}
	if err := tx.Create(&User{Uid: 124, Name: "name2"}).Error; err != nil {
		return err
	}

	// 返回 nil 提交事务
	return nil
}
// Transaction 方法调用事务
err := db.Transaction(txF)

// 直接在 Transaction 方法中定义事务函数
err := db.Transaction(func(tx *gorm.DB) error {
  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
    // 返回任何错误都会回滚事务
    return err
  }
  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
    return err
  }

  // 返回 nil 提交事务
  return nil
})
```

GORM 支持嵌套事务。

```go
db.Transaction(func(tx *gorm.DB) error {
  tx.Create(&user1)

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx2.Create(&user2)
    return errors.New("rollback user2") // Rollback user2
  })

  tx.Transaction(func(tx3 *gorm.DB) error {
    tx3.Create(&user3)
    return nil
  })

  return nil // Commit user1, user3
})
```

**手动事务**

通过直接调用事务控制方法来执行事务。

```go
// 开始事务
tx := db.Begin()

// 在事务中执行一些 db 操作
tx.Create(...)

// 遇到错误时回滚事务
tx.Rollback()

// 否则，提交事务
tx.Commit()
```

**保存点**

可以设置保存点，并且回滚至保存点。

```go
tx := db.Begin()
tx.Create(&user1)

tx.SavePoint("sp1")
tx.Create(&user2)
tx.RollbackTo("sp1") // Rollback user2

tx.Commit() // Commit user1
```

## 3.5 日志

**默认 logger 实现**

GORM 的默认 logger会打印慢 SQL 和错误。

```go
newLogger := logger.New(
  log.New(os.Stdout, "\r\n", log.LstdFlags), // io writer
  logger.Config{
    SlowThreshold:              time.Second,   // Slow SQL threshold
    LogLevel:                   logger.Silent, // Log level
    IgnoreRecordNotFoundError: true,           // Ignore ErrRecordNotFound error for logger
    ParameterizedQueries:      true,           // Don't include params in the SQL log
    Colorful:                  false,          // Disable color
  },
)

// globally mode
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  Logger: newLogger,
})

// session mode
tx := db.Session(&Session{Logger: newLogger})
tx.First(&user)
tx.Model(&user).Update("Age", 18)
```

日志级别有 Silent、Error、Warn、Info。

```go
// 连接全局设置级别
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  Logger: logger.Default.LogMode(logger.Silent),
})

// 单个操作设置级别
db.Debug().Where("name = ?", "jinzhu").First(&User{})
```

**自定义 logger**

可以自定义 logger，只需要实现以下接口：

```go
type Interface interface {
    LogMode(LogLevel) Interface
    Info(context.Context, string, ...interface{})
    Warn(context.Context, string, ...interface{})
    Error(context.Context, string, ...interface{})
    Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error)
}
```

## 3.6 迁移

当我们用多个结构体定义了数据库的表，可以在运行时创建表结构。

```go
db.AutoMigrate(&User{})
db.AutoMigrate(&User{}, &Product{}, &Order{})
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```

GORM 的 Migrator 接口提供了包括自动迁移、建表、删表、增删改列、增删改索引、视图、约束等方法。

```go
type Migrator interface {
  // AutoMigrate
  AutoMigrate(dst ...interface{}) error

  // Database
  CurrentDatabase() string
  FullDataTypeOf(*schema.Field) clause.Expr

  // Tables
  CreateTable(dst ...interface{}) error
  DropTable(dst ...interface{}) error
  HasTable(dst interface{}) bool
  RenameTable(oldName, newName interface{}) error
  GetTables() (tableList []string, err error)

  // Columns
  AddColumn(dst interface{}, field string) error
  DropColumn(dst interface{}, field string) error
  AlterColumn(dst interface{}, field string) error
  MigrateColumn(dst interface{}, field *schema.Field, columnType ColumnType) error
  HasColumn(dst interface{}, field string) bool
  RenameColumn(dst interface{}, oldName, field string) error
  ColumnTypes(dst interface{}) ([]ColumnType, error)

  // Views
  CreateView(name string, option ViewOption) error
  DropView(name string) error

  // Constraints
  CreateConstraint(dst interface{}, name string) error
  DropConstraint(dst interface{}, name string) error
  HasConstraint(dst interface{}, name string) bool

  // Indexes
  CreateIndex(dst interface{}, name string) error
  DropIndex(dst interface{}, name string) error
  HasIndex(dst interface{}, name string) bool
  RenameIndex(dst interface{}, oldName, newName string) error
}
```

# 4. 参考

* [GORM 指南](https://gorm.io/zh_CN/docs/)

