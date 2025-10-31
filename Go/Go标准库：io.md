# 1. 介绍

文档地址：https://pkg.go.dev/io

Go 标准库 io 提供了基本的I/O功能。

从字符串 Reader 读取并复制到标准输出：

```go
r := strings.NewReader("some io.Reader stream to be read\n")
_, err := io.Copy(os.Stdout, r)
```

从字符串 Reader 读取并带缓冲地复制到标准输出，两次不需要重复申请额外内存：

```
r1 := strings.NewReader("first reader\n")
r2 := strings.NewReader("second reader\n")
buf := make([]byte, 8)

// buf is used here...
if _, err := io.CopyBuffer(os.Stdout, r1, buf); err != nil {
	log.Fatal(err)
}

// ... reused here also. No need to allocate an extra buffer.
if _, err := io.CopyBuffer(os.Stdout, r2, buf); err != nil {
	log.Fatal(err)
}
```

创建一个管道，并通过管道进行数据传输：

```go
r, w := io.Pipe()

go func() {
	fmt.Fprint(w, "some io.Reader stream to be read\n")
	w.Close()
}()

if _, err := io.Copy(os.Stdout, r); err != nil {
	log.Fatal(err)
}
```

# 2. 常量

文件打开指针：

```go
const (
	SeekStart   = 0 // seek relative to the origin of the file
	SeekCurrent = 1 // seek relative to the current offset
	SeekEnd     = 2 // seek relative to the end
)
```

# 3. 变量

读写时的异常：

```go
var EOF = errors.New("EOF")
var ErrClosedPipe = errors.New("io: read/write on closed pipe")
var ErrNoProgress = errors.New("multiple Read calls return no data or error")
var ErrShortBuffer = errors.New("short buffer")
var ErrShortWrite = errors.New("short write")
var ErrUnexpectedEOF = errors.New("unexpected EOF")
```

# 4. 类型

## 4.1 Reader

类型定义：

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

## 4.2 Writer

类型定义：

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

## 4.3 PipeReader

类型定义：

```go
type PipeReader struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Read 读取
func (r *PipeReader) Read(data []byte) (n int, err error)

// 关闭
func (r *PipeReader) Close() error
func (r *PipeReader) CloseWithError(err error) error
```

## 4.4 PipeWriter

类型定义：

```go
type PipeWriter struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Write 写入
func (w *PipeWriter) Write(data []byte) (n int, err error)

// 关闭
func (w *PipeWriter) Close() error
func (w *PipeWriter) CloseWithError(err error) error
```

# 5. 函数

```go
// Pipe 创建同步内存管道
func Pipe() (*PipeReader, *PipeWriter)

// 读取 Reader
func ReadAll(r Reader) ([]byte, error)
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)
func ReadFull(r Reader, buf []byte) (n int, err error)

// 写入 Writer
func WriteString(w Writer, s string) (n int, err error)

// 从 Reader 复制到 Writer
func Copy(dst Writer, src Reader) (written int64, err error)
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error)
func CopyN(dst Writer, src Reader, n int64) (written int64, err error)
```

