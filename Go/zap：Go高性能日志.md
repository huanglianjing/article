# 1. 简介

github仓库地址：https://github.com/uber-go/zap

文档地址：https://pkg.go.dev/github.com/uber-go/zap

zap 是 Go 语言的日志库，相比标准日志库 log，对性能和内存分配做了极致的优化。

由于 fmt.Printf 之类的方法大量使用了 interface{} 和 反射，性能不高且内存分配较频繁，zap 为了提高性能和减少内存分配次数，默认使用强类型、结构化的日志，对每个类型都提供了相应的方法记录字段。

# 2. 使用

## 2.1 安装

使用 go get 将 zap 包下载到 GOPATH 指定的目录下。

```bash
go get go.uber.org/zap
```

## 2.2 示例

使用 Logger 打印日志，每个字段必须确定类型，使用指定的函数。

```go
logger, _ := zap.NewProduction()
defer logger.Sync()
logger.Info("failed to fetch URL",
	zap.String("url", "www.google.com"),
	zap.Int("attempt", 3),
	zap.Duration("backoff", time.Second),
)

// {"level":"info","ts":1713955182.968443,"caller":"go-test/main.go:12","msg":"failed to fetch URL","url":"www.google.com","attempt":3,"backoff":1}
```

使用 SugaredLogger 来将 key - value 对传入参数，不需要确认字段类型，性能比 Logger 低一些。

```go
logger, _ := zap.NewProduction()
defer logger.Sync()
sugar := logger.Sugar()
sugar.Infow("failed to fetch URL",
	"url", "www.google.com",
	"attempt", 3,
	"backoff", time.Second,
)

// {"level":"info","ts":1713955331.505285,"caller":"go-test/main.go:13","msg":"failed to fetch URL","url":"www.google.com","attempt":3,"backoff":1}
```

## 2.3 创建

有几个创建 logger 的方法，适用于不同场景。

```go
// New 高度定制化
func New(core zapcore.Core, options ...Option) *Logger

// NewExample 测试场景
func NewExample(options ...Option) *Logger

// NewDevelopment 开发场景
func NewDevelopment(options ...Option) (*Logger, error)

// NewProduction 生产环境
func NewProduction(options ...Option) (*Logger, error)
```

创建 logger 可以设置配置。

```go
logger, _ := zap.NewProduction(zap.AddCaller())
```

## 2.4 打印日志

zap 提供了不同的方法打印日志，根据 Logger 创建的不同会判断是否应该打印该日志，如生产环境下，Debug 方法将不会打印日志。

日志打印包含一条信息和若干个字段。

```go
// Debug
func (log *Logger) Debug(msg string, fields ...Field)

// Info
func (log *Logger) Info(msg string, fields ...Field)

// Error
func (log *Logger) Error(msg string, fields ...Field)

// Fatal
func (log *Logger) Fatal(msg string, fields ...Field)
```

打印的参数中，根据打印字段的类型不同，Field 使用 zap 定制的多个函数来生成。

```go
// Any
func Any(key string, value interface{}) Field

// Binary
func Binary(key string, val []byte) Field

// Error
func Error(err error) Field

// Bool
func Bool(key string, val bool) Field

// 整数
func Int(key string, val int) Field
func Int32(key string, val int32) Field
func Int64(key string, val int64) Field
func Uint(key string, val uint) Field
func Uint32(key string, val uint32) Field
func Uint64(key string, val uint64) Field

// String
func String(key string, val string) Field

// 浮点数
func Float32s(key string, nums []float32) Field
func Float64(key string, val float64) Field

// 时间
func Duration(key string, val time.Duration) Field
func Time(key string, val time.Time) Field
```

通过 zap.Namespace 可以构建命名空间，打印嵌套的层级。

```go
logger.Info("failed to fetch URL",
	zap.String("url", "www.google.com"),
	zap.Namespace("metrics"),
	zap.Int("attempt", 3),
	zap.Namespace("duration"),
	zap.Duration("backoff", time.Second),
)

// {"level":"info","ts":1713957058.075815,"caller":"go-test/main.go:12","msg":"failed to fetch URL","url":"www.google.com","metrics":{"attempt":3,"duration":{"backoff":1}}}
```

## 2.5 配置

配置结构如下：

```go
// config.go
// Config 配置
type Config struct {
  Level AtomicLevel                    // 日志级别
  Encoding string                      // 输出格式，默认为 JSON
  EncoderConfig zapcore.EncoderConfig  // 编码配置
  OutputPaths []string                 // 输出路径，可以是文件路径和 stdout
  ErrorOutputPaths []string            // 错误输出路径，可以是文件路径和 stdout
  InitialFields map[string]interface{} // 每条日志固定输出字段
}

// zapcore/encoder.go
// EncoderConfig 编码配置
type EncoderConfig struct {
  MessageKey     string // 信息的键名，默认为 msg
  LevelKey       string // 级别的键名，默认为 level
  TimeKey        string // 时间的键名
  NameKey        string
  CallerKey      string
  StacktraceKey  string
  LineEnding     string
  EncodeLevel    LevelEncoder // 级别的格式，默认为小写
  EncodeTime     TimeEncoder
  EncodeDuration DurationEncoder
  EncodeCaller   CallerEncoder
  EncodeName     NameEncoder
}
```

通过自定义配置来创建 logger。

```go
rawJSON := `{
  "level":"info",
  "encoding":"json",
  "outputPaths": ["stdout", "server.log"],
  "errorOutputPaths": ["stderr"],
  "initialFields":{"name":"dj"},
  "encoderConfig": {
    "messageKey": "message",
    "levelKey": "level",
    "levelEncoder": "lowercase"
  }
}`
var config zap.Config
json.Unmarshal([]byte(rawJSON), &config)
logger, _ := config.Build()
```

# 3. 参考

* [Go 每日一库之 zap - 大俊的博客](https://darjun.github.io/2020/04/23/godailylib/zap/)

