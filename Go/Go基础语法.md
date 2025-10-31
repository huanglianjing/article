![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/go/go_logo.png)

# 1. 程序结构

## 1.1 名称

Go中函数、变量、常量、类型、语句标签、包的名称遵循规则：开头是一个字母或下划线，后面可以跟任意数量的字符、数字和下划线，并区分大小写。

对于语法中需要有变量名但程序逻辑用不到的，可以直接以一个下划线_表示空标识符。

**关键字**

Go有25个关键字，不能用于定义名称。

```
break     default      func    interface  select
case      defer        go      map        struct
chan      else         goto    package    switch
const     fallthrough  if      range      type
continue  for          import  return     var
```

**内置的预声明的常量、类型、函数**

```
常量
true false iota nil

类型
int int8 int16 int32 int64
uint uint8 uint16 uint32 uint64 uintptr
float32 float64 complex64 complex128
bool byte rune string error

函数
make len cap new append copy close delete
complex real imag
panic recover
```

## 1.2 声明

有四个主要的声明：变量（var）、常量（const）、类型（type）、函数（func）。

如果实体在函数中声明，它只在函数局部有效。如果声明在函数外，它将对包里面的所有源文件可见。如果名称以大写字母开头，就是导出的，对包外是可见可访问的。包名总是又小写字母组成。Go程序员通常使用驼峰式而不是下划线来命名，而一些如 HTML 或 ASCII的单词在命名中缩写通常使用相同的大小写。

Go程序存储在一个或多个.go后缀的文件里，每个文件以 package 声明开头，表明文件属于哪个包。后面是 import 声明。然后是包级别的类型、变量、常量、函数的声明，他们不区分顺序。

## 1.3 变量

Go 中的变量分类值类型和引用类型。值类型包括基础类型 int、float、string、bool 等和结构体、数组，储存值，分配存储在栈空间，函数调用结束后会释放。引用类型包括 slice、map、通道和指针，储存指向数据的地址，通常存储在堆空间，通过 GC 进行回收。

**变量声明**

变量声明的通用形式：创建一个具体类型的变量，定一个名字，并设置初始值。

```go
var name type = expression
```

省略类型，类型将由初始化表达式决定。

```go
var name = expression
```

省略表达式，初始值对应于类型的零值，对于数字是0，布尔值是false，字符串是""，对于指针、接口和引用类型是nil，对于数组或结构体这样的复合类型是所有元素或成员的零值。因此Go中不存在未初始化变量。

```go
var name type
```

批量声明，声明多个变量并分别指定类型吗，可以选择给它们赋值，不进行赋值则默认为该类型的零值。

```go
var (
	a string
	b int = 234
	c bool
	d float64
)
```

短变量声明，用来声明和初始化局部变量，变量类型由表达式的类型决定。在局部变量中主要使用短声明，var声明通常用于跟初始化表达式类型不一致的局部变量，或后面才赋值的情况。

```go
name := expression
```

多重声明，对多个变量分别声明和初始化。

```go
i, j := 0, 1
out, err := f() // 函数有多个返回值时
```

**变量初始化**

包级别的初始化在main开始之前进行，局部变量初始化在函数执行期间进行。

**变量赋值**

```go
name = expression
```

多重赋值，可以给两个或更多变量同时赋值。

```go
i, j = 0， 1
i, j = j, i // 交换i和j的值
```

递增递减，Go不支持前置的递增和递减。

```go
i++
i--
```

**指针**

指针存储变量的地址。

```go
i := 1
p := &i // 获取变量的地址
j := *p // 由指针获取变量的值
```

**new函数**

new函数创建一个未命名的某类型变量，初始化为零值，并返回其地址指针。

```go
p := new(int)
i := *p
```

**生命周期**

包级别变量的生命周期是整个程序的执行时间。

局部变量有一个动态的生命周期，每次执行声明语句时创建一个新的实体，直到生命周期终结，它占用的存储空间被回收。

**类型声明**

type声明为一个已有类型定义一个新的类型名字，然后可以使用新定义的类型来声明变量。

```go
type newTypeName oldTypeName
type newTypeName = oldTypeName

type Height float64
var h Height
```

## 1.4 包和文件

一个包的源代码保存在一个或多个.go文件中，它所在目录明的尾部就是包的导入路径。每一个包给它的声明提供独立的命名空间。

包中的变量如果是要设置为对外可见，首字母应该为大写字母。

