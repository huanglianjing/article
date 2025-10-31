# 1. 介绍

文档地址：https://pkg.go.dev/math

Go 标准库 math 提供了一些基础的数学常量与工具函数。

# 2. 常量

一些数学常量：

```go
const (
	E   = 2.71828182845904523536028747135266249775724709369995957496696763 // https://oeis.org/A001113
	Pi  = 3.14159265358979323846264338327950288419716939937510582097494459 // https://oeis.org/A000796
	Phi = 1.61803398874989484820458683436563811772030917980576286213544862 // https://oeis.org/A001622

	Sqrt2   = 1.41421356237309504880168872420969807856967187537694807317667974 // https://oeis.org/A002193
	SqrtE   = 1.64872127070012814684865078781416357165377610071014801157507931 // https://oeis.org/A019774
	SqrtPi  = 1.77245385090551602729816748334114518279754945612238712821380779 // https://oeis.org/A002161
	SqrtPhi = 1.27201964951406896425242246173749149171560804184009624861664038 // https://oeis.org/A139339

	Ln2    = 0.693147180559945309417232121458176568075500134360255254120680009 // https://oeis.org/A002162
	Log2E  = 1 / Ln2
	Ln10   = 2.30258509299404568401799145468436420760110148862877297603332790 // https://oeis.org/A002392
	Log10E = 1 / Ln10
)
```

整形的极限值：

```go
const (
	MaxInt    = 1<<(intSize-1) - 1  // MaxInt32 or MaxInt64 depending on intSize.
	MinInt    = -1 << (intSize - 1) // MinInt32 or MinInt64 depending on intSize.
	MaxInt8   = 1<<7 - 1            // 127
	MinInt8   = -1 << 7             // -128
	MaxInt16  = 1<<15 - 1           // 32767
	MinInt16  = -1 << 15            // -32768
	MaxInt32  = 1<<31 - 1           // 2147483647
	MinInt32  = -1 << 31            // -2147483648
	MaxInt64  = 1<<63 - 1           // 9223372036854775807
	MinInt64  = -1 << 63            // -9223372036854775808
	MaxUint   = 1<<intSize - 1      // MaxUint32 or MaxUint64 depending on intSize.
	MaxUint8  = 1<<8 - 1            // 255
	MaxUint16 = 1<<16 - 1           // 65535
	MaxUint32 = 1<<32 - 1           // 4294967295
	MaxUint64 = 1<<64 - 1           // 18446744073709551615
)
```

浮点数的极限值：

```go
const (
	MaxFloat32             = 0x1p127 * (1 + (1 - 0x1p-23)) // 3.40282346638528859811704183484516925440e+38
	SmallestNonzeroFloat32 = 0x1p-126 * 0x1p-23            // 1.401298464324817070923729583289916131280e-45

	MaxFloat64             = 0x1p1023 * (1 + (1 - 0x1p-52)) // 1.79769313486231570814527423731704356798070e+308
	SmallestNonzeroFloat64 = 0x1p-1022 * 0x1p-52            // 4.9406564584124654417656879286822137236505980e-324
)
```

# 3. 函数

```go
// Abs 绝对值
func Abs(x float64) float64

// Max 最大值
func Max(x, y float64) float64
// Min 最小值
func Min(x, y float64) float64

// Ceil 向上取整
func Ceil(x float64) float64
// Floor 向下取整
func Floor(x float64) float64
// Round 四舍五入取整
func Round(x float64) float64
// Trunc 返回整数部分
func Trunc(x float64) float64

// Mod 余数，x%y
func Mod(x, y float64) float64
// Pow x的y次方，x^y
func Pow(x, y float64) float64
// Pow10 10的n次方，10^n
func Pow10(n int) float64

// Sqrt 平方根，x^0.5
func Sqrt(x float64) float64

// Log 自然对数
func Log(x float64) float64
// Log10 10为底的对数
func Log10(x float64) float64
// Log2 2为底的对数
func Log2(x float64) float64

// Inf 根据参数返回正无限大、负无限大
func Inf(sign int) float64
// IsInf 判断是否正或负无限大
func IsInf(f float64, sign int) bool
// NaN 返回NaN
func NaN() float64
// IsNaN 判断是否NaN
func IsNaN(f float64) (is bool)

// 三角函数
func Sin(x float64) float64
func Cos(x float64) float64
func Tan(x float64) float64
```

