# 1. 简介

github仓库地址：https://github.com/robfig/cron

文档地址：https://pkg.go.dev/github.com/robfig/cron/v3

cron 是 Go 语言的管理定时任务的库，用于实现根据不同的定时规则来定时触发操作。

# 2. 使用

## 2.1 安装

使用 go get 将 cron 包下载到 GOPATH 指定的目录下。

```bash
go get github.com/robfig/cron/v3
```

## 2.2 创建和添加任务

首先创建一个 cron.Cron 对象，用于管理定时任务。

```go
// 创建对象
c := cron.New()

// 创建对象并指定配置，多个配置在参数列表中并列
c := cron.New(cron.WithSeconds())
```

支持的选项有：

* WithSeconds：让时间格式支持秒
* WithLocation：指定时区
* WithParser：使用自定义的解析器
* WithLogger：自定义Logger
* WithChain：Job 包装器

然后通过 AddFunc 方法向其添加任务函数和执行的时间，这里的执行时间格式比较灵活，在下面详细介绍。

```go
// 定义任务函数
func F() {
	fmt.Println("exec func F")
}

// 设置任务函数及其执行时间
c.AddFunc("30 * * * *", F)
```

启动定时执行。

```go
// 在当前协程启动定时任务
c.Run()

// 创建一个协程启动定时任务
c.Start()
```

## 2.3 时间格式

执行时间的格式比较灵活，有几种格式。

**crontab 格式**

类似于 Linux 中的 crontab 命令的格式，最细粒度为分钟，使用 5 个域来表示，用逗号分隔。

1. 分钟：取值范围 0 - 59
2. 小时：取值范围 0 - 23
3. 天：取值范围 1 - 31
4. 月：取值范围 1 - 12 或 JAN - DEC（不区分大小写）
5. 周几：取值范围 0 - 6 或 SUN - SAT（不区分大小写）

每个字段有灵活的表示方法：

```
* 表示任何值

特定数字表示固定的数字
	1 表示固定为 1

/ 表示步长
	*/10 表示所有 10 的倍数的值
	5/10 表示从 5 开始每次加 10 的所有值
	5-30/10 表示从 5 开始且小于等于 30 的每次加 10 的所有值

- 表示范围
	0-11 表示从 0 到 11 的所有值

, 表示列举多个值或范围的并集
	1,2,4,7 表示枚举的各个值
	0-3,*/10 表示多个值或范围的并集

? 等同于 *，只能用于天和周几
```

下面列举几个例子：

```
30 * * * * 每小时的 30 分

0 0-3,10,23 * * * 每天 0-3 点、10 点、23 点的 0 分

0 1 * * * 每天 01:00

*/10 12-15 1 * * 每个月 1 号的 12 到 15 点的每 10 分钟
```

如果创建定时器时带有选项 cron.WithSeconds，则格式应该是 6 段，第一段表示秒。

**预定义时间规则**

这些是 cron 预定义的一些时间规则：

```
@yearly 或 @annually 每年第一天的 0 点
@monthly 每月第一天的 0 点
@weekly 每周第一天（周日）的 0 点
@daily 或 @midnight 每天的 0 点
@hourly 每小时的 0 分
```

**固定时间间隔**

从定时器开始运行起，每隔固定时间执行。

```
@every 1s 每秒
@every 2m 每 2 分钟
@every 1h 每小时
@every 1m30s 每 1 分钟 30 秒
```

# 3. 源码分析

## 3.1 Cron

cron.Cron 结构体表示定时执行器。所有的任务和调度器保存在 entries 成员内。

```go
// cron.go
// Cron 定时器
type Cron struct {
	entries   []*Entry          // 保存所有 Entry 指针的 slice
	chain     Chain             // 将任务函数和其它如日志、同步的函数绑定执行，可以自定义
	stop      chan struct{}     // 定时器停止信号
	add       chan *Entry       // Entry 添加通道
	remove    chan EntryID      // Entry 删除通道
	snapshot  chan chan []Entry // Entry 快照通道
	running   bool              // 定时器是否开始运行
	logger    Logger
	runningMu sync.Mutex        // 锁，保证线程安全
	location  *time.Location
	parser    ScheduleParser    // 解析器，可以自定义
	nextID    EntryID           // 待分配给新 Entry 的 ID
	jobWaiter sync.WaitGroup    // 停止时任务执行等待
}
```

