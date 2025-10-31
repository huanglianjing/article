# 1. go test

go test 命令是一个按照一定约定和组织的测试代码的驱动程序，它会遍历当前包目录内的所有名称符合 *_test.go 的源文件，找出其中命名规范符合以下之一的函数，然后生成一个临时的 main 包用于调用相应的测试函数，然后构建并运行、报告测试结果，最后清理测试中生成的临时文件。

* 测试函数：函数名前缀为 Test，用来测试程序的一些逻辑行为是否正确
* 基准函数：函数名前缀为 Benchmark，用来测试函数的性能
* 示例函数：函数名前缀为 Example，为文档提供示例文档

命令格式：go test [-c] [-i] [build flags] [packages] [flags for test binary]

go test 命令参数如下：

-c 编译成可执行的二进制文件，但是不运行测试

-i 安装测试包依赖的package，但是不运行测试

-v 是否输出全部的单元测试用例，默认没有加上，所以只输出失败的单元测试用例

-run=pattern 只跑哪些单元测试用例

-bench=patten 只跑哪些性能测试用例

-benchmem 是否在性能测试的时候输出内存情况

-benchtime=t 性能测试运行的时间，默认是1s

-cpuprofile=cpu.out 是否输出cpu性能分析文件

-memprofile=mem.out 是否输出内存性能分析文件

-blockprofile=block.out 是否输出内部goroutine阻塞的性能分析文件

-memprofilerate=n 内存性能分析打点的内存分配间隔

-blockprofilerate=n goroutine阻塞时候打点的纳秒数

-parallel=n 性能测试的程序并行cpu数，默认等于GOMAXPROCS

-timeout=t 如果测试用例运行时间超过t，则抛出panic

-cpu=1,2,4 程序运行在哪些CPU上面，使用二进制的1所在位代表

-short 将那些运行时间较长的测试用例运行时间缩短

-failfast 失败后不继续后面的测试

# 2. 单元测试

单元测试（unit testing），是指对软件中的最小可测试单元进行检查和验证，通过测试函数和方法等小段代码，使得我们可以及早发现程序错误，保证了代码的正确性。

## 2.1 测试函数

我们可以对已有的源文件，在同一目录下，增加一个文件名后追加 _test 的新源文件，用来编写测试函数，例子如下：

```go
// 已有源文件
tool.go

// 单元测试源文件
tool_test.go
```

测试函数必须导入 testing 包，函数名必须以 Test 开头，后面名称以大写字母或下划线开头，一般针对某个函数的测试函数可以在原函数名称前加 Test 前缀。测试函数的参数必须为 t *testing.T。

```go
// 原函数
func Concat() {}

// 测试函数
func TestConcat(t *testing.T) {
	// ...
}
```

testing 包文档：https://pkg.go.dev/testing

参数 t 可以用于在测试函数中报告测试失败和附加的日志信息，包含方法：

```go
// 标记测试失败，继续执行
func (c *T) Fail()

// 标记测试失败，中止退出
func (c *T) FailNow()

// 测试是否失败
func (c *T) Failed() bool

// 打印日志
func (c *T) Log(args ...interface{})
func (c *T) Logf(format string, args ...interface{})

// 打印日志，标记测试失败，继续执行
func (c *T) Error(args ...interface{})
func (c *T) Errorf(format string, args ...interface{})

// 打印日志，标记测试失败，中止退出
func (c *T) Fatal(args ...interface{})
func (c *T) Fatalf(format string, args ...interface{})

// 单元测试的名称
func (c *T) Name() string

// 并发执行
func (t *T) Parallel()

// 执行子测试
func (t *T) Run(name string, f func(t *T)) bool

// 跳过测试
func (c *T) Skip(args ...interface{})
func (c *T) SkipNow()
func (c *T) Skipf(format string, args ...interface{})

// 测试是否跳过
func (c *T) Skipped() bool
```

## 2.2 示例

以已有的函数为例：

```go
// src/tool/tool.go
package tool

// Concat 拼接字符串
func Concat(strs ...string) string {
	str := ""
	for _, s := range strs {
		str += s
	}
	return str
}
```

编写测试函数：

```go
// src/tool/tool_test.go
package tool

import (
	"testing"
)

func TestConcat(t *testing.T) {
	strs := []string{"a", "b", "c"}
	str := Concat(strs...)
	expect := "abc"
	if str != expect {
		t.Errorf("expect: %s, got: %s", expect, str)
	}
}
```

在命令行去到当前包路径，执行测试命令：

