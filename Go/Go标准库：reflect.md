# 1. 介绍

文档地址：https://pkg.go.dev/reflect

Go 标准库 reflect 用于运行时的映射，使程序运行时可以检查和操纵任意类型的对象。例如，在运行时获取一个任意类型 interface{} 的对象，并获取它的动态类型。

获取对象的类型并使用不同的取值方法：

```go
for _, v := range []any{"hi", 42, func() {}} {
	switch v := reflect.ValueOf(v); v.Kind() {
	case reflect.String:
		fmt.Println(v.String())
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		fmt.Println(v.Int())
	default:
		fmt.Printf("unhandled kind %s", v.Kind())
	}
}
```

获取变量的类型：

```go
var x int
typ := reflect.TypeOf(x)
fmt.Println(typ)
```

根据函数名调用对应函数：

```go
func add(a int, b int) int {
	return a + b
}
var FuncMap = map[string]interface{}{
	"add":   add,
}

funcName := "add"
f := reflect.ValueOf(FuncMap[funcName])
args := []reflect.Value{
	reflect.ValueOf(1),
	reflect.ValueOf(2),
}
ret := f.Call(args)
i := ret[0].Int()
```

根据方法名调用对应方法，需要注意方法需要是可导出方法才能被反射调用：

```go
type People struct {
	name string
}
func (p *People) SetName(s string) {
	p.name = s
}
func (p *People) GetName() string {
	return p.name
}

p := People{}
setFunc := reflect.ValueOf(&p).MethodByName("SetName")
args := []reflect.Value{reflect.ValueOf("moondo")}
setFunc.Call(args)
getFunc := reflect.ValueOf(&p).MethodByName("GetName")
name := getFunc.Call([]reflect.Value{})[0].String()
```

获取结构体成员的标签：

```go
type S struct {
	F string `species:"gopher" color:"blue"`
}

s := S{}
st := reflect.TypeOf(s)
field := st.Field(0)
fmt.Println(field.Tag.Get("color"), field.Tag.Get("species"))
```

# 2. 常量

```go
const Ptr = Pointer
```

# 3. 类型

## 3.1 Kind

类型定义：

```go
// Kind 类型
type Kind uint
const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Pointer
	Slice
	String
	Struct
	UnsafePointer
)
```

方法：

```go
// String 返回类型名称
func (k Kind) String() string
```

## 3.2 Type

类型定义：

```go
// Type 类型的接口
type Type interface {

	// Align returns the alignment in bytes of a value of
	// this type when allocated in memory.
	Align() int

	// FieldAlign returns the alignment in bytes of a value of
	// this type when used as a field in a struct.
	FieldAlign() int

	// Method returns the i'th method in the type's method set.
	// Methods are sorted in lexicographic order.
	Method(int) Method

	// MethodByName returns the method with that name in the type's
	// method set and a boolean indicating if the method was found.
	MethodByName(string) (Method, bool)

	// NumMethod returns the number of methods accessible using Method.
	NumMethod() int

	// Name returns the type's name within its package for a defined type.
	Name() string

	// PkgPath returns a defined type's package path.
	PkgPath() string

	// Size returns the number of bytes needed to store a value of the given type.
	Size() uintptr

	// String returns a string representation of the type.
	String() string

	// Kind returns the specific kind of this type.
	Kind() Kind

	// Implements reports whether the type implements the interface type u.
	Implements(u Type) bool

	// AssignableTo reports whether a value of the type is assignable to type u.
	AssignableTo(u Type) bool

	// ConvertibleTo reports whether a value of the type is convertible to type u.
	ConvertibleTo(u Type) bool

	// Comparable reports whether values of this type are comparable.
	Comparable() bool

	// Bits returns the size of the type in bits.
	Bits() int

	// ChanDir returns a channel type's direction.
	ChanDir() ChanDir

	// IsVariadic reports whether a function type's final input parameter
	// is a "..." parameter. If so, t.In(t.NumIn() - 1) returns the parameter's
	// implicit actual type []T.
	IsVariadic() bool

	// Elem returns a type's element type.
	Elem() Type

	// Field returns a struct type's i'th field.
	Field(i int) StructField

	// FieldByIndex returns the nested field corresponding to the index sequence.
	FieldByIndex(index []int) StructField

	// FieldByName returns the struct field with the given name
	// and a boolean indicating if the field was found.
	FieldByName(name string) (StructField, bool)

	// FieldByNameFunc returns the struct field with a name
	// that satisfies the match function and a boolean indicating if
	// the field was found.
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// In returns the type of a function type's i'th input parameter.
	In(i int) Type

	// Key returns a map type's key type.
	Key() Type

	// Len returns an array type's length.
	Len() int

	// NumField returns a struct type's field count.
	NumField() int

	// NumIn returns a function type's input parameter count.
	NumIn() int

	// NumOut returns a function type's output parameter count.
	NumOut() int

	// Out returns the type of a function type's i'th output parameter.
	Out(i int) Type
}
```

方法：