创建对象。创建使用了选项模式，可以自由地在参数中追加配置，并在函数中依次应用配置。

```go
// New 创建
func New(opts ...Option) *Cron {
	c := &Cron{
		entries:   nil,
		chain:     NewChain(),
		add:       make(chan *Entry),
		stop:      make(chan struct{}),
		snapshot:  make(chan chan []Entry),
		remove:    make(chan EntryID),
		running:   false,
		runningMu: sync.Mutex{},
		logger:    DefaultLogger,
		location:  time.Local,
		parser:    standardParser,
	}
	for _, opt := range opts { // 依次应用配置
		opt(c)
	}
	return c
}
```

添加任务。如果定时器已经在执行，则任务先添加到 add 通道，在定时器循环中获取通道信息再向 entries 切片中加入任务。

```go
// AddFunc 添加函数和时间规则
func (c *Cron) AddFunc(spec string, cmd func()) (EntryID, error) {
	return c.AddJob(spec, FuncJob(cmd))
}

// AddJob 解析时间规则并添加任务
func (c *Cron) AddJob(spec string, cmd Job) (EntryID, error) {
	schedule, err := c.parser.Parse(spec) // 解析时间规则
	if err != nil {
		return 0, err
	}
	return c.Schedule(schedule, cmd), nil // 添加任务
}

// Schedule
func (c *Cron) Schedule(schedule Schedule, cmd Job) EntryID {
	c.runningMu.Lock() // 加锁
	defer c.runningMu.Unlock()
	c.nextID++
	entry := &Entry{ // 创建 Entry 对象
		ID:         c.nextID,
		Schedule:   schedule,
		WrappedJob: c.chain.Then(cmd), // 将执行函数绑定保存在 chain 的其它行为，执行时依次调用
		Job:        cmd, // 只保存函数
	}
	if !c.running { // 未在运行，将 Entry 对象加到 entries
		c.entries = append(c.entries, entry)
	} else {
		c.add <- entry // 运行中，将 Entry 对象加到 add
	}
	return entry.ID
}
```

启动定时执行。在一个 select 中去等待下次任务执行的定时器

```go
// Run 在当前协程启动定时任务
func (c *Cron) Run() {
	c.runningMu.Lock()
	if c.running { // 避免重复启动
		c.runningMu.Unlock()
		return
	}
	c.running = true
	c.runningMu.Unlock()
	c.run() // 当前协程启动
}

// run 启动
func (c *Cron) run() {
	c.logger.Info("start")

	// Figure out the next activation times for each entry.
	now := c.now()
	for _, entry := range c.entries { // 遍历 Entry 计算任务下次运行的时间
		entry.Next = entry.Schedule.Next(now)
		c.logger.Info("schedule", "now", now, "entry", entry.ID, "next", entry.Next)
	}

	for {
		sort.Sort(byTime(c.entries)) // 任务下次运行的时间排序

		var timer *time.Timer
		if len(c.entries) == 0 || c.entries[0].Next.IsZero() { // 没有要执行的任务，一直等待
			timer = time.NewTimer(100000 * time.Hour)
		} else { // 计算当前距离最近要执行任务的等待时间
			timer = time.NewTimer(c.entries[0].Next.Sub(now))
		}

		for {
			select {
			case now = <-timer.C: // 有任务要执行
				now = now.In(c.location)
				c.logger.Info("wake", "now", now)

				for _, e := range c.entries { // 遍历每个 Entry
					if e.Next.After(now) || e.Next.IsZero() { // entries 有序，因此遍历到晚于现在的就 break
						break
					}
					c.startJob(e.WrappedJob) // 创建协程执行 Job
					e.Prev = e.Next
					e.Next = e.Schedule.Next(now) // 重新计算下次执行时间
					c.logger.Info("run", "now", now, "entry", e.ID, "next", e.Next)
				}

			case newEntry := <-c.add: // 有任务添加，将退出 for 循环触发任务重新排序
				timer.Stop()
				now = c.now()
				newEntry.Next = newEntry.Schedule.Next(now)
				c.entries = append(c.entries, newEntry)
				c.logger.Info("added", "now", now, "entry", newEntry.ID, "next", newEntry.Next)

			case replyChan := <-c.snapshot: // 有任务快照调用
				replyChan <- c.entrySnapshot()
				continue

			case <-c.stop: // 停止定时器
				timer.Stop()
				c.logger.Info("stop")
				return

			case id := <-c.remove: // 有任务要删除
				timer.Stop()
				now = c.now()
				c.removeEntry(id)
				c.logger.Info("removed", "entry", id)
			}

			break
		}
	}
}

// startJob 创建协程执行 Job
func (c *Cron) startJob(j Job) {
	c.jobWaiter.Add(1) // jobWaiter 加 1
	go func() {
		defer c.jobWaiter.Done() // 执行完后 jobWaiter 减 1，停止定时器时等待所有任务完成
		j.Run()
	}()
}
```

