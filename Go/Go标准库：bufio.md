# 1. 介绍

文档地址：https://pkg.go.dev/bufio

Go 标准库 bufio 通过包装 io.Reader 和 io.Writer 实现了带缓冲的I/O。

输出内容到带缓冲的输出：

```go
w := bufio.NewWriter(os.Stdout)
fmt.Fprint(w, "Hello, ")
fmt.Fprint(w, "world!")
w.Flush()
```

从标准输入读取并输出到标准输出：

```go
scanner := bufio.NewScanner(os.Stdin)
for scanner.Scan() {
	fmt.Println(scanner.Text())
}
if err := scanner.Err(); err != nil {
	fmt.Fprintln(os.Stderr, "reading standard input:", err)
}
```

从字符串中输入，并计算单词数量：

```go
const input = "Now is the winter of our discontent,\nMade glorious summer by this sun of York.\n"
scanner := bufio.NewScanner(strings.NewReader(input))
scanner.Split(bufio.ScanWords)
count := 0
for scanner.Scan() {
	count++
}
if err := scanner.Err(); err != nil {
	fmt.Fprintln(os.Stderr, "reading input:", err)
}
fmt.Printf("%d\n", count)
```

# 2. 常量

```go
const (
	// MaxScanTokenSize is the maximum size used to buffer a token
	// unless the user provides an explicit buffer with Scanner.Buffer.
	// The actual maximum token size may be smaller as the buffer
	// may need to include, for instance, a newline.
	MaxScanTokenSize = 64 * 1024
)
```

# 3. 变量

读写异常：

```go
var (
	ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
	ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
	ErrBufferFull        = errors.New("bufio: buffer full")
	ErrNegativeCount     = errors.New("bufio: negative count")
)

var (
	ErrTooLong         = errors.New("bufio.Scanner: token too long")
	ErrNegativeAdvance = errors.New("bufio.Scanner: SplitFunc returns negative advance count")
	ErrAdvanceTooFar   = errors.New("bufio.Scanner: SplitFunc returns advance count beyond input")
	ErrBadReadCount    = errors.New("bufio.Scanner: Read returned impossible count")
)

var ErrFinalToken = errors.New("final token")
```

# 4. 类型

## 4.1 Reader

io.Reader 的带缓冲实现。

类型定义：

```go
type Reader struct {
	// contains filtered or unexported fields
}
```

函数：

```go
// 创建 Reader
func NewReader(rd io.Reader) *Reader
func NewReaderSize(rd io.Reader, size int) *Reader
```

方法：

```go
// Buffered 现在缓冲区可以读取的字节数
func (b *Reader) Buffered() int

// Size 缓冲区大小
func (b *Reader) Size() int

// Reset 清除缓冲区内容
func (b *Reader) Reset(r io.Reader)

// Discard 忽略接下来n个字节
func (b *Reader) Discard(n int) (discarded int, err error)

// Peek 获取接下来n个字节内容但不推进偏移量
func (b *Reader) Peek(n int) ([]byte, error)

// 读取内容
func (b *Reader) Read(p []byte) (n int, err error)
func (b *Reader) ReadByte() (byte, error)
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
func (b *Reader) ReadRune() (r rune, size int, err error)
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
func (b *Reader) ReadString(delim byte) (string, error)

// WriteTo 输出到 Writrer
func (b *Reader) WriteTo(w io.Writer) (n int64, err error)
```

## 4.2 Scanner

Scanner 提供了从换行划分的文本读取内容的方便方式。

类型定义：

```go
type Scanner struct {
	// contains filtered or unexported fields
}
```

函数：

```go
// 创建 Scanner
func NewScanner(r io.Reader) *Scanner
```

方法：

```go
// Buffer 设置缓冲区
func (s *Scanner) Buffer(buf []byte, max int)

// Split 设置分隔函数
func (s *Scanner) Split(split SplitFunc)

// Text 读取内容
func (s *Scanner) Text() string

// Bytes 读取内容
func (s *Scanner) Bytes() []byte

// Scan 是否还能够继续读取
func (s *Scanner) Scan() bool
```

## 4.3 Writer

io.Writer 的带缓冲实现。

类型定义：

```go
type Writer struct {
	// contains filtered or unexported fields
}
```

函数：

```go
// 创建 Writer
func NewWriter(w io.Writer) *Writer
func NewWriterSize(w io.Writer, size int) *Writer
```

方法：

```go
// Size 缓冲区大小
func (b *Writer) Size() int

// Buffered 已写入缓冲区的字节数
func (b *Writer) Buffered() int

// Available 缓冲区还可用字节数
func (b *Writer) Available() int

// AvailableBuffer 返回缓冲区可用大小的切片
func (b *Writer) AvailableBuffer() []byte

// Reset 清除缓冲区内容
func (b *Writer) Reset(w io.Writer)

// Flush 将缓冲区写入 Writer
func (b *Writer) Flush() error

// ReadFrom 从 Reader 读取
func (b *Writer) ReadFrom(r io.Reader) (n int64, err error)

// Write 写入缓冲区
func (b *Writer) Write(p []byte) (nn int, err error)
func (b *Writer) WriteByte(c byte) error
func (b *Writer) WriteRune(r rune) (size int, err error)
func (b *Writer) WriteString(s string) (int, error)
```

## 4.4 ReadWriter

保存了 Reader 和 Writer 指针的结构体。

类型定义：

```go
type ReadWriter struct {
	*Reader
	*Writer
}
```

函数：

```go
func NewReadWriter(r *Reader, w *Writer) *ReadWriter
```

# 5. 函数

```go
// 用于 Scanner，按字节读取
func ScanBytes(data []byte, atEOF bool) (advance int, token []byte, err error)

// 用于 Scanner，按行读取
func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)

// 用于 Scanner，按UTF-8字符读取
func ScanRunes(data []byte, atEOF bool) (advance int, token []byte, err error)

// 用于 Scanner，按单词读取
func ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error)
```

