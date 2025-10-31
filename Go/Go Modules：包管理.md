# 1. Go Modules介绍

Go Modules是Go管理包的依赖的工具，自Go 1.11加入，以解决Go的依赖管理问题，淘汰了旧的GOPATH模式。

旧的GOPATH模式将代码存放在GOPATH/src目录下，通过go get来下载外部依赖包。但是GOPATH模式没有版本控制的概念，无法确保下载的依赖包是期望的版本。因此引入了Go Modules解决这些问题。

# 2. 使用Go Modules

首先要确保Go升级到了1.11或以上版本。

然后是设置GO111MODULE，GO111MODULE有三个值：off、on和auto（默认值）。

- GO111MODULE=off：不支持modules功能，通过GOPATH来查找依赖包。
- GO111MODULE=on：使用modules，完全不会通过GOPATH来查找依赖包。
- GO111MODULE=auto：根据当前目录决定是否启用modules，当前目录在GOPATH/src之外，且包含go.mod文件或在包含go.mod文件的目录下面，就会启用modules。

设置GO111MODULE：

```bash
$ go env GO111MODULE # 默认模式为auto
auto

$ go env -w GO111MODULE=on # 设置GO111MODULE
```

当启用modules功能时，依赖包存放位置在GOPATH/pkg/mod，并且允许同一个package的多个版本并存，供项目指定引用。

默认的源地址下载包很容易超时，可以将下载源替换为国内源：

```shell
$ go env GOPROXY # 默认源地址
https://proxy.golang.org,direct

$ go env -w GOPROXY=https://goproxy.cn # 替换为国内源
```

# 3. go mod

go mod带有多个命令，且会在项目中生成和维护go.mod、go.sum两个文件。

## 3.1 go.mod

Go提供了go mod命令来管理包。

新建目录并初始化生成go.mod文件：

```bash
$ mkdir hehe
$ cd hehe
$ go mod init hehe # 初始化go.mod文件
```

新建go.mod后，当其他源文件引用了任何包时，go get、go build、go mod等命令都会更新go.mod文件，并会自动下载用到的依赖包。

生成的go.mod内容示例如下：

```go
module hehe

go 1.15

require (
	github.com/labstack/echo v3.3.10+incompatible // indirect
	github.com/mattn/go-colorable v0.1.1
	github.com/mattn/go-isatty v0.0.7
)

exclude (
	go.etcd.io/etcd/client/v2 v2.305.0-rc.0
	go.etcd.io/etcd/client/v3 v3.5.0-rc.0
)

replace github.com/coreos/bbolt => go.etcd.io/bbolt v1.3.3
```

go.mod中可以包含以下几个语句：

- module：定义当前项目的模块路径。
- go：当前模块的Go语言版本，目前只是标识作用。
- require：指定依赖的包以及特定版本。
- exclude：指定忽略某个依赖包的某版本，Go在版本选择时就会主动跳过这些版本。
- replace：替换依赖包，用来解决错误的包引用，或是用来调试以将依赖包替换成本地的版本。
- retract：是Go 1.16新增的，在某个包的自身中定义，用于声明本包前面某个版本被撤回，提示大家不要用了。

### 3.1.1 require

require中的包后面带有版本号，就可以指定Go下载使用对应的包版本，若没有指定版本号则使用最新的版本。

出现在源文件import中的包是直接依赖的包，而间接依赖的包则会加上indirect注释，且不是所有间接依赖都会出现在go.mod文件中。当module A引用的包B再引用别的包C时，如果B包还不支持Go Modules或go.mod没有带有包C，则会在module A的go.mod文件中以indirect注释引用包C。出现间接依赖可能意味着正在使用过时的包，推荐尽快消除间接依赖，使用依赖的新版本或替换间接依赖。

### 3.1.2 replace

当项目代码依赖一个第三方包，而又想做本地的修改和调试时，可以将这个包下载到本地，然后在go.mod中通过replace来使用它。

首先将依赖的第三方包项目下载到本地。

```bash
$ cd /User/moondo/git
$ git clone github.com/coreos/bbolt
```

然后在go.mod中已经require这个包之后，通过replace将依赖的包从使用GOPATH下的本地缓存目录，改为使用指定目录下的项目。这里指向的路径要用绝对路径。

```
require (
	github.com/coreos/bbolt
)
replace github.com/coreos/bbolt => /User/moondo/git/bbolt
```

然后我们的项目引用这个包的时候，实际上就会引用replace后的路径下的包。我们可以修改它的代码，来测试和观察程序执行的结果了。

## 3.2 go.sum

Go还会生成一个go.sum来记录依赖信息，其中的每一行格式如下：

```
<module> <version>[/go.mod] <hash>
```

module是依赖的路径，version是版本号，hash是h1:开头的字符串。具体内容示例：

```
github.com/google/uuid v1.1.1 h1:Gkbcsh/GbpXz7lPftLA3P6TYMwjCLYm83jiFQZF/3gY=  
github.com/google/uuid v1.1.1/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

每个依赖包版本会包含两条记录，第一条是该依赖包整体的哈希值，第二条是依赖包中go.mod文件的哈希值，如果该依赖包没有go.mod则没有第二条记录。go.sum文件记录的依赖包往往比go.mod文件要多，因为go.mod只需要记录直接依赖的包，而go.sum要记录构建用到的所有包。

Go会将依赖包下载到本地缓存目录GOPATH/pkg/mod/cache/download，并计算哈希值，写入go.sum记录。构建项目时会对本地缓存中go.mod记录的依赖包计算哈希值，若与go.sum中的记录不一致，则表示依赖包不是期望的版本，构建失败。

## 3.3 go mod命令

### 3.3.1 init

初始化创建go.mod文件。

```bash
$ go mod init hehe # 指定模块路径
$ go mod init # 根据.go文件中import的来确认模块路径
```

### 3.3.2 download

下载需要用到的依赖包到缓存目录。

```bash
$ go mod download
$ go mod download golang.org/x/mod@v0.2.0 # 下载指定包
```

### 3.3.3 tidy

检查当前项目代码，加入缺少的包，并移除不需要的包。

```bash
$ go mod tidy
```

### 3.3.4 edit

修改go.mod文件的内容。

```bash
$ go mod edit --require=rsc.io/quote@v3.1.0 # 修改require
$ go mod edit --droprequire=golang.org/x/crypto # 移除require
$ go mod edit -fmt # 格式化go.mod
```

### 3.3.5 why

显示包含依赖包的import路径。

```bash
$ go mod why golang.org/x/text/language
```

### 3.3.6 verify

检查依赖是否正确。

```
$ go mod verify
```

### 3.3.7 vendor

在当前项目新建目录vendor，并将所有依赖包复制到vendor目录里。

```go
$ go mod vendor
```

### 3.3.8 graph

打印模块依赖图。

```bash
$ go mod graph
```

# 4. 参考

- [Go Modules Reference](https://golang.org/ref/mod)
- [go mod 使用 - 掘金](https://juejin.cn/post/6844903798658301960)

