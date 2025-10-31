# 1. 介绍

文档地址：https://pkg.go.dev/builtin

Go 标准库 builtin 提供 Go 预定义的标识符。

# 2. 常量

```go
const (
	true  = 0 == 0 // Untyped bool.
	false = 0 != 0 // Untyped bool.
)

const iota = 0 // Untyped int.
```

# 3. 变量

```go
var nil Type
```

# 4. 类型

公共类型：

```go
type Type int
type Type1 int
type any = interface{}
type comparable interface{ comparable }
```

布尔类型：

```go
type bool bool
```

字符类型：

```go
type byte = uint8
type rune = int32
```

整数类型：

```go
type IntegerType int
type int int
type int8 int8
type int16 int16
type int32 int32
type int64 int64
type uint uint
type uint8 uint8
type uint16 uint16
type uint32 uint32
type uint64 uint64
type uintptr uintptr
```

浮点类型：

```go
type FloatType float32
type float32 float32
type float64 float64
```

复数类型：

```go
type ComplexType complex64
type complex64 complex64
type complex128 complex128
```

异常：

```go
type error interface {
	Error() string
}
```

# 5. 函数

```go
// new 创建对象并返回指针
func new(Type) *Type

// make 生成并出示话切片、map、通道
func make(t Type, size ...IntegerType) Type

// 打印对象
func print(args ...Type)
func println(args ...Type)

// cap 根据参数类型获得容量
func cap(v Type) int

// len 根据参数类型获得长度
func len(v Type) int

// max 最大值
func max[T cmp.Ordered](x T, y ...T) T

// min 最小值
func min[T cmp.Ordered](x T, y ...T) T

// complex 生成复数
func complex(r, i FloatType) ComplexType

// real 获取复数的实数部分
func real(c ComplexType) FloatType

// imag 获取复数的虚数部分
func imag(c ComplexType) FloatType

// append 向切片追加元素
func append(slice []Type, elems ...Type) []Type

// copy 切片复制
func copy(dst, src []Type) int

// delete 删除map的元素
func delete(m map[Type]Type1, key Type)

// clear 清空切片或map
func clear[T ~[]Type | ~map[Type]Type1](t T)

// close 关闭通道
func close(c chan<- Type)

// panic 异常
func panic(v any)

// recover 获取panic的返回
func recover() any
```