在包级别，声明的顺序没有关系，一个声明可以引用它前面或者后面的声明，类型和函数还可以引用它自己。

**包初始化**

包的初始化从初始化包级别的变量开始，这些变量按照声明顺序初始化，如果包包含多个.go文件，初始化按编译器收到文件的顺序进行。如果有导入的包，按照依赖一个个地初始化每个包，先是被导入的包，再到导入别的包的包。

**作用域**

在不同语法块中声明的变量具有不同的作用域，包含所有全代码的语法块叫全局块。

当编译器遇到一个名字的引用时，将从当前语法块到全局块寻找其声明，使用先找到最内层的语法块中的变量，都找不到则报错。

## 1.5 注释

注释有单行注释和多行注释。

```go
// todo

/*
todo
*/
```

## 1.6 语句

**判断语句**

条件判断condition不需要加括号。

```go
if condition {
    // do sth
}

if condition {
    // do sth
} else {
    // do sth
}

if condition {
    // do sth
} else if {
    // do sth
} else {
    // do sth
}
```

条件判断中可以使用分号隔开两条语句，例如一个语句为调用函数，另一个语句为判断返回是否出错，从而接着作出处理。

```go
if f, err := os.Open("file"); err != nil {
	// error handle
}
```

**循环语句**

for是Go支持的唯一一种循环语句，条件不需要加括号，左大括号{必须和post语句在同一行。

init、condition、post三者都可以忽略。

```go
for init; condition; post {
    // do sth
}

for condition { // 省略init和post
    // do sth
}

for { // 省略三部分，无限循环
    // do sth
}
```

通过break、return语句退出循环或通过continue进入下一循环。

**switch语句**

判断x的值，从上至下匹配case的值，匹配就会对应后面的代码。

默认情况下case自带break，匹配成功后不会执行接下来的case。如果需要继续执行下一个case，使用fallthrough。

```go
switch x {
    case val1:
        // do sth
    case val2:
        // do sth
    default: // 可选
        // do sth
}

switch x {
    case val1, val2: // 多条件匹配
        // do sth
    default:
        // do sth
}

switch { // 无标签，等价于switch true，会判断每个case的布尔值
    case val1:
        // do sth
    case val2:
        // do sth
    default: // 可选
        // do sth
}
```

**select语句**

select 语句中，每个 case 必须是一个发送或接收的 IO 操作，即从通道读取或写入通道操作。

有任意个case可以进行操作时，会随机选一个执行。若没有一个case可操作，有default则执行，否则阻塞等待。

```go
select {
    case case1:
        // do sth
    case case2:
        // do sth
    default: // 可选
        // do sth
}
```

单独的 select 只会随机挑选一个满足的 case 进行操作，可以在外面套一层 for 来循环执行。

```go
for {
	select {
	case case1:
		// do sth
	case case2:
		// do sth
	default: // 可选
		// do sth
	}
}
```

如果 select 的所有 case 都陷入了休眠状态，如读取的通道设置为 nil，又没有 default 语句，Go 将检测到协程陷入了睡眠，所有协程都陷入睡眠将会使程序崩溃退出。

**goto语句**

设置一个标记，然后用goto转移到该行处运行。

会造成程序逻辑混乱，不建议使用。

```go
MARK: if true {
}

goto MARK
```

# 2. 类型

Go的数据类型分为四大类：基础数据（basic type）、聚合类型（aggregate type）、引用类型（reference type）、接口类型（interface type）。

- 基础类型：包括数字、字符串和布尔型。
- 聚合类型：数组和结构。
- 引用类型：包括指针、slice、map、函数和通道。
- 接口类型：定义一组方法的集合。

## 2.1 整数

有符号整数分为int8、int16、int32、int64，无符号整数分为uint8、uint16、uint32、uint64。还有类型int、uint，其大小根据平台决定，与原生的整数相同。

rune类型等同于int32，表示任何一个字符，支持英文和中文字符。

byte类型等同于uint8，表示一个字节。

uintptr大小不明确，但足够完整存放指针，仅仅用于底层编程。

**二元操作符**

二元操作符按优先级降序排序如下：

```
*   /   %   <<  >>  &   &^
+   -   |   ^
==  !=  <   <=  >   >=
&&
||
```

**一元操作符**

```
+  一元取正，无实际影响
-  一元取负
```

**进制**

整数正常为十进制。八进制以0开头，如0666。十六进制以0x或0X开头，如0x343aff。

## 2.2 浮点数

浮点数有float32和float64，遵从IEEE754标准。十进制下，float32的有效数字大于是6位，float64的有效数字大约是15位，大多数情况优先选用float64。

非常大或非常小的数字用科学计数法，如1.4e30、-4E-10。

NaN（Not a Number）表示数学上无意义的运算结果，math.IsNaN函数判断参数是否为非数值，math.NaN函数返回一个NaN，不应该直接用==和NaN比较，而应该用math.IsNaN函数来判断。

## 2.3 复数

复数有complex64和complex128，构成的实部和虚部分别为float32和float64。

```go
x := 1 + 2i                      // 定义复数

var y complex128 = complex(1, 2) // 创建复数
y1 := real(y)                    // 提取复数的实部
y2 := imag(y)                    // 提取复数的虚部
```

## 2.4 布尔值

布尔值为bool型，值只能是真（true）和假（false）。

布尔值不能隐式转换成数值，反之也不行，只能通过if语句判断再赋值。

## 2.5 字符串

字符串是不可变的字节序列。字符串内部数据不能修改，所以两个字符串嫩安全地共用同一段底层内存。

```go
s := "hello world" // 声明字符串
l := len(s)        // 返回字符串的字节数
c := s[1]          // 下标访问操作，访问超出范围将出发异常

sub := s[i:j]      // 子串，内容取自原字符串的字节，取从下标i到j-1的字节
sub := s[i:]       // 从下标i到最后
sub := s[:j]       // 从下标0到j-1
sub := s[:]        // 整个字符串

con := s + "new"   // 字符串连接
con += s           // 字符串连接
```

字符串的第i个字节不一定是第i个字符，因为非ASCII字符的编码有可能需要多个字节。Go的源文件总是按UTF-8编码，因此字符串会按UTF-8解读。

通过string函数可以将其他类型转换为字符串并返回。

```go
s := string(123) // 其他类型转换为字符串
```

Go 的字符串使用 UTF-8 编码，因此每个字符占用的字节是不一样的，如英文字符占一个字节，而中文字符占3个字节。因此有两种遍历字符串的方式：

```go
// 按照字符遍历字符串
for _, c := range s {
    fmt.Println(c)
}

// 按照字节遍历字符串
for i := 0; i < len(s); i++ {
    fmt.Println(s[i])
}
```

**字符串字面量**

字符串字面量用双引号包括，以反斜杠（\）为转义字符。十六进制转移字符为\xhh，h是十六进制数字且有两位。八进制转移字符为\ooo，o是八进制数字且有三位，而且不能超过\377。这两个进制转移字符都代表单个字节。

```go
s := "text\035\xff"
```

原生的字符串字面量用反引号包括，不用加转移字符，且可以跨多行。

```go
s := `line1
line2 word text
line3`
```

几个字符串操作相关的标准包：

- bytes：操作字节slice
- strings：字符串搜索、替换、比较、切分连接
- strconv：转换布尔值、整数、浮点数与对应的字符串
- unicode：判别文字符号值特性
- path、path/filepath：操作文件路径

**字符串和数字的转换**

整数转字符串

```go
x := 123
s := fmt.Sprintf("%d", x)
s := strconv.Itoa(x) // int
s := strconv.FormatInt(2435345, 10) // int64，10进制
```

字符串转整数

```go
s := "123"
x, err := strconv.Atoi(s)
y, err := strconv.ParseInt(s, 10, 64) // 10进制，最长64位
```

字符串中，英文字符数字和标点等半角字符占 1 个字节，而中文等其他编码的全角字符则占 3 个字节，通过 for range 是遍历一个个实际的字符，而通过下标索引则会按照字节一个个遍历。

## 2.6 常量

常量是一个表达式，在编译阶段就计算出值，有布尔型、字符串、数字三种。常量的数学运算、逻辑运算、内置函数依然是常量。

声明一个变量或同时声明多个变量：

```go
const pi = 3.14

const (
	pi = 3.14
	e = 2.718
)
```

**常量生成器iota**

iota用于创建一系列的相关值，而不是逐个写出，这种类型称为枚举型enum。

iota从0开始取值并逐项加1，可以通过iota的计算表达式表示不同的递增关系。

在中间插入一些_可以跳过某些值。

```go
// 从0开始递增加1
type Weekday int
const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)

// 指定起始值和递增值
const (
    a = iota*2 + 1
    b
    c
    d
)

// 从1开始左移1位，即乘以2
type Flag uint
const (
    Flag1 Flag = 1 << iota
    Flag2
    Flag3
    Flag4
)
```

**无类型常量**

无类型常量的数字精度更高，如math.Pi。

声明为变量或赋值给常量后会隐式转换成该变量类型。

# 3. 结构

## 3.1 数组

数组是具有固定长度且拥有零至多个相同数据类型元素的序列。定义数组时长度必须为常量，如果是变量需要定义为 slice。

定义和访问数组元素：

```go
var a [3]int // 定义数组
var t []int // 定义空数组，初始值为nil
t := []int{} // 定义空数组，当用于json编码时会是空数组而不是nil

a[0] // 访问下标
len(a) // 数组长度

// 遍历数组索引
for i := range a {
    fmt.Printf("%d %d\n", i, a[i])
}
// 遍历数组元素
for _, v := range a {
    fmt.Printf("%d\n", v)
}
// 遍历数组索引和元素
for i, v := range a {
    fmt.Printf("%d %d\n", i, v)
}
```

初始化数组：

```go
var a [3]int = [3]int{1, 2, 3} // 固定长度，若初始化的长度少于数组定义长度，后面下标设为零值
a = [3]int{0: 1, 2: 3} // 指定下标的初始化值，其他下标设为零值
a := [...]int{1, 2, 3} // 数组长度根据初始化的元素个数决定
```

多维数组：

```go
var a [5][3]int // 定义数组
a := [2][3]int{{1, 2, 3}, {4, 5, 6}} // 定义数组并赋值
b := [...][2]int{{1, 1}, {2, 2}, {3, 3}}  // 定义数组并赋值，只有第一个维度可以用...
```

数组的长度必须是常量表达式，在程序编译的时候就需要确定。不能给一个数组变量赋值不同长度的数组。

对下标的访问必须在数组的下标范围内，否则将会 panic。

函数调用的时候，数组参数是传值调用而不是引用调用，与C不同。

## 3.2 slice

slice是具有相同类型元素的可变长度的序列，通常写成[]T，T为元素类型，就像是没有长度的数组类型，slice的零值是nil。slice是轻量级的数据结构，可以用来访问数组的部分或全部元素，这个数组称为slice的底层数组。一个底层数组可以对应多个slice，这些slice可以引用数组的任何位置，元素也可以重叠。

slice有三个属性：指针、长度和容量。指针指向数组中第一个可以从slice访问的元素，这个元素不一定是数组的第一个元素。长度是slice中的元素个数，不能超过slice容量。容量通常是从slice的起始元素到底层数组最后一个元素之间的元素个数。

s[i:j]创建一个新的slice，s可以是数组、指向数组的指针或slice。i和j满足0 <= i <= j <= cap(s)，引用了序列s从i到j-1索引位置的所有元素。若省略i，则表示i=0，若省略j，则表示j=len(s)，同时省略ij则引用了整个序列。若j超过容量cap(s)则会导致程序崩溃 panic，若j只是超过长度len(s)则会比原slice长。

```go
sl := []int{1,2,3} // 创建切片

sl := s[1:5] // 从另一个数组或切片创建slice
// s[:high] 等同于 s[0:high]
// s[low:] 等同于 s[low:len(s)]
// s[:] 等同于 s[0:len(s)]
// s[low:high:max] 取切片，但是max限制了长度，这里low可以忽略

sl[1] // 引用元素
```

函数调用的时候，slice 参数可以在函数内部被修改底层数组的元素，因为 slice 本身是对底层数组的指向。而对于该 slice 参数进行 append 将不会有效，因为 slice 参数本身是值传递。

slice不能直接用==进行比较判断是否有相同的元素，通过bytes.Equal比较两个字节slice（[]byte），写函数来比较其他类型的slice。

**make函数**

make函数创建了一个无名数组，并返回它的一个slice。

```go
make([]T, len) // 创建slice
make([]T, len, cap) // 等同于make([]T, cap)[:len]
```

使用 make 函数创建多维切片，可以在创建时使用变量指定长度。

```go
length := 10
arr := make([][]int, length)
```

**append函数**

将元素追加到slice后面。

```go
sl = append(sl, 3)
```

调用append会检查slice是否有足够容量存储，如果slice容量足够，就会在同样底层数组定义一个新的slice，追加新元素，并返回新的slice。如果slice容量不够，就会创建足够容量的新的底层数组，将元素从旧的slice复制到这个数组，追加新元素并返回。创建新的底层数组往往会比实际需要更大，通常扩展一倍容量，减少数组扩展的内存分配次数。

可以将另一个数组的每个元素分别追加到 slice 后面，只需要在第二个参数后加上 ...。

```go
arr1 := []int64{1, 2, 3}
arr2 := []int64{4, 5}
arr1 = append(arr1, arr2...)
```

**copy函数**

将源slice数据逐个拷贝到目标slice，拷贝数量取两个slice现有元素长度最小值。

```go
copy(dst, src)
```

对下标的访问必须在 slice 本身的下标范围内，否则将会 panic。

## 3.3 map

map是键值对元素的无序集合，类型是map[K]V，K和V对应map的键和值的数据类型。键是唯一的，对应的值通过键来获取、更新、移除。

map的零值是nil，不能给零值map设置元素，需要先初始化。

创建map：

```go
m := make(map[string]int) // make函数创建
m := map[string]int{} // 创建空map
m := map[string]int{ // 初始化值
    "a": 23,
    "b": 7,
}
```

访问map：

```go
m["a"] // 下标访问，若不存在则返回值的零值

value, ok := m["a"]
if ok { // 检测下标是否存在
}

// 遍历map，顺序是随机的
for k := range m {
    fmt.Printf("%s:%d\n", k, m[k])
}
for k, v := range m { // 对 v 的修改将不会应用至 m，需要修改 m[k]
    fmt.Printf("%s:%d\n", k, v)
}

delete(m, "a") // 移除元素
```

直接定义一个 map 的默认值是 nil，设置元素将会 panic：

```go
m := map[int]int // m = nil
m[1] = 1 // panic
```

集合通常用 map[T]bool 或 map[T]struct{} 来实现。

## 3.4 结构体

结构体将零至多个任意类型的命名变量组合起来的聚合数据类型。

成员变量的名称首字母是大写的，则是可导出的，可以在外部访问到。结构体不能包含自身类型的成员，但可以包含自身类型的指针。

```go
// 定义结构体
type People struct {
    ID      int
    Name    string
    Address string
}

var pp People // 定义对象
pp := People{12345, "Mike"} // 未定义的成员设为零值
pp := People{ID: 12345, Name: "Mike"} // 指定设置的成员，代码维护性好

pp.Name // 访问成员
```

**结构体嵌套**

结构体嵌套，逐层访问成员。

```go
type Point struct {
    X, Y int
}
type Circle struct {
    Center Point
    Radius int
}
var c Circle
c.Point.X // 逐层访问成员
```

通过定义不带名称的结构体成员，也就是匿名成员，该成员必须是命名类型或指向命名类型的指针。可以直接访问到需要的变量而不需要指定中间变量。

```go
type Point struct {
    X, Y int
}
type Circle struct {
    Point // 不指定成员名称
    Radius int
}
var c Circle
c.X // 直接访问成员
```

匿名成员拥有隐式的名字，所以不能在一个结构体里定义两个相同类型的匿名成员，否则会引起冲突。

**标签**

结构体成员可以定义标签（tag），用于 JSON 序列化和反序列化、GORM 数据库表字段映射等作用。

```go
type People struct {
    ID      int    `json:"id" gorm:"column:id"`
    Name    string `json:"name" gorm:"column:name"`
    Address string `json:"address" gorm:"column:address"`
}
```

## 3.5 JSON

把Go数据结构转换为JSON称为marshal，生成一个字节slice。

```go
data, err := json.Marshal(st)
data, err := json.MarshalIndent(st, "", "    ") // 整齐格式化
```

使用json.Unmarshal把字节slice解码为单个JSON实体。

# 4. 函数、方法和接口

## 4.1 函数

函数声明包括一个名字、一个形参列表、一个可选的返回列表以及函数体。当函数返回一个未命名的返回值或没有返回值时，返回列表的圆括号可以忽略。函数的左大括号{符号必须和关键字func在同一行。

函数类型的零值是nil。

```go
func name(param-list) (result-list) {
    body
}
```

函数的类型称作函数签名，当两个函数有相同的形参列表和返回列表时，它们的类型或签名是相同的，这和形参和返回值的名字无关。如果函数的返回列表写了名字，则在调用函数时会被初始化为零值，不需要在函数中定义，可以提高代码可读性。

在 Go 中，所有函数实参都是按值传递的，函数接收到的是每个实参的副本，在函数内修改参数的值是对副本的修改。

但是有些数据类型是属于引用类型，如指针、slice、map、函数或通道，这些类型的参数传递也是值传递，但值本身是对数据的引用，函数内修改实参变量将会改变实际变量的内容。需要注意，对于 slice 内部保存的是指向底层数组的指针，虽然可以通过函数参数修改内容，但是 append 会造成底层数组重新分配从而不会生效。

想要实际修改作为参数传递的变量的值，可以传它的指针作为参数。

```go
func modify(i *int) {
	*i = 10
}
i := 1
modify(&i)
```

函数可以返回不止一个结果，比如返回一个计算结果和一个错误值。

```go
func f(i int) {
    return i, nil
}
ret, err := f(10)
ret, _ := f(11) // 忽略其中一个返回值
```

error是内置接口类型，一个错误返回是空值意味着成功，非空值意味着失败，有一个错误消息字符串。

```go
fmt.Println(err) // 输出错误消息
fmt.Printf("%v", err)
```

**匿名函数**

匿名函数在func关键字后面没有函数名称，在需要使用的时候才定义。

```go
strings.Map(func(r rune) rune { return r + 1 }, "qwerty123")

defer func(param-list) {
    // do sth
}(variables)
```

**变长函数**

变长函数有可变的参数个数，参数列表最后的类型名称之前用...声明，调用函数时可以传递任意数目的参数。在函数体内，可变参数是一个该类型的slice。

```go
func sum(vals ...int) int {
    body
}
```

**延迟函数调用**

在函数或方法调用前加defer，在执行return语句或者函数执行完毕，或异常情况如发生宕机时，会执行defer语句的函数调用。

```go
// 执行指定函数
defer mu.Unlock()

// 执行一段代码，使用匿名函数
defer func() {
    // do sth
}()
```

defer语句没有使用次数限制，执行的时候以调用defer语句顺序的倒序进行。

**宕机**

内置函数panic引起异常并退出。

如果触发了 panic，仍然会遍历本函数调用的 defer 链表，并依次执行。如果其中有调用 recover，则会停止 panic，否则在执行完本函数的 defer 链表后，向上一层函数抛出 panic，直至当前协程的最上层。当其中一个协程的 panic 未处理，会向 stderr 抛出 panic 信息，引发整个程序的崩溃退出。

```go
panic("x is nil")
```

**恢复**

内置函数recover可以捕获代码调用的panic或程序执行时触发的panic，使panic不会发生，并返回panic的内容，如果没有触发panic则返回nil。

通常在延迟函数defer内部调用，函数发生宕机时recover会终止当前的宕机状态并且返回宕机的值。函数不会从宕机的地方继续运行，而是正常返回。

```go
func Parse(input string) (s *Syntax, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("internal error: %v", p)
        }
    }()
    // do sth
}
```

可以将一个函数的 panic 转换为 error，避免单个函数执行的错误引起整个程序的崩溃。

```go
// PanicToError Panic转换为error
func PanicToError(f func()) (err error) {
	defer func() {
		if e := recover(); e != nil {
			err = fmt.Errorf(PanicTrace(e))
		}
	}()
	f()
	return
}

// PanicTrace panic调用链跟踪
func PanicTrace(err interface{}) string {
	stackBuf := make([]byte, 4096)
	n := runtime.Stack(stackBuf, false)
	return fmt.Sprintf("panic: %v %s", err, stackBuf[:n])
}
```

## 4.2 方法

在函数名字前面多加一个参数，就变成了方法，方法的声明把函数绑定到某个类型上。

```go
type Poing struct{ X, Y float64 } // 定义类型
func (p Point) f(q Point) float64 { // 定义方法
    return q.X - p.X
}
p.f() // 对象调用方法
```

方法名字前的参数p称为方法的接收者（receiver），表达式p.f称作选择字（selector），即为接收者p选择合适的f方法。对一个类型拥有的所有方法名必须唯一。

如果要设置方法不能通过对象访问到，则称作封装的方法，方法名称首字母为大写则可以访问到，否则不能由对象访问到，只能通过结构体的其他方法访问。

如果方法需要更新结构体中的成员值，将接收者`p Point`改为指针类型`p *Point`，将避免复制整个接收者。但是不允许本身是指针的类型进行方法声明。

```go
type Poing struct{ X, Y float64 } // 定义类型
func (p *Point) f(q Point) float64 { // 定义方法
    return q.X - p.X
}
(*p).f() // 对象指针调用方法
```

如果接收者p是Point类型变量，但方法要求*Point接收者，可以使用简写p.f()，编译器会隐式转换为&p。

可以将选择子p.f赋予一个方法类型的变量，它是一个函数。调用时不需要提供接收者，只需要提供实参。

```go
varP := p.f // 定义一个方法变量
varP(q) // 调用方法变量
```

## 4.3 接口

接口是一种抽象类型，仅仅提供一些方法，没有具体的数据和内部结构。

一个接口类型定义了一套方法，具体类型要实现该接口，必须实现接口类型定义的所有方法。只要结构体实现了接口定义的方法，就算是实现了该接口，不需要指定说明实现的是什么接口。

```go
// 定义接口
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}

// 组合已有接口，也可以在其中定义新的方法
type ReaderCloser interface {
    Reader
    Closer
}
```

类型实现接口示例：

```go
// Person 接口，定义了方法getName
type Person interface {
    getName() string
}

// Student 结构体，实现了接口Person
type Student struct {
    name string
    age int
}
func (s Student) getName() string {
    return s.name
}

// 声明一个Student对象为Person接口类型
var p Person = &Student{name:"Tom",age:20}
name := p.getName()
```

**空接口**

空接口没有定义任何方法，因此任何类型都实现了空接口，所以空接口类型的变量可以储存任何类型的变量。

空接口可以作为函数参数，用以接受任意类型的参数，也可用于作为map的key类型，实现保存任意类型值的map。

```go
var x interface{}
```

**类型断言**

类型断言是作用在接口值的操作，如x.(T)，x是一个接口类型的表达式，T是一个类型（称为断言类型），会检查作为操作数的动态类型是否满足指定的断言类型。

```go
var x interface{}
x = "hello"
var, ok := x.(string) // ok 为 true 表示类型断言成功
```



# 5. 协程和通道

## 5.1 协程

Go 中每一个并发执行的活动称为 goroutine，又称为协程。当一个程序启动时，有一个主 goroutine 来调用 main 函数，通过go语句可以创建新的 goroutine，不同的 goroutine 将会分别独立运行。

```go
f() // 调用函数，等待返回
go f() // 新建调用f()的goroutine，不用等待
```

当 main 函数返回时，所有 goroutine 都暴力地直接终结，然后程序退出。

如果任意一个协程发生 panic 时，将会触发整个程序的终结退出，可以在这个协程开始时使用 defer 调用一个匿名函数，在其中调用 recover 函数来避免 panic。

Go 中的结构体不是协程安全的，如果在多个协程中并发写将会触发 panic，可以通过 sync 包的锁将它们保护起来，也可以使用一些协程安全的实现，如使用 sync.Map 代替 map。

## 5.2 通道

通道又称为channel，是可以让一个goroutine发送特定值到另一个goroutine的通信机制，每个通道时一个具体类型的导管，叫做通道的元素类型，如一个int类型元素的通道写为chan int。

```go
ch := make(chan int) // 创建通道，类型是chan int，元素类型是int
```

通道是使用make创建的数据结构的引用，当复制或者作为参数传递给函数时，复制的是引用。

通道的零值是nil，对nil通道发送和接收将永远阻塞。同种类型的通道可以使用==进行比较。

通道的两个主要操作：发送（send）和接受（receive），统称为通信。send语句从一个goroutine传输一个值到另一个在执行接收表达式的goroutine。

```go
ch <- x  // 发送语句

x = <-ch     // 接收语句，赋值
x, ok = <-ch // 接收语句，判断是否成功接受
<-ch         // 接收语句，丢弃结果
```

通道还有一个操作：关闭（close）。关闭后发送操作将导致宕机，在已关闭的通道进行接收操作将获取所有已发送的值直至通道为空，然后可以一直获取通道元素类型对应的零值。可以多接收一个 ok 变量来判断，是成功获取到元素，还是已经读光了元素，然后再将通道变量设置为 nil，读取就会阻塞，select 语句就不会走到对应条件了。

```go
close(ch) // 关闭通道
```

**缓冲**

默认创建的是无缓冲通道，也称为同步通道。发送操作会阻塞，直至另一个goroutine对通道执行接收操作，接收操作也会阻塞，直至另一个goroutine对通道发送一个值。

缓冲通道有一个元素队列，队列的最大长度是通道的容量，在创建通道时指定。发送操作向队列尾部插入元素，若通道满了则阻塞，接收操作从通道头部移除一个元素，若通道空了则阻塞。

```go
ch = make(chan int)     // 无缓冲通道
ch = make(chan int, 0) // 无缓冲通道
ch = make(chan int, 3) // 容量为3的缓冲通道

cap(ch) // 获取通道容量
len(ch) // 获取通道当前元素个数
```

无缓冲通道提供强同步保障，发送和接收同步，缓冲通道将发送和接收解耦。当发送快于接收和接收快于发送，缓冲区保持满或者空，缓冲通道都是没有价值的。

有缓冲区的通道成为异步通道，创建时需要指定缓冲区的容量，当缓冲区为空时，读操作将会阻塞至写入元素，当缓冲区满时，写操作将会阻塞至读取元素。

**遍历通道**

有两种方式遍历一个通道读取内容。第一种是使用 for range 语句，遍历所有通道中的元素。对于一个打开的通道，遍历完后会阻塞，等待新的元素写入通道。对于关闭的通道，会遍历完所有缓冲区的元素，然后退出 for 循环。

```go
for i := range c {
}
```

另一种是使用 select 语句来读取管道。对于一个打开的通道，遍历完后会阻塞，等待新的元素写入通道。对于关闭的通道，会先遍历完所有缓冲区的元素，返回第二个 ok 变量是 true，然后继续返回元素类型的零值，ok 变量是 false。使用 ok 变量判断关闭的管道已经获取所有元素后，可以将管道变量设置为 nil，select 语句就不会再走到这个 case 了。

```go
for {
	select {
	case i := <-c:
	}
}

for {
	select {
	case i, ok := <-c:
	}
}
```

## 5.3 协程与线程

本节对比协程与线程的区别。

**栈内存**

每个线程有一个固定大小的栈内存，通常为2MB，用于保存函数调用期间的函数局部变量。

Go程序可以创建很多goroutine，因此一个goroutine在生命周期开始时有一个很小的栈，如2KB，但不是固定大小的，也用于保存函数调用的局部变量。goroutine的栈大小限制可以达到1GB，远大于线程的栈大小。

**调度**

线程由操作系统内核调度，每隔几毫秒发送硬件时钟中断到CPU，调用调度器，暂停当前线程并选择下一个线程恢复执行。控制权限在线程间需要完整的上下文切换。

Go运行时包含一个自己的调度器，使用m:n调度技术，调度m个goroutine到n个线程。Go的调度器由Go语言结构触发，不需要切换到内核语境，调度一个goroutine比调度线程成本低很多。

参数GOMAXPROCS确定使用多少个线程执行Go代码，默认是机器的CPU数量。

**标识**

线程有独特的标识，而goroutine没有可供程序员访问的标识。

# 6. 包

## 6.1 包

Go代码使用包来组织，一个包由一至多个.go源文件组成，放在一个文件夹中，文件夹名字描述了包的作用。

每个Go源文件的开头都要进行包声明，当该包被其他包引入时作为包名。当包定义一条命令，是可执行的Go程序，包名是main，main包中的main函数是程序开始执行的地方。

每个包有一个导入路径，是唯一的字符串来标识，包名是导入路径的最后一段。

```go
package test
```

在同一个目录下的所有 Go 源文件必须使用同一个包名，但不是必须要和目录名称相同，可以在不同目录的源文件使用同一个包名。

## 6.2 导入

在package声明后紧接着包含零至多个import声明。

```go
// 分别导入
import "fmt"
import "os"

// 合并导入
import (
    "fmt"
    "os"
)
```

习惯上导入包用空行分组，将标准库、第三方包、自身的包分隔开来，每一组按字母排序。

相同名字的包通过重命名导入避免冲突，重命名也可以将冗长的包名简化缩写，调用的时候使用重命名的包名。

```go
import (
    "crypto/rand"
    mrand "math/rand" // 重命名导入
)
```

导入的包没有在文件引用会产生编译错误，通过重命名导入，用替代的名字_表示空白导入。还有一种情况是，我们不直接调用该包的函数，但是需要执行包的 init 函数进行一些初始化工作，也可以通过空白导入的方式来完成。

```go
import _ "image/png"
```

点操作，导入包后，使用它的函数可以省略包名。

```go
import . "fmt"

func test() {
	Println("hello") // 不用使用包名也可以调用fmt的函数
}
```

Go 还可以导入 git 上的远程包。

首先通过 go get 命令下载包到 GOPATH下：

```bash
go get github.com/a/b
```

然后就可以在 import 语句中使用这个包：

```go
import "github.com/a/b"
```

# 7. 参考

- [《Go程序设计语言》](https://book.douban.com/subject/27044219/)

