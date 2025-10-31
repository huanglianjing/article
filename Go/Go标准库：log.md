# 1. 介绍

文档地址：https://pkg.go.dev/log

Go 标准库 log 用于实现简单的日志打印功能。

指定日志文件打印日志：

```go
type People struct {
	name string
	age  int64
}

func main() {
	logFile, _ := os.OpenFile("log/info.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	logger := log.New(logFile, "", log.Ldate|log.Ltime)
	p := People{name: "moondo", age: 20}
	logger.Printf("%#v", p)
}
```

# 2. 常量

前缀常量，定义了日志每一行的前缀，各选项可以通过或运算组合起来：

```go
const (
	Ldate         = 1 << iota     // the date in the local time zone: 2009/01/23
	Ltime                         // the time in the local time zone: 01:23:23
	Lmicroseconds                 // microsecond resolution: 01:23:23.123123.  assumes Ltime.
	Llongfile                     // full file name and line number: /a/b/c/d.go:23
	Lshortfile                    // final file name element and line number: d.go:23. overrides Llongfile
	LUTC                          // if Ldate or Ltime is set, use UTC rather than the local time zone
	Lmsgprefix                    // move the "prefix" from the beginning of the line to before the message
	LstdFlags     = Ldate | Ltime // initial values for the standard logger
)
```

# 3. 类型

## 2.1 Logger

类型定义：

```go
type Logger struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Default 返回包级别的Logger对象
func Default() *Logger

// New 创建Logger对象
func New(out io.Writer, prefix string, flag int) *Logger

// Print 打印参数内容
func (l *Logger) Print(v ...any)

// Printf 按照格式串打印内容
func (l *Logger) Printf(format string, v ...any)

// Println 打印内容加换行
func (l *Logger) Println(v ...any)
```

# 4. 函数

```go
// SetOutput 设置日志打印的地方
func SetOutput(w io.Writer)

// Writer 返回日志打印的logger
func Writer() io.Writer

// SetFlags 设置前缀标记
func SetFlags(flag int)

// SetPrefix 设置前缀内容
func SetPrefix(prefix string)

// Flags 获取日志的前缀标记
func Flags() int

// Prefix 返回日志的前缀内容
func Prefix() string

// Print 打印参数内容
func Print(v ...any)

// Printf 按照格式串打印内容
func Printf(format string, v ...any)

// Println 打印内容加换行
func Println(v ...any)

// Fatal 等于调用 Print() 后调用 os.Exit(1)
func Fatal(v ...any)
func Fatalf(format string, v ...any)
func Fatalln(v ...any)

// Panic 等于调用 Print() 后调用 panic()
func Panic(v ...any)
func Panicf(format string, v ...any)
func Panicln(v ...any)
```