```bash
$ cd src/tool/
$ go test
PASS
ok      go-test/src/tool        0.301s

# 也可以通过相对路径指定测试路径
$ go test ./src/tool/
```

尝试增加一个错误的测试函数：

```go
func TestConcatFail(t *testing.T) {
	strs := []string{"1", "2"}
	str := Concat(strs...)
	expect := "123"
	if str != expect {
		t.Errorf("expect: %s, got: %s", expect, str)
	}
}
```

执行测试命令，将会执行所有测试函数，只能在全部执行成功时展示成功，否则展示出错的那个报错，信息如下：

```bash
$ go test
--- FAIL: TestConcatFail (0.00s)
    tool_test.go:21: expect: 123, got: 12
FAIL
exit status 1
FAIL    go-test/src/tool        0.169s
```

-v 参数可以分别输出每个测试函数的执行情况。

```bash
$ go test -v
=== RUN   TestConcat
--- PASS: TestConcat (0.00s)
=== RUN   TestConcatFail
    tool_test.go:21: expect: 123, got: 12
--- FAIL: TestConcatFail (0.00s)
FAIL
exit status 1
FAIL    go-test/src/tool        0.267s
```

-run 参数可以指定要执行的测试函数，通过正则表达式匹配测试函数名称。

```bash
$ go test -v -run=Fail
=== RUN   TestConcatFail
    tool_test.go:21: expect: 123, got: 12
--- FAIL: TestConcatFail (0.00s)
FAIL
exit status 1
FAIL    go-test/src/tool        0.182s
```

## 2.3 测试组

可以讲多组测试用例合并到一起，放到一个测试函数内进行测试。

```go
// src/tool/tool_test.go
package tool

import (
	"testing"
)

func TestGroupConcat(t *testing.T) {
	tests := []struct {
		strs []string
		want string
	}{
		{[]string{"a", "b", "c"}, "abc"},
		{[]string{"11 2", "3"}, "1 23"},
		{[]string{}, ""},
	}
	for _, test := range tests {
		if got := Concat(test.strs...); got != test.want {
			t.Errorf("expect: %s, got: %s", test.want, got)
		}
	}
}
```

如果执行出错，会展示具体哪个测试用例报错。

```bash
$ go test -v
=== RUN   TestGroupConcat
    tool_test.go:18: expect: 1 23, got: 11 23
--- FAIL: TestGroupConcat (0.00s)
FAIL
exit status 1
FAIL    go-test/src/tool        0.265s
```

## 2.4 子测试

当测试用例较多时，可以在测试函数内用 t.Run 方法执行子函数，

```go
// src/tool/tool_test.go
package tool

import (
	"testing"
)

func TestSubConcat(t *testing.T) {
	type args struct {
		strs []string
	}
	tests := []struct {
		name string
		args args
		want string
	}{
		{"alphabet", args{strs: []string{"a", "b", "c"}}, "abc"},
		{"number", args{strs: []string{"11 2", "3"}}, "1 23"},
		{"empty", args{strs: []string{}}, ""},
	}
	for _, test := range tests {
		t.Run(test.name, func(t *testing.T) {
			if got := Concat(test.args.strs...); got != test.want {
				t.Errorf("expect: %s, got: %s", test.want, got)
			}
		})
	}
}
```

如果执行出错，会展示具体出错的测试用例名称以及报错。

```bash
$ go test -v
=== RUN   TestSubConcat
=== RUN   TestSubConcat/alphabet
=== RUN   TestSubConcat/number
    tool_test.go:23: expect: 1 23, got: 11 23
=== RUN   TestSubConcat/empty
--- FAIL: TestSubConcat (0.00s)
    --- PASS: TestSubConcat/alphabet (0.00s)
    --- FAIL: TestSubConcat/number (0.00s)
    --- PASS: TestSubConcat/empty (0.00s)
FAIL
exit status 1
FAIL    go-test/src/tool        0.253s
```

可以通过 -run 参数以正则表达式的形式指定子测试。

```go
$ go test -v -run=TestSubConcat/alphabet
=== RUN   TestSubConcat
=== RUN   TestSubConcat/alphabet
--- PASS: TestSubConcat (0.00s)
    --- PASS: TestSubConcat/alphabet (0.00s)
PASS
ok      go-test/src/tool        0.193s
```

## 2.5 测试覆盖率

-cover 参数可以查看测试覆盖率，即当前包的所有代码中，在单元测试函数中被至少运行到一次的代码，占全部代码的比例。

执行后，go test 会先走一遍所有的测试函数，然后再统计出测试覆盖率。

