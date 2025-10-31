# 1. 介绍

文档地址：https://pkg.go.dev/strings

Go 标准库 strings 用于字符串相关的操作。

# 2. 类型

## 2.1 Builder

类型定义：

```go
// Builder 字符串构建器
type Builder struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Cap 容量大小
func (b *Builder) Cap() int

// Grow 增加容量
func (b *Builder) Grow(n int)

// Len 字符串长度
func (b *Builder) Len() int

// Reset 清空
func (b *Builder) Reset()

// 返回现有的字符串
func (b *Builder) String() string

// 追加内容
func (b *Builder) Write(p []byte) (int, error)
func (b *Builder) WriteString(s string) (int, error)
```

## 2.2 Reader

类型定义：

```go
// Reader 字符串读取器
type Reader struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// NewReader 创建对象
func NewReader(s string) *Reader

// Len 未读取的字符串长度
func (r *Reader) Len() int

// Size 字符串本身的长度
func (r *Reader) Size() int64

// Reset 重置
func (r *Reader) Reset(s string)

// Read 读取
func (r *Reader) Read(b []byte) (n int, err error)

// ReadByte 读取一个字节
func (r *Reader) ReadByte() (byte, error)

// WriteTo 输出
func (r *Reader) WriteTo(w io.Writer) (n int64, err error)
```

## 2.3 Replacer

类型定义：

```go
// Replacer 字符串替换器
type Replacer struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// NewReplacer 创建对象，指定多组替换内容
// 例子：r := strings.NewReplacer("<", "&lt;", ">", "&gt;")
func NewReplacer(oldnew ...string) *Replacer

// Replace 用自身的每组替换内容替换字符串s
func (r *Replacer) Replace(s string) string

// WriteString 替换字符串s输出到w
func (r *Replacer) WriteString(w io.Writer, s string) (n int, err error)
```

# 3. 函数

```go
// 复制
func Clone(s string) string

// Compare 比较，0 if a == b, -1 if a < b, 1 if a > b
func Compare(a, b string) int

// Contains 是否包含子串
func Contains(s, substr string) bool

// ContainsAny 是否包含任意字符
func ContainsAny(s, chars string) bool

// Index，子串出现的第一个索引，-1表示无
func Index(s, substr string) int

// LastIndex 子串出现的最后一个索引，-1表示无
func LastIndex(s, substr string) int

// IndexAny 任意字符出现的第一个索引，-1表示无
func IndexAny(s, chars string) int

// LastIndexAny 任意字符出现的最后一个索引，-1表示无
func LastIndexAny(s, chars string) int

// 是否包含前后缀
func HasPrefix(s, prefix string) bool
func HasSuffix(s, suffix string) bool

// Count 子串不重叠的出现次数
func Count(s, substr string) int

// 根据分隔符切割
func Cut(s, sep string) (before, after string, found bool)
func CutPrefix(s, prefix string) (after string, found bool)
func CutSuffix(s, suffix string) (before string, found bool)

// Split 字符串分割
func Split(s, sep string) []string

// Fields 根据空白符切割字符串
func Fields(s string) []string

// Join 字符串数组用连接符连接
func Join(elems []string, sep string) string

// Repeat 字符串s拼接多次
func Repeat(s string, count int) string

// Replace 字符串替换，不重叠，n为次数，-1表示不限次数
func Replace(s, old, new string, n int) string

// ReplaceAll 字符串替换，不重叠
func ReplaceAll(s, old, new string) string

// ToLower 字母转小写
func ToLower(s string) string

// ToUpper 字母转大写
func ToUpper(s string) string

// TrimSpace 删除前后的空白字符
func TrimSpace(s string) string

// 删除前后的cutset包含的字符
func Trim(s, cutset string) string
func TrimLeft(s, cutset string) string
func TrimRight(s, cutset string) string

// 删除前缀/后缀
func TrimPrefix(s, prefix string) string
func TrimSuffix(s, suffix string) string
```

