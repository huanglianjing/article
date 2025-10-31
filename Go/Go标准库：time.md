# 1. 介绍

文档地址：https://pkg.go.dev/time

Go 标准库 time 用于测量和展示时间。

一个计算代码耗时的例子：

```go
start := time.Now()
// do sth.
end := time.Now()
elapsed := end.Sub(start)
fmt.Printf("%s\n", elapsed)
```

从已知格式字符串解析时间对象：

```go
str := "2024-02-13 15:30:00"
t, _ = time.Parse("2006-01-02 15:04:05", str)
```

格式化时间：

```go
t := time.Now()
fmt.Println(t.Format("2006-01-02 15:04:05-07:00"))
```

设置时间的时区：

```go
// 指定时区
loc, _ := time.LoadLocation("Asia/Tokyo")
t := time.Now().In(loc)

// UTC 时区
t = time.Now().UTC()
```

# 2. 常量

时间单位常量：

```go
const (
	Nanosecond  Duration = 1
	Microsecond          = 1000 * Nanosecond
	Millisecond          = 1000 * Microsecond
	Second               = 1000 * Millisecond
	Minute               = 60 * Second
	Hour                 = 60 * Minute
)
```

月份常量：

```go
const (
	January Month = 1 + iota
	February
	March
	April
	May
	June
	July
	August
	September
	October
	November
	December
)
```

星期常量：

```go
const (
	Sunday Weekday = iota
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)
```

# 3. 时间布局

当需要将时间转化为格式化的字符串时，需要根据规定在格式串填写对应的值，表示对应的格式化内容。

如下所示，如果想要展示年月日的字符串例如 2023-05-23，则格式串应该对应的是 2006-01-02，Go会在格式串中匹配2006识别为年份，然后替换为实际年份，01识别为月份，02识别为天，并逐个替换。

```
Year: "2006" "06"
Month: "Jan" "January" "01" "1"
Day of the week: "Mon" "Monday"
Day of the month: "2" "_2" "02"
Day of the year: "__2" "002"
Hour: "15" "3" "03" (PM or AM)
Minute: "4" "04"
Second: "5" "05"
AM/PM mark: "PM"
```

# 4. 类型

## 4.1 Duration

类型定义：

```go
// Duration 表示时间间隔
type Duration int64
```

方法：

```go
// ParseDuration 字符串表示的时间间隔转化为Duration对象
// 如 "300ms", "-1.5h" or "2h45m"
// 支持单位："ns", "us" (or "µs"), "ms", "s", "m", "h"
func ParseDuration(s string) (Duration, error)

// String Duration转化为字符串biao shi
func (d Duration) String() string

// Since 从一个时间到现在的Duration
func Since(t Time) Duration

// Until 现在到一个时间的Duration
func Until(t Time) Duration

// Abs 算Duration的绝对值
func (d Duration) Abs() Duration

// Duration转化为一个单位对应的数量
func (d Duration) Hours() float64
func (d Duration) Microseconds() int64
func (d Duration) Milliseconds() int64
func (d Duration) Minutes() float64
func (d Duration) Nanoseconds() int64
func (d Duration) Seconds() float64
```

## 4.2 Time

类型定义：

```go
// Time 时间，精度为纳秒
type Time struct {}
```

方法：

```go
// Date 指定时间变量初始化Time
func Date(year int, month Month, day, hour, min, sec, nsec int, loc *Location) Time
func (t Time) Date() (year int, month Month, day int)

// Unix 根据时间戳返回Time，sec为秒，msec为毫秒，usec为微秒，nsec为纳秒
func Unix(sec int64, nsec int64) Time
func UnixMicro(usec int64) Time
func UnixMilli(msec int64) Time

// Now 现在时间
func Now() Time

// Unix 获取时间戳
func (t Time) Unix() int64
func (t Time) UnixMilli() int64
func (t Time) UnixMicro() int64
func (t Time) UnixNano() int64

// Parse 根据layout表示的格式串，将value的内容转化为Time
func Parse(layout, value string) (Time, error)
func ParseInLocation(layout, value string, loc *Location) (Time, error)

// Format 根据layout格式串将Time转化为字符串
func (t Time) Format(layout string) string

// Add 加减运算
func (t Time) Add(d Duration) Time
func (t Time) AddDate(years int, months int, days int) Time
func (t Time) Sub(u Time) Duration

// 比较时间
func (t Time) Before(u Time) bool
func (t Time) After(u Time) bool
func (t Time) Compare(u Time) int
func (t Time) Equal(u Time) bool
func (t Time) IsZero() bool

// 获取不同单位的数量
func (t Time) Year() int
func (t Time) Month() Month
func (t Time) Weekday() Weekday
func (t Time) YearDay() int
func (t Time) Clock() (hour, min, sec int)
func (t Time) Day() int
func (t Time) Hour() int
func (t Time) Minute() int
func (t Time) Second() int
func (t Time) Nanosecond() int

// Local 设置为当前时区
func (t Time) Local() Time

// Location 获取所属的Location
func (t Time) Location() *Location
```

## 4.3 Location

类型定义：

```go
// Location 表示某个时区的时间
type Location struct{}
```

## 4.4 Month

类型定义：

```go
// Month 月
type Month int
```

方法：

```go
// String 返回月份代表的字符串，如1返回January
func (m Month) String() string
```

## 4.5 Weekend

类型定义：

```go
// Weekend 星期
type Weekday int
```

方法：

```go
// String 返回星期几字符串，如0返回Sunday
func (d Weekday) String() string
```

## 4.5 Ticker

类型定义：

```go
// Ticker 包含一个通道，发送固定间隔的时钟节拍
type Ticker struct {
	C <-chan Time
}
```

方法：

```go
// NewTicker 创建Ticker，设置节拍
func NewTicker(d Duration) *Ticker

// Reset 重新设置节拍，在时间间隔后重新发送节拍
func (t *Ticker) Reset(d Duration)

// Stop 停止发送节拍
func (t *Ticker) Stop()
```

## 4.6 Timer

类型定义：

```go
// Timer 包含一个通道，当Timer过期时发送时间
type Timer struct {
	C <-chan Time
}
```

方法：

```go
// NewTimer 创建Timer，设置过期时间
func NewTimer(d Duration) *Timer

// AfterFunc 创建Timer，设置过期时间，过期后新建协程执行函数
func AfterFunc(d Duration, f func()) *Timer

// Reset 重新设置过期时间
func (t *Timer) Reset(d Duration) bool

// Stop 停止计时
func (t *Timer) Stop() bool
```

# 5. 函数

```go
// Sleep 停止当前协程一段时间
func Sleep(d Duration)

// After 返回一个通道，等待一段时间后发送当前时间到通道
func After(d Duration) <-chan Time

// Tick 返回一个通道，定时发送当前时间到通道
func Tick(d Duration) <-chan Time
```

