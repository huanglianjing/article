# 1. 介绍

文档地址：https://pkg.go.dev/sync/atomic

Go 标准库 sync/atomic 用于提供低层级的同步原子操作。

# 2. 类型

包括的类型定义如下：

```go
// Bool
type Bool struct {
	// contains filtered or unexported fields
}

// Int32
type Int32 struct {
	// contains filtered or unexported fields
}

// Int64
type Int64 struct {
	// contains filtered or unexported fields
}

// Uint32
type Uint32 struct {
	// contains filtered or unexported fields
}

// Uint64
type Uint64 struct {
	// contains filtered or unexported fields
}

// Pointer
type Pointer[T any] struct {
	// contains filtered or unexported fields
}

// Uintptr
type Uintptr struct {
	// contains filtered or unexported fields
}

// Value
type Value struct {
	// contains filtered or unexported fields
}
```

该包中的每一个类型内部都保存了该类型对应的基础类型的值，并提供了如下几个原子操作的方法：

* Load：返回储存的值
* Store：设置为指定的值
* Add：增加指定值
* Swap：设置为指定值，并返回旧值
* CompareAndSwap：如果值等于参数 old，则设置为参数 new

其中 Pointer、Value 类型不包含方法 Add。

# 3. 函数

```go
// 增加变量并返回新的值，原子操作
func AddInt32(addr *int32, delta int32) (new int32)
func AddInt64(addr *int64, delta int64) (new int64)
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddUint64(addr *uint64, delta uint64) (new uint64)
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)

// 如果值等于 old，则设置为 new，原子操作
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)

// 获取指针的值，原子操作
func LoadInt32(addr *int32) (val int32)
func LoadInt64(addr *int64) (val int64)
func LoadUint32(addr *uint32) (val uint32)
func LoadUint64(addr *uint64) (val uint64)
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
func LoadUintptr(addr *uintptr) (val uintptr)

// 将值保存至指针，原子操作
func StoreInt32(addr *int32, val int32)
func StoreInt64(addr *int64, val int64)
func StoreUint32(addr *uint32, val uint32)
func StoreUint64(addr *uint64, val uint64)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
func StoreUintptr(addr *uintptr, val uintptr)

// 将指针的值保存为新的值并返回旧值，原子操作
func SwapInt32(addr *int32, new int32) (old int32)
func SwapInt64(addr *int64, new int64) (old int64)
func SwapUint32(addr *uint32, new uint32) (old uint32)
func SwapUint64(addr *uint64, new uint64) (old uint64)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)
```

