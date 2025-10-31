# 1. 介绍

文档地址：https://pkg.go.dev/errors

Go 标准库 errors 用于操作error。

如果一个error e包含以下任一方法，则可以包裹另一个error。

```go
Unwrap() error
Unwrap() []error
```

自定义error结构体示例：

```go
type MyError struct {
	When time.Time
	What string
}
func (e MyError) Error() string {
	return fmt.Sprintf("%v: %v", e.When, e.What)
}

err := MyError{
	time.Date(1989, 3, 15, 22, 30, 0, 0, time.UTC),
	"the file system has gone away",
}
fmt.Println(err)
```

# 2. 函数

```go
// New 创建error
func New(text string) error

// As err是否和target类型匹配，如果是则设置target的内容
func As(err error, target any) bool

// Is err是否和target类型匹配
func Is(err, target error) bool

// Join 将多个error包裹起来
func Join(errs ...error) error

// Unwrap 调用err.Unwrap打开包裹
func Unwrap(err error) error
```