停止定时器。

```go
// Stop 停止
func (c *Cron) Stop() context.Context {
	c.runningMu.Lock() // 加锁
	defer c.runningMu.Unlock()
	if c.running {
		c.stop <- struct{}{} // 发送停止信号到 stop 通道
		c.running = false
	}
	ctx, cancel := context.WithCancel(context.Background())
	go func() {
		c.jobWaiter.Wait() // 全部任务执行完成后才能停止
		cancel()
	}()
	return ctx
}
```

## 3.2 Entry

Entry 结构体包含调度器和要执行的函数。

```go
// cron.go
// Entry 包含调度器和要执行的函数
type Entry struct {
	ID EntryID
	Schedule Schedule // 调度器
	Next time.Time // 任务下次运行的时间
	Prev time.Time // 任务上次运行的时间
	// WrappedJob is the thing to run when the Schedule is activated.
	WrappedJob Job // 调度器执行的任务
	Job Job // 提交定时器的任务
}

// EntryID entry 的 ID
type EntryID int

// FuncJob 实现了 cron.Job 接口，即实现 Run 方法，直接调用该函数
type FuncJob func()
func (f FuncJob) Run() { f() }
```

## 3.3 Chain

Chain 结构体使用装饰器模式，将要执行的函数用其它自定义的行为包含起来，在执行的时候依次执行。

```go
// chain.go
// JobWrapper 将函数绑定一些其他行为
type JobWrapper func(Job) Job

// Chain 一系列的 JobWrappers，将提交的函数绑定一些日志、同步等的行为
type Chain struct {
	wrappers []JobWrapper
}

// NewChain 创建对象
func NewChain(c ...JobWrapper) Chain {
	return Chain{c}
}

// Then 将添加的函数用 Chain 中的层层 wrapper 包起来
// 执行 NewChain(m1, m2, m3).Then(job) 等同于 m1(m2(m3(job)))
func (c Chain) Then(j Job) Job {
	for i := range c.wrappers {
		j = c.wrappers[len(c.wrappers)-i-1](j)
	}
	return j
}
```

## 3.4 Parser

Parser 是解析定时规则的解析器。

```go
// parser.go
// Parser 解析器
type Parser struct {
	options ParseOption
}

// ParseOption 解析配置
type ParseOption int
const (
	Second         ParseOption = 1 << iota // 秒
	SecondOptional                         // 秒
	Minute                                 // 分钟
	Hour                                   // 小时
	Dom                                    // 天
	Month                                  // 月
	Dow                                    // 周几
	DowOptional                            // 周几
	Descriptor                             // 描述，包含预定义时间规则和固定时间间隔
)

// 默认解析器，包含分钟、小时、天、月、周几、描述的方式
var standardParser = NewParser(
	Minute | Hour | Dom | Month | Dow | Descriptor,
)

// option.go
// WithSeconds 相比默认解析器增加了秒
func WithSeconds() Option {
	return WithParser(NewParser(
		Second | Minute | Hour | Dom | Month | Dow | Descriptor,
	))
}
```

解析时间格式并返回 Schedule 对象。