```go
// TypeOf 获取变量的类型
func TypeOf(i any) Type

// ArrayOf 给定长度和元素类型的数组的类型
func ArrayOf(length int, elem Type) Type

// ChanOf 给定方向和元素类型的通道的类型
func ChanOf(dir ChanDir, t Type) Type

// FuncOf 给定参数和返回类型的函数的类型
func FuncOf(in, out []Type, variadic bool) Type

// MapOf 给定key和元素类型的map的类型
func MapOf(key, elem Type) Type

// PointerTo 给定类型的指针的类型
func PointerTo(t Type) Type

// PtrTo 给定类型的指针的类型，等同PointerTo
func PtrTo(t Type) Type

// SliceOf 给定类型元素的slice的类型
func SliceOf(t Type) Type

// StructOf 给定成员类型的结构体的类型
func StructOf(fields []StructField) Type
```

## 3.3 Value

类型定义：

```go
// Value 变量
type Value struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Append 将多个变量x追加到slice s
func Append(s Value, x ...Value) Value

// AppendSlice 将slice t追加到slice s
func AppendSlice(s, t Value) Value

// Indirect 返回指针v指向的变量
func Indirect(v Value) Value

// MakeChan 创建不同的类型
func MakeChan(typ Type, buffer int) Value
func MakeFunc(typ Type, fn func(args []Value) (results []Value)) Value
func MakeMap(typ Type) Value
func MakeMapWithSize(typ Type, n int) Value
func MakeSlice(typ Type, len, cap int) Value

// New 创建指定类型的空指针
func New(typ Type) Value

// NewAt 创建指针并指向p
func NewAt(typ Type, p unsafe.Pointer) Value

// ValueOf 创建变量并用i初始化
func ValueOf(i any) Value

// Zero 创建指定类型变量并赋空值
func Zero(typ Type) Value

// Call 本身是一个函数，调用自己，传入in作为参数列表
func (v Value) Call(in []Value) []Value

// 还包括类型Value的许多方法，如判断能否转化为各种类型、设置各种类型的值、转为各种类型的值等
```

## 3.4 Method

类型定义：

```go
// Method 方法
type Method struct {
	Name string    // method name
	PkgPath string // 对非导出的方法是包路径，对可导出的方法为空
	Type  Type     // method type
	Func  Value    // func with receiver as first argument
	Index int      // index for Type.Method
}
```

方法：

```go
// IsExported 是否可导出（首字母大写）
func (m Method) IsExported() bool
```

## 3.5 StructField

类型定义：

```go
// StructField 结构体的某个成员
type StructField struct {
	Name string         // field name
	PkgPath string      // 对非导出的成员是包路径，对可导出的成员为空
	Type      Type      // field type
	Tag       StructTag // field tag string
	Offset    uintptr   // offset within struct, in bytes
	Index     []int     // index sequence for Type.FieldByIndex
	Anonymous bool      // is an embedded field
}
```

方法：

```go
// VisibleFields 返回可见的成员列表
func VisibleFields(t Type) []StructField

// IsExported 是否可导出（首字母大写）
func (f StructField) IsExported() bool
```

## 3.6 StructTag

类型定义：

```go
// StructTag 结构体成员的标签
type StructTag string
```

方法：

```go
// Get 获取标签对应key的值，key不存在返回空字符串
func (tag StructTag) Get(key string) string

// Lookup 获取标签对应key的值以及key是否存在
func (tag StructTag) Lookup(key string) (value string, ok bool)

```

## 3.7 MapIter

类型定义：

```go
// MapIter map遍历的迭代器
type MapIter struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// Key 返回迭代器当前位置的key
func (iter *MapIter) Key() Value

// Value 返回迭代器当前位置的value
func (iter *MapIter) Value() Value

// Next 迭代器指向下一个元素，返回false表示遍历到尽头了
func (iter *MapIter) Next() bool

// Reset 将迭代器指向另一个map
func (iter *MapIter) Reset(v Value)
```

## 3.8 其他类型

类型定义：

```go
// ChanDir 通道类型的方向
type ChanDir int
const (
	RecvDir ChanDir             = 1 << iota // <-chan
	SendDir                                 // chan<-
	BothDir = RecvDir | SendDir             // chan
)

// SelectCase select语句中的某个case
type SelectCase struct {
	Dir  SelectDir // direction of case
	Chan Value     // channel to use (for send or receive)
	Send Value     // value to send (for send)
}

// SelectDir 一个SelectCase的通信方向
type SelectDir int
const (
	SelectSend    SelectDir // case Chan <- Send
	SelectRecv              // case <-Chan:
	SelectDefault           // default
)

// SliceHeader 运行时的slice
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

// StringHeader 运行时的string
type StringHeader struct {
	Data uintptr
	Len  int
}
```

# 4. 函数

```go
// Copy 变量复制，两个参数必须是同样元素类型的slice或数组，返回复制的元素个数
func Copy(dst, src Value) int

// DeepEqual 比较参数是否深度相等，对于函数同样是nil才是深度相等
func DeepEqual(x, y any) bool

// Swapper 返回一个函数，执行这个函数将调换现在的参数slice中的对应索引的元素
func Swapper(slice any) func(i, j int)
```

