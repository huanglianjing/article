# 1. 介绍

文档地址：https://pkg.go.dev/os

Go 标准库 os 提供了一些独立于平台的操作系统函数。

打开文件并读取内容：

```go
file, err := os.Open("file.go")
data := make([]byte, 100)
count, err := file.Read(data)
```

# 2. 常量

文件打开的权限选项：

```go
const (
	// Exactly one of O_RDONLY, O_WRONLY, or O_RDWR must be specified.
	O_RDONLY int = syscall.O_RDONLY // open the file read-only.
	O_WRONLY int = syscall.O_WRONLY // open the file write-only.
	O_RDWR   int = syscall.O_RDWR   // open the file read-write.
	// The remaining values may be or'ed in to control behavior.
	O_APPEND int = syscall.O_APPEND // append data to the file when writing.
	O_CREATE int = syscall.O_CREAT  // create a new file if none exists.
	O_EXCL   int = syscall.O_EXCL   // used with O_CREATE, file must not exist.
	O_SYNC   int = syscall.O_SYNC   // open for synchronous I/O.
	O_TRUNC  int = syscall.O_TRUNC  // truncate regular writable file when opened.
)
```

文件打开指针：

```go
const (
	SEEK_SET int = 0 // seek relative to the origin of the file
	SEEK_CUR int = 1 // seek relative to the current offset
	SEEK_END int = 2 // seek relative to the end
)
```

空文件，用于将不需要的内容输出：

```go
const DevNull = "/dev/null"
```

# 3. 变量

文件相关异常：

```go
var (
	// ErrInvalid indicates an invalid argument.
	// Methods on File will return this error when the receiver is nil.
	ErrInvalid = fs.ErrInvalid // "invalid argument"

	ErrPermission = fs.ErrPermission // "permission denied"
	ErrExist      = fs.ErrExist      // "file already exists"
	ErrNotExist   = fs.ErrNotExist   // "file does not exist"
	ErrClosed     = fs.ErrClosed     // "file already closed"

	ErrNoDeadline       = errNoDeadline()       // "file type does not support deadline"
	ErrDeadlineExceeded = errDeadlineExceeded() // "i/o timeout"
)
```

标准输入输出：

```go
var (
	Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
	Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
	Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

# 4. 类型

## 4.1 File

File 表示打开文件的文件描述符。

类型定义：

```go
type File struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Fd 返回文件描述符
func (f *File) Fd() uintptr

// Name 文件名
func (f *File) Name() string

// Stat 文件状态
func (f *File) Stat() (FileInfo, error)

// Read 读文件
func (f *File) Read(b []byte) (n int, err error)

// Write 写文件
func (f *File) Write(b []byte) (n int, err error)

// Close 关闭文件
func (f *File) Close() error
```

## 4.2 Process

Process 表示通过 StartProcess 创建的一个进程的信息。

类型定义：

```go
type Process struct {
	Pid int
	// contains filtered or unexported fields
}
```

方法：

```go
// FindProcess 通过pid查找进程
func FindProcess(pid int) (*Process, error)

// StartProcess 启动进程
func StartProcess(name string, argv []string, attr *ProcAttr) (*Process, error)

// Kill 终止进程
func (p *Process) Kill() error

// Signal 向进程发送信号
func (p *Process) Signal(sig Signal) error

```

## 4.3 Signal

Signal 表示操作系统对进程发送的信号。

类型定义：

```go
type Signal interface {
	String() string
	Signal() // to distinguish from other Stringers
}
```

# 5. 函数

```go
// Chdir 切换目录
func Chdir(dir string) error

// Getwd 获取绝对路径
func Getwd() (dir string, err error)

// Link 创建硬链接
func Link(oldname, newname string) error

// Symlink 创建软链接
func Symlink(oldname, newname string) error

// Mkdir 创建目录
func Mkdir(name string, perm FileMode) error

// MkdirAll 创建目录，多级目录不存在时直接创建
func MkdirAll(path string, perm FileMode) error

// Create 创建文件
func Create(name string) (*File, error)

// Open 打开文件
func Open(name string) (*File, error)


// Rename 重命名
func Rename(oldpath, newpath string) error

// Remove 删除文件或空目录
func Remove(name string) error

// RemoveAll 删除目录
func RemoveAll(path string) error

// ReadFile 读文件返回内容
func ReadFile(name string) ([]byte, error)

// WriteFile 写入文件
func WriteFile(name string, data []byte, perm FileMode) error

// Chmod 修改文件权限
func Chmod(name string, mode FileMode) error

// Chown 修改文件用户
func Chown(name string, uid, gid int) error

// Environ 返回所有环境变量，形式为 key=value
func Environ() []string

// Getenv 获取环境变量
func Getenv(key string) string

// LookupEnv 获取环境变量
func LookupEnv(key string) (string, bool)

// Setenv 设置环境变量
func Setenv(key, value string) error

// Getpid 获取进程id
func Getpid() int

// Getppid 获取父进程id
func Getppid() int

// Getuid 获取用户id
func Getuid() int

// Getgid 获取组id
func Getgid() int

// Exit 退出当前进程并返回返回码
func Exit(code int)
```