```bash
$ go test -cover
PASS
        go-test/src/tool        coverage: 100.0% of statements
ok      go-test/src/tool        0.228s
```

Go 还提供了一个额外的 -coverprofile 参数，用来将覆盖率相关的记录信息输出到一个文件。然后再以 -func 参数来查看每个函数的覆盖率。

```bash
go test -cover -coverprofile=cover.out -covermode=count
go tool cover -func=cover.out
```

# 3. 基准测试

基准测试是在一定的工作负载之下检测程序性能的一种方法。

## 3.1 基准测试函数

基准测试函数必须导入 testing 包，函数名必须以 Benchmark 开头，后面名称以大写字母或下划线开头，一般针对某个函数的测试函数可以在原函数名称前加 Benchmark 前缀。基准测试函数的参数必须为 b *testing.B，基准测试必须包含一个 for 循环且要执行 b.N 次。

```go
// 原函数
func Concat() {}

// 基准测试函数
func BenchmarkConcat(b *testing.B) {
	// ...
}
```

参数 b 可以用于在测试函数中报告测试失败和附加的日志信息，包含方法：

```go
// 标记测试失败，继续执行
func (c *B) Fail()

// 标记测试失败，中止退出
func (c *B) FailNow()

// 测试是否失败
func (c *B) Failed() bool

// 打印日志
func (c *B) Log(args ...interface{})
func (c *B) Logf(format string, args ...interface{})

// 打印日志，标记测试失败，继续执行
func (c *B) Error(args ...interface{})
func (c *B) Errorf(format string, args ...interface{})

// 打印日志，标记测试失败，中止退出
func (c *B) Fatal(args ...interface{})
func (c *B) Fatalf(format string, args ...interface{})

// 单元测试的名称
func (c *B) Name() string

// 执行子测试
func (b *B) Run(name string, f func(b *B)) bool

// 并发执行
func (b *B) RunParallel(body func(*PB))

// 跳过测试
func (c *B) Skip(args ...interface{})
func (c *B) SkipNow()
func (c *B) Skipf(format string, args ...interface{})

// 测试是否跳过
func (c *B) Skipped() bool
```

## 3.2 示例

还是以上面单元测试的 Concat 函数为例：

```go
// src/tool/tool.go
package tool

// Concat 拼接字符串
func Concat(strs ...string) string {
	str := ""
	for _, s := range strs {
		str += s
	}
	return str
}
```

编写基准测试函数：

```go
package tool

import (
	"testing"
)

func BenchmarkConcat(b *testing.B) {
	strs := []string{"a", "b", "c"}
	for i := 0; i < b.N; i++ {
		Concat(strs...)
	}
}
```

执行测试命令：

```bash
$ cd src/tool/
$ go test -bench=Concat -benchmem
goos: darwin
goarch: amd64
pkg: go-test/src/tool
cpu: Intel(R) Core(TM) i5-1038NG7 CPU @ 2.00GHz
BenchmarkConcat-8       19056243                59.41 ns/op            6 B/op          2 allocs/op
PASS
ok      go-test/src/tool        1.412s
```

-benchmem 参数可以得到内存分配的统计数据。

这里的 BenchmarkConcat-8 表示 GOMAXPROCS = 19056243 表示调用 Concat 函数的次数，59.41 ns/op 表示每次调用的平均耗时，6 B/op 表示每次调用分配的内存大小，2 allocs/op 表示每次调用进行了多少次内存分配。

## 3.3 重置时间

在进行基准测试的开头如果需要做一些其他工作，但是又不想统计进时间统计内，可以先重置一下计时器。

```go
func BenchmarkConcat(b *testing.B) {
	// do sth.
	b.ResetTimer() // 重置计时器
	strs := []string{"a", "b", "c"}
	for i := 0; i < b.N; i++ {
		Concat(strs...)
	}
}
```

# 4. 示例函数

示例函数以 Example 作为前缀，它们没有参数和返回值。

```go
func ExampleConcat() {
	strs := []string{"a", "b", "c"}
	expect := "abc"
	got := Concat(strs...)
	fmt.Printf("expect: %s, got: %s", expect, got)
}
```

示例函数能够作为文档直接使用，例如基于web的godoc中能把示例函数与对应的函数或包相关联。

# 5. 参考

* [单元测试 · Go语言中文文档](https://www.topgoer.com/%E5%87%BD%E6%95%B0/%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95.html)
* [testing package - testing - Go Packages](https://pkg.go.dev/testing)