# 1. 简介

github仓库地址：https://github.com/jinzhu/copier

文档地址：https://pkg.go.dev/github.com/jinzhu/copier

copier 是 Go 语言的一个实现不同结构体之间成员复制的库。

它可以实现不同结构体的对象到对象的复制、对象到 slice 的复制、slice 到 slice 的复制，支持同名方法和成员、同名成员和方法的复制，还支持 map 到 map 间的复制。

复制结构体成员默认基于成员名称，可以给成员添加标签实现不同的特性。

```go
type Employee struct 
	Name string `copier:"must"` // 如果未复制则 panic
	Age int32 `copier:"must,nopanic"` // 如果未复制则返回 error
	Salary int `copier:"-"` // 忽略该字段
	EmployeeId int64 `copier:"EmployeeNum"` // 指定复制的字段索引，源结构体和目标结构体都要定义
}
```

copier 的实现基于反射，用于对性能没有太多要求的场景，能简化代码量，提高开发效率。在对性能要求较高的场景，还是建议通过手动列举字段的方式进行复制。

# 2. 使用

使用 go get 将 copier 包下载到 GOPATH 指定的目录下。

```bash
go get github.com/jinzhu/copier
```

主要的复制方法有两个，不带配置的和带有配置的复制。

```go
// Copy 直接复制
func Copy(toValue interface{}, fromValue interface{}) (err error)

// CopyWithOption 带有配置的复制
func CopyWithOption(toValue interface{}, fromValue interface{}, opt Option) (err error)
```

配置相关的结构体如下：

```go
// Option 配置
type Option struct {
	IgnoreEmpty   bool // 忽略复制零值的成员
	CaseSensitive bool // 区分大小写
	DeepCopy      bool // 深度复制
	Converters    []TypeConverter // 指定类型转化的逻辑
	FieldNameMapping []FieldNameMapping // 指定对应成员复制的映射表
}

// TypeConverter 类型转化的逻辑
type TypeConverter struct {
	SrcType interface{}
	DstType interface{}
	Fn      func(src interface{}) (dst interface{}, err error)
}

// FieldNameMapping 对应成员复制的映射表
type FieldNameMapping struct {
	SrcType interface{}
	DstType interface{}
	Mapping map[string]string
}
```

不带配置的复制。

```go
// User 源结构体
type User struct {
	Name         string
	Role         string
	Age          int32
	EmployeeCode int64 `copier:"EmployeeNum"` // 指定复制的索引
	Salary       int
}

func (u *User) DoubleAge() int32 { // 复制时匹配到目标结构体同名字段后执行
	return u.Age * 2
}

// Employee 目标结构体
type Employee struct {
	Name         string
	Age          int32
	DoubleAge    int32
	EmployeeId   int64 `copier:"EmployeeNum"` // 指定复制的索引
	Salary       int   `copier:"-"`           // 忽略复制
	EmployeeRole string
}

func (e *Employee) Role(role string) { // 复制时匹配到源结构体同名字段后执行
	e.EmployeeRole = role
}

func main() {
	var (
		user     = User{Name: "Jinzhu", Age: 18, Role: "Admin", Salary: 200000}
		employee = Employee{Salary: 150000}
	)

	// 复制
	copier.Copy(&employee, &user)

	fmt.Printf("%#v \n", employee)
	// main.Employee{Name:"Jinzhu", Age:18, DoubleAge:36, EmployeeId:0, Salary:150000, EmployeeRole:"Admin"}
}
```

带有配置的复制。

```go
copier.CopyWithOption(&to, &from, copier.Option{IgnoreEmpty: true, DeepCopy: true})
```

map 到 map 的复制。

```go
	map1 := map[int]int{3: 6, 4: 8}
	map2 := map[int32]int8{}
	copier.Copy(&map2, map1)

	fmt.Printf("%#v \n", map2)
	// map[int32]int8{3:6, 4:8}
```

