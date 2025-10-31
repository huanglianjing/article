# 1. 简介

github仓库地址：https://github.com/spf13/viper

文档地址：https://pkg.go.dev/github.com/spf13/viper

viper 是 Go 语言的配置文件管理库，支持 JSON、YAML、TOML 等多种格式的配置文件，可以设置监听配置文件的修改自动加载新配置。

# 2. 使用

## 2.1 安装

使用 go get 将 viper 包下载到 GOPATH 指定的目录下。

```bash
go get github.com/spf13/viper
```

## 2.2 读取配置文件

设置默认值，可以在读取配置文件前设置一些默认的配置。

```go
viper.SetDefault("ContentDir", "content")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```

先设置配置文件的名称和文件类型，然后添加配置文件搜索路径，可以添加多个。然后就可以读取配置文件了。

```go
viper.SetConfigName("config")                // 配置文件名称
viper.SetConfigType("yaml")                  // 配置文件类型
viper.AddConfigPath("$HOME")                 // 添加配置文件搜索路径
viper.AddConfigPath("etc")                   // 添加配置文件搜索路径
if err := viper.ReadInConfig(); err != nil { // 读取配置
	if _, ok := err.(viper.ConfigFileNotFoundError); ok {
		panic(fmt.Errorf("config file not found"))
	}
	panic(fmt.Errorf("ReadInConfig fail: %w", err))
}
```

也可以将内存中的配置修改后写入配置文件，写入时会将设置的默认值也一并写入。

```go
viper.WriteConfig()                         // 将配置写入搜索到的配置文件
viper.SafeWriteConfig()                     // 仅当文件不存在时，将配置写入搜索到的配置文件
viper.WriteConfigAs("etc/config1.yaml")     // 将配置文件写入指定文件
viper.SafeWriteConfigAs("etc/config1.yaml") // 仅当文件不存在时，将配置文件写入指定文件
```

以上设置和读取是将配置文件读取到了 viper 默认的一个全局的对象中，对于需要读取多个配置文件的场景，可以为每个配置创建一个 viper.Viper 对象。

```go
x := viper.New()
y := viper.New()
x.SetDefault("ContentDir", "content")
y.SetDefault("ContentDir", "foobar")
```

## 2.3 监听配置文件

viper 支持监听配置文件，当配置文件发生改动时触发操作，并会重新读取配置。

```go
viper.OnConfigChange(func(e fsnotify.Event) {
	fmt.Println("Config file changed:", e.Name)
})
viper.WatchConfig()
```

## 2.4 读写配置

对于根目录下的配置 key，直接使用 key 名称，如 mysql。对于数组和映射表，使用数字和子结构的名称的多层结构，如 mysql.0.port。

主动设置配置，可能是来自于命令行参数或是程序的逻辑。

```go
viper.Set("Verbose", true)
viper.Set("LogFile", LogFile)

// Set 设置配置
func Set(key string, value any)
```

以三种最常用的配置文件类型为例子：

**JSON**

```json
{
  "env": "prod",
  "mysql": [
    {
      "host": "127.0.0.1",
      "port": 3306
    }
  ]
}
```

**YAML**

```yaml
env: prod
mysql:
  - host: 127.0.0.1
    port: 3306
```

**TOML**

```toml
env = "prod"
[[mysql]]
host = "127.0.0.1"
port = 3306
```

读取配置，有多个方法可以读取不同类型的配置，如果读取不到或者类型不匹配则返回零值。

设置和读取都是不区分大小写的。

```go
env := viper.GetString("env")
mysqlPort := viper.GetInt("mysql.0.port")

// Get 读取配置
func Get(key string) any
func GetBool(key string) bool
func GetInt(key string) int
func GetInt64(key string) int64
func GetIntSlice(key string) []int
func GetFloat64(key string) float64
func GetString(key string) string
func GetStringSlice(key string) []string
func GetStringMap(key string) map[string]any
func GetStringMapString(key string) map[string]string
```

判断配置是否设置，对与配置的值恰好是该类型的零值，可以用该方法区分这种情况。

```go
viper.IsSet("env")
```

将配置映射到一个结构体对象。

映射默认会按照成员名称匹配，切不区分大小写。也可以使用标签的 mapstructure 指定配置的 key。

```go
// Config 配置
type Config struct {
	Env   string        `mapstructure:"env"`
	Mysql []MysqlConfig `mapstructure:"mysql"`
}

// MysqlConfig 配置
type MysqlConfig struct {
	Host string `mapstructure:"host"`
	Port int    `mapstructure:"port"`
}

var C Config
err := viper.Unmarshal(&C)
```

