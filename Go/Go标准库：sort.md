# 1. 介绍

文档地址：https://pkg.go.dev/sort

Go 标准库 sort 用于进行数组的排序。

字符串数组排序：

```go
names := []string{"Tom", "Mike", "Will"}
sort.Strings(names)
```

通过实现接口 Interface 的方法 Len/Swap/Less，然后对结构体进行排序：

```go
type Person struct {
	Name string
	Age  int
}

type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
	people := []Person{
		{"Bob", 31},
		{"John", 42},
		{"Michael", 17},
		{"Jenny", 26},
	}
	sort.Sort(ByAge(people))
}
```

# 2. 类型

## 2.1 Interface

作为一些排序函数的对象必须实现 Interface 接口。

类型定义：

```go
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int

	// Less reports whether the element with index i
	// must sort before the element with index j.
	Less(i, j int) bool

	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

## 2.2 IntSlice

类型定义：

```go
type IntSlice []string
```

方法：

```go
func (x IntSlice) Len() int
func (x IntSlice) Less(i, j int) bool
func (p IntSlice) Search(x int) int
func (x IntSlice) Sort()
func (x IntSlice) Swap(i, j int)
```

## 2.3 Float64Slice

类型定义：

```go
type Float64Slice []float64
```

方法：

```go
func (x Float64Slice) Len() int
func (x Float64Slice) Less(i, j int) bool
func (p Float64Slice) Search(x float64) int
func (x Float64Slice) Sort()
func (x Float64Slice) Swap(i, j int)
```

## 2.4 StringSlice

类型定义：

```go
type StringSlice []string
```

方法：

```go
func (x StringSlice) Len() int
func (x StringSlice) Less(i, j int) bool
func (p StringSlice) Search(x string) int
func (x StringSlice) Sort()
func (x StringSlice) Swap(i, j int)
```

# 3. 函数

```go
// Slice 数组排序，指定小于函数
func Slice(x any, less func(i, j int) bool)

// Sort 数组排序
func Sort(data Interface)

// Stable 数组排序，排序确保稳定，原先相等的元素排序后不改变相对顺序
func Stable(data Interface)

// Reverse 数组倒序
func Reverse(data Interface) Interface

// IsSorted 是否有序
func IsSorted(data Interface) bool

// Ints int数组排序
func Ints(x []int)

// IntsAreSorted 判断数组是否递增
func IntsAreSorted(x []int) bool

// Float64s float64数组排序
func Float64s(x []float64)

// Float64sAreSorted 判断数组是否递增
func Float64sAreSorted(x []float64) bool

// Strings string数组排序
func Strings(x []string)

// StringsAreSorted 判断数组是否递增
func StringsAreSorted(x []string) bool

// Find 二分搜索查找，cmp(i)<=0 的最小下标
func Find(n int, cmp func(int) int) (i int, found bool)

// Search 二分搜索查找，f(i)==true 的最小下标
func Search(n int, f func(int) bool) int

// 查询数组中特定值
func SearchInts(a []int, x int) int
func SearchFloat64s(a []float64, x float64) int
func SearchStrings(a []string, x string) int
```

