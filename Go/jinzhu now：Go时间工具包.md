# 1. 简介

github仓库地址：https://github.com/jinzhu/now

文档地址：https://pkg.go.dev/github.com/jinzhu/now

jinzhu/now 是 Go 语言的时间工具包。

# 2. 使用

## 2.1 安装

使用 go get 将 jinzhu/now 包下载到 GOPATH 指定的目录下。

```go
go get -u github.com/jinzhu/now
```

## 2.2 时间计算

根据当前时间，可以很方便地获取如当前分钟的开始、当前一天的开始、当前一个月的最后时间等。

```go
import "github.com/jinzhu/now"

time.Now() // 2013-11-18 17:51:49.123456789 Mon

now.BeginningOfMinute()        // 2013-11-18 17:51:00 Mon
now.BeginningOfHour()          // 2013-11-18 17:00:00 Mon
now.BeginningOfDay()           // 2013-11-18 00:00:00 Mon
now.BeginningOfWeek()          // 2013-11-17 00:00:00 Sun
now.BeginningOfMonth()         // 2013-11-01 00:00:00 Fri
now.BeginningOfQuarter()       // 2013-10-01 00:00:00 Tue
now.BeginningOfYear()          // 2013-01-01 00:00:00 Tue

now.EndOfMinute()              // 2013-11-18 17:51:59.999999999 Mon
now.EndOfHour()                // 2013-11-18 17:59:59.999999999 Mon
now.EndOfDay()                 // 2013-11-18 23:59:59.999999999 Mon
now.EndOfWeek()                // 2013-11-23 23:59:59.999999999 Sat
now.EndOfMonth()               // 2013-11-30 23:59:59.999999999 Sat
now.EndOfQuarter()             // 2013-12-31 23:59:59.999999999 Tue
now.EndOfYear()                // 2013-12-31 23:59:59.999999999 Tue

now.WeekStartDay = time.Monday // Set Monday as first day, default is Sunday
now.EndOfWeek()                // 2013-11-24 23:59:59.999999999 Sun
```

或者根据给定时间，计算对应的时间，实现的方法同上。

```go
t := time.Date(2013, 02, 18, 17, 51, 49, 123456789, time.Now().Location())
now.With(t).EndOfMonth()   // 2013-02-28 23:59:59.999999999 Thu
```

计算下一个星期几。

```go
now.Monday()              // 2013-11-18 00:00:00 Mon
now.Monday("17:44")       // 2013-11-18 17:44:00 Mon
now.Sunday()              // 2013-11-24 00:00:00 Sun (Next Sunday)
now.Sunday("18:19:24")    // 2013-11-24 18:19:24 Sun (Next Sunday)
now.EndOfSunday()         // 2013-11-24 23:59:59.999999999 Sun (End of next Sunday)

t := time.Date(2013, 11, 24, 17, 51, 49, 123456789, time.Now().Location()) // 2013-11-24 17:51:49.123456789 Sun
now.With(t).Monday()              // 2013-11-18 00:00:00 Mon (Last Monday if today is Sunday)
now.With(t).Monday("17:44")       // 2013-11-18 17:44:00 Mon (Last Monday if today is Sunday)
now.With(t).Sunday()              // 2013-11-24 00:00:00 Sun (Beginning Of Today if today is Sunday)
now.With(t).Sunday("18:19:24")    // 2013-11-24 18:19:24 Sun (Beginning Of Today if today is Sunday)
now.With(t).EndOfSunday()         // 2013-11-24 23:59:59.999999999 Sun (End of Today if today is Sunday)
```

## 2.3 解析时间

now 包支持解析的字符串格式包含在切片变量 now.TimeFormats 中，解析时将依次选择不同格式尝试解析。

从字符串解析出时间，返回 error。

```go
t, err := now.Parse("2017")                // 2017-01-01 00:00:00, nil
t, err := now.Parse("2017-10")             // 2017-10-01 00:00:00, nil
t, err := now.Parse("2017-10-13")          // 2017-10-13 00:00:00, nil
t, err := now.Parse("1999-12-12 12")       // 1999-12-12 12:00:00, nil
t, err := now.Parse("1999-12-12 12:20")    // 1999-12-12 12:20:00, nil
t, err := now.Parse("1999-12-12 12:20:21") // 1999-12-12 12:20:21, nil
t, err := now.Parse("10-13")               // 2013-10-13 00:00:00, nil
t, err := now.Parse("12:20")               // 2013-11-18 12:20:00, nil
t, err := now.Parse("12:20:13")            // 2013-11-18 12:20:13, nil
t, err := now.Parse("14")                  // 2013-11-18 14:00:00, nil
t, err := now.Parse("99:99")               // 2013-11-18 12:20:00, Can't parse string as time: 99:99
```

从字符串解析出时间，错误时 panic。

```go
now.MustParse("2013-01-13")             // 2013-01-13 00:00:00
now.MustParse("02-17")                  // 2013-02-17 00:00:00
now.MustParse("2-17")                   // 2013-02-17 00:00:00
now.MustParse("8")                      // 2013-11-18 08:00:00
now.MustParse("2002-10-12 22:14")       // 2002-10-12 22:14:00
now.MustParse("99:99")                  // panic: Can't parse string as time: 99:99
```