```go
// Parse 解析时间格式
func (p Parser) Parse(spec string) (Schedule, error) {
	if strings.HasPrefix(spec, "@") { // 描述格式
		if p.options&Descriptor == 0 {
			return nil, fmt.Errorf("parser does not accept descriptors: %v", spec)
		}
		return parseDescriptor(spec, loc) // 解析描述格式
	}

	fields := strings.Fields(spec) // 根据空格切分为 slice

	var err error
	fields, err = normalizeFields(fields, p.options) // 检验并补充忽略/可选字段
	if err != nil {
		return nil, err
	}

	field := func(field string, r bounds) uint64 {
		if err != nil {
			return 0
		}
		var bits uint64
		bits, err = getField(field, r) // 解析字段中 * / - 等格式，将匹配的值以位存储到一个 uint64 中
		return bits
	}

	var (
		second     = field(fields[0], seconds)
		minute     = field(fields[1], minutes)
		hour       = field(fields[2], hours)
		dayofmonth = field(fields[3], dom)
		month      = field(fields[4], months)
		dayofweek  = field(fields[5], dow)
	)
	if err != nil {
		return nil, err
	}

	return &SpecSchedule{ // 返回 SpecSchedule 对象
		Second:   second,
		Minute:   minute,
		Hour:     hour,
		Dom:      dayofmonth,
		Month:    month,
		Dow:      dayofweek,
		Location: loc,
	}, nil
}
```

## 3.5 Schedule

Schedule 接口是调度器，只有一个方法 Next，表示任务的下次执行时间。

```go
// parse.go
// Schedule 调度器
type Schedule interface {
	// Next 下次执行时间
	Next(time.Time) time.Time
}
```

SpecSchedule 结构体实现了 Schedule 接口。

```go
// spec.go
// SpecSchedule 调度器
type SpecSchedule struct {
	Second, Minute, Hour, Dom, Month, Dow uint64 // 匹配的值以位存储
	Location *time.Location // 地区
}

// Next 获取下次执行时间
func (s *SpecSchedule) Next(t time.Time) time.Time {
	// 转换时间为调度器的时区
	origLocation := t.Location()
	loc := s.Location
	if loc == time.Local {
		loc = t.Location()
	}
	if s.Location != time.Local {
		t = t.In(s.Location)
	}

	t = t.Add(1*time.Second - time.Duration(t.Nanosecond())*time.Nanosecond) // 转化为不早于当前时间的最早整秒
	added := false                                                           // 标记一个字段是否增加了
	yearLimit := t.Year() + 5

WRAP:
	if t.Year() > yearLimit {
		return time.Time{} // 如果 5 年内都没有匹配则返回 0
	}

	for 1<<uint(t.Month())&s.Month == 0 { // 匹配月
		if !added {
			added = true
			t = time.Date(t.Year(), t.Month(), 1, 0, 0, 0, 0, loc)
		}
		t = t.AddDate(0, 1, 0)
		if t.Month() == time.January {
			goto WRAP
		}
	}

	for !dayMatches(s, t) { // 匹配日
		if !added {
			added = true
			t = time.Date(t.Year(), t.Month(), t.Day(), 0, 0, 0, 0, loc)
		}
		t = t.AddDate(0, 0, 1)
		if t.Hour() != 0 {
			if t.Hour() > 12 {
				t = t.Add(time.Duration(24-t.Hour()) * time.Hour)
			} else {
				t = t.Add(time.Duration(-t.Hour()) * time.Hour)
			}
		}
		if t.Day() == 1 {
			goto WRAP
		}
	}

	for 1<<uint(t.Hour())&s.Hour == 0 { // 匹配小时
		if !added {
			added = true
			t = time.Date(t.Year(), t.Month(), t.Day(), t.Hour(), 0, 0, 0, loc)
		}
		t = t.Add(1 * time.Hour)
		if t.Hour() == 0 {
			goto WRAP
		}
	}

	for 1<<uint(t.Minute())&s.Minute == 0 { // 匹配分钟
		if !added {
			added = true
			t = t.Truncate(time.Minute)
		}
		t = t.Add(1 * time.Minute)
		if t.Minute() == 0 {
			goto WRAP
		}
	}

	for 1<<uint(t.Second())&s.Second == 0 { // 匹配秒
		if !added {
			added = true
			t = t.Truncate(time.Second)
		}
		t = t.Add(1 * time.Second)
		if t.Second() == 0 {
			goto WRAP
		}
	}

	return t.In(origLocation) // 转换时区
}
```

# 4. 参考

* [Go 每日一库之 cron - 大俊的博客](https://darjun.github.io/2020/06/25/godailylib/cron/)

