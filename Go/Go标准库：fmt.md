# 1. 介绍

文档地址：https://pkg.go.dev/fmt

Go 标准库之 fmt 包实现了格式化 IO，也就是类似于 C 语言的 printf 和 scanf 方式的格式化。

控制格式化的字符串中，以百分号（%）作为格式的通配符，后面跟着不同的字母表示不同的类型格式。例如 %d 表示10进制整数，于是有以下例子：

```go
import "fmt"

func fmtTest() {
	i := 23
    str := "good"
    fmt.Printf("%d:%s\n", i, str)
}

/* 输出结果
23:good
*/
```

%d 表示一个10进制整数，而 fmt.Printf() 方法在它的首个格式化字符串参数中找到它后，就从后续的参数里取到变量 i，将变量 i 的值输出到 %d 所占的位置。如果格式化字符串中包含多个以 % 表示的格式，就会依次从后面的参数中取参数值以格式所对应的类型，输出到那个位置。

# 2. 格式

通用格式：

```
%v 任何值的默认格式，对于结构体只会显示每个成员的值
%+v 对于结构体还会显示每个成员的字段名称
%#v Go语法的值，还会显示结构体的类型名

%T 值的类型

%% 表示百分号%
```

使用 %v 对于一些复合结构所打印的格式如下：

```
struct:             {field0 field1 ...}
array, slice:       [elem0 elem1 ...]
maps:               map[key1:value1 key2:value2 ...]
pointer to above:   &{}, &[], &map[]
```

布尔值：

```
%t 布尔值
```

整数：

```
%b 2进制整数
%d 10进制整数
%o 8进制整数
%O 8进制整数，带有0o前缀
%x 16进制整数，使用字母a-f
%X 16进制整数，使用字母A-F

%c 值对应的unicode字符
%q 值对应的unicode字符，带单引号
%U unicode格式
```

浮点数：

```
%f 浮点数，默认宽度和精度

%9f 宽度为9，默认精度
%.2f 默认宽度，精度为2
%9.2 宽度为9，精度为2
%9. 宽度为9，精度为0

%e 科学技术法，使用e
%E 科学技术法，使用E
%g 普通用%f，指数大用%e
%G 普通用%f，指数大用%E
%x 16进制，使用字母a-f
%X 16进制，使用字母A-F
```

字符串：

```
%s 字符串

%q 带有双引号
%x 16进制，使用字母a-f
%X 16进制，使用字母A-F
```

slice：

```
%p 第0号元素的16进制地址，以0x作前缀
```

指针：

```
%p 16进制，以0x作前缀
```

除了在 % 和字母中间加精度和宽度，还可以在中间加其它的一些标记：

```
+ 对于整数和浮点数，总是显示正号或者负号
- 在右边加空格
# 对于不同格式，在前面加0b、0x、0
空格 就算数字为整数不显示正号，也会在前面显示一个空格
0 用0代替开头的空格
```

# 3. 函数

## 3.1 异常

**func Errorf**

```go
func Errorf(format string, a ...interface{}) error
```

格式化字符串，返回一个error对象。

## 3.2 输出

**func Fprint**

```go
func Fprint(w io.Writer, a ...interface{}) (n int, err error)
```

将对象写到IO中，返回写入的字节数。

**func Fprintf**

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
```

格式化字符串，写到IO中，返回写入的字节数。

**func Fprintln**

```go
func Fprintln(w io.Writer, a ...interface{}) (n int, err error)
```

将对象写到IO中，然后换行，返回写入的字节数。

**func Print**

```go
func Print(a ...interface{}) (n int, err error)
```

将对象写到标准输出。

**func Printf**

```go
func Printf(format string, a ...interface{}) (n int, err error)
```

按格式将对象写到标准输出。

**func Println**

```go
func Println(a ...interface{}) (n int, err error)
```

将对象写到标准输出，然后换行。

**func Sprint**

```go
func Sprint(a ...interface{}) string
```

将对象写到字符串返回。

**func Sprintf**

```go
func Sprintf(format string, a ...interface{}) string
```

将对象按格式写到字符串返回。

**func Sprintln**

```go
func Sprintln(a ...interface{}) string
```

将对象写到字符串加上换行返回。

## 3.3 输入

**func Fscan**

```go
func Fscan(r io.Reader, a ...interface{}) (n int, err error)
```

从IO读取写到对象，返回读取的字节数。

**func Fscanf**

```go
func Fscanf(r io.Reader, format string, a ...interface{}) (n int, err error)
```

从IO读取写到对象，指定格式，返回读取的字节数。

**func Fscanln**

```go
func Fscanln(r io.Reader, a ...interface{}) (n int, err error)
```

从IO读取写到对象，读到换行时停止，返回读取的字节数。

**func Scan**

```go
func Scan(a ...interface{}) (n int, err error)
```

从标准输入读对象。

**func Scanln**

```go
func Scanln(a ...interface{}) (n int, err error)
```

从标准输入读对象，读到换行时停止。

**func Sscan**

```go
func Sscan(str string, a ...interface{}) (n int, err error)
```

从字符串读取对象。

**func Sscanf**

```go
func Sscanf(str string, format string, a ...interface{}) (n int, err error)
```

从字符串按照格式读取对象。

**func Sscanln**

```go
func Sscanln(str string, a ...interface{}) (n int, err error)
```

从字符串读取对象，到换行时停止。

