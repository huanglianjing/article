# 1. 介绍

文档地址：https://pkg.go.dev/strconv

Go 标准库 strconv 用于字符串和其他类型的相互转化。

数字转化示例：

```go
// 字符串转整数
i, err := strconv.Atoi("-42")

// 整数转字符串
s := strconv.Itoa(-42)

// 字符串转int，指定进制和位数
i, err := strconv.ParseInt("-42", 10, 64)

// 数字转字符串
s := strconv.FormatInt(-42, 16)
```

# 2. 常量与变量

```go
// IntSize int和uint的位数
const IntSize = intSize

// ErrRange 值超出范围
var ErrRange = errors.New("value out of range")

// ErrSyntax 语法错误
var ErrSyntax = errors.New("invalid syntax")
```

# 3. 函数

```go
// Atoi string转int
func Atoi(s string) (int, error)

// Itoa int转string
func Itoa(i int) string

// 其他类型转string
func FormatBool(b bool) string
func FormatComplex(c complex128, fmt byte, prec, bitSize int) string
func FormatFloat(f float64, fmt byte, prec, bitSize int) string
func FormatInt(i int64, base int) string
func FormatUint(i uint64, base int) string

// string转其他类型
func ParseBool(str string) (bool, error)
func ParseComplex(s string, bitSize int) (complex128, error)
func ParseFloat(s string, bitSize int) (float64, error)
func ParseInt(s string, base int, bitSize int) (i int64, err error)
func ParseUint(s string, base int, bitSize int) (uint64, error)
```

