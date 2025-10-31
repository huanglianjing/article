# 1. 概述

泛型（Generics）是程序设计语言的一种风格或范式，允许程序员在强类型程序设计语言中编写代码时使用一些以后才指定的类型，在实例化时作为参数指明这些类型。

Go 从 1.18 版本增加了对泛型的支持，算是自 Go 1.0 以来最大的语法变化了。

假设我们需要实现两个数的相加，原本实现 int 类型的版本和 float 类型的版本需要给它们分别定义函数。

```go
func AddInt(a int, b int) int {
	return a + b
}
func AddFloat(a, b float32) float32 {
	return a + b
}
```

也可以使用反射来实现。

```go
func Add(a interface{}, b interface{}) interface{} {
	switch a.(type) {
	case int:
		return a.(int) + b.(int)
	case float32:
		return a.(float32) + b.(float32)
	default:
		return nil
	}
}
```

前者对于每一种类型都需要重新实现一个新的函数，代码开发量比较大，而后者使用反射则会对性能有所影响，且也需要在一个函数里面罗列各种类型。

如果使用泛型，则会变得简洁的多。

```go
func Add[T int | float32 | float64](a, b T) T {
	return a + b
}
```

# 2. 语法

在 Go 的语法中，interface{} 用于表示任意类型，因为这是一个没有任何方法的接口，所以任何类型都实现了这个接口。而在 1.18 版本中，引入了新的类型 any 用来表示 interface{}，使代码更加简化。

```go
type any = interface{}
```

泛型的核心是类型参数的定义，Go 的类型参数通过方括号 [] 包裹起来，格式为`[T Constraint]`。类型参数由两部分构成，第一个是类型参数名，一般使用大写字母如 T，第二部分是类型参数约束，可以用 any 表示任意类型，也可以是某个接口，表示类型参数需要实现这个接口。

泛型函数的定义格式：`func FunctionName[T Constraint](param T) { ... }`

范型类型的定义格式：`type TypeName[T Constraint] ...`

类型参数这里可以定义一到多个类型。

## 2.1 泛型约束

Go 提供了一些预定义约束：

* any 表示任何类型
* comparable 表示可比较，即支持 == 和 !=

```go
func Add[T any](a, b T) {}
func Add[T comparable](a, b T) {}
```

约束可以使用某个接口类型，表示该类型参数必须实现该接口。

```go
// 类型 T 需要实现接口 fmt.Stringer 的 String 方法
func Print[T fmt.Stringer](a T) {
	fmt.Println(a.String())
}
```

约束也可以是多个类型或接口的集合，然后用 | 连接起来，表示类型 T 需要是这几个类型之一。

```go
func Add[T int | float32 | float64](a, b T) T {
	return a + b
}
```

定义类型集表示一堆类型的集合，先定义类型集，然后在约束中使用它。

```go
// 定义类型集
type number interface {
	int | int32 | uint32 | int64 | uint64 | float32 | float64
}

// 使用类型集
func Add[T number](a, b T) T {
	return a + b
}
```

对于代码中的类型别名，可以用 ~ 加上原先的基础类型来包含它，称作近似约束元素。

```go
// 类型别名
type SendStatus int
type ReceiveStatus int

// ~int 可以代表所有基础类型为 int 的类型
type AnyStatus interface{ ~int }

// 使用近似约束元素
[T ~int]
[T AnyStatus]
```

联合约束元素是一系列用 | 连起来的约束元素。

```go
type Integer interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}
```

类型集通过 | 连接起来表示求它们的并集，也可以定义它们的交集，当两个类型集没有交集时则结果是空集。

```go
type Int interface {
	int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64
}
type Uint interface {
	uint | uint8 | uint16 | uint32 | uint64
}

// Int和Uint的交集
type Status interface {  
	Int
	Uint
}
```

## 2.2 泛型函数

泛型函数定义了类型参数并在函数体中用到它。

```go
// 泛型函数
func Equal[T comparable](a, b T) bool {
	return a == b
}

// 使用时指定具体的类型
Equal(1, 2)
Equal("fdsg4r53", "fdsfds")
```

## 2.3 泛型类型

泛型类型定义时的类型参数可以在类型中使用到，而使用泛型类型时需要指定具体的类型。

```go
// 泛型类型
type Stack[T any] struct {
	data []T
}
func (s *Stack[T]) Push(x T) {
	s.data = append(s.data, x)
}

// 使用时指定具体的类型
s1 := Stack[int]{}
s1.Push(1)
s2 := Stack[string]{}
s2.Push("abc")
```

目前不支持泛型方法，但是可以通过泛型类型来实现，方法使用泛型类型中定义的类型参数。

```go
// 泛型类型
type Person[T int | string] struct{}

// 方法
func (p *Person[T]) Say(s T) {
	fmt.Println(s)
}
```

范型类型可以通过嵌套，定义出更加复杂的类型。

```go
type Slice[T int|string|float32|float64] []T

type SMap1[T int|string] map[string]Slice[T]
type SMap2[T Slice[int] | Slice[string]] map[string]T
```

## 2.4 泛型接口

接口可以定义为泛型接口，是可以处理多种类型数据的接口。

```go
type Container[T any] interface {
	Len() int
	Add(T)
	Remove() T
}
```

# 3. 其他特性

匿名函数不能定义为泛型函数，但是匿名函数的参数可以是定义好的类型实参。

```go
// 错误，匿名函数不能自己定义类型实参
fn := func[T int | float32](a, b T) T {
	return a + b
}

// 正确，匿名函数可使用已经定义好的类型形参
func MyFunc[T int | int32 | int64](a, b T) {
	fn2 := func(i T, j T) T {
		return i % j
	}
	fn2(a, b)
}
```

Go 编译器在编译时会将泛型代码转换为具体类型的代码，叫做泛型特化，避免了运行时的类型检查和类型转换，可以提高代码的性能和运行效率。

# 4. 应用

实现一个基于泛型的队列。

```go
// Queue 泛型队列
type Queue[T any] struct {
	items []T
}

// Put 放入队列尾部
func (q *Queue[T]) Put(value T) {
	q.items = append(q.items, value)
}

// Pop 从队列头部取出并删除
func (q *Queue[T]) Pop() (T, bool) {
	var value T
	if len(q.items) == 0 {
		return value, false
	}
	value = q.items[0]
	q.items = q.items[1:]
	return value, true
}

// Size 队列大小
func (q *Queue[T]) Size() int {
	return len(q.items)
}
```

# 5. 参考

* [深入理解Golang的泛型](https://www.kunkkawu.com/archives/shen-ru-li-jie-golang-de-fan-xing)

