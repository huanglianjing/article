# 1. 介绍

文档地址：https://pkg.go.dev/regexp

Go 标准库 regexp 用于正则表达式搜索。

使用正则表达式匹配字符串：

```go
matched, err := regexp.Match("foo.*", []byte("seafood"))
```

编译正则表达式并匹配字符串：

```go
var re = regexp.MustCompile("^[0-9]+$")
fmt.Println(re.MatchString("a3423"))
fmt.Println(re.MatchString("342432"))
fmt.Println(re.MatchString("snakey"))
```

# 2. 类型

## 2.1 Regexp

类型定义：

```go
type Regexp struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Compile 编译正则表达式
func Compile(expr string) (*Regexp, error)

// MustCompile 编译正则表达式，如果无法处理则panic
func MustCompile(str string) *Regexp

// String 编译的原字符串
func (re *Regexp) String() string

// 是否匹配
func (re *Regexp) Match(b []byte) bool
func (re *Regexp) MatchString(s string) bool

// 返回匹配的内容
func (re *Regexp) Find(b []byte) []byte
func (re *Regexp) FindString(s string) string
func (re *Regexp) FindAll(b []byte, n int) [][]byte
func (re *Regexp) FindAllString(s string, n int) []string

// 返回匹配的索引
func (re *Regexp) FindIndex(b []byte) (loc []int)
func (re *Regexp) FindStringIndex(s string) (loc []int)
func (re *Regexp) FindAllIndex(b []byte, n int) [][]int
func (re *Regexp) FindAllStringIndex(s string, n int) [][]int

// 替换
func (re *Regexp) ReplaceAll(src, repl []byte) []byte
func (re *Regexp) ReplaceAllString(src, repl string) string
func (re *Regexp) ReplaceAllFunc(src []byte, repl func([]byte) []byte) []byte
func (re *Regexp) ReplaceAllStringFunc(src string, repl func(string) string) string

// Split 分割
func (re *Regexp) Split(s string, n int) []string
```

# 3. 函数

```go
// 匹配正则表达式和字符串
func Match(pattern string, b []byte) (matched bool, err error)
func MatchString(pattern string, s string) (matched bool, err error)

// MatchReader 匹配正则表达式和输入流
func MatchReader(pattern string, r io.RuneReader) (matched bool, err error)
```

