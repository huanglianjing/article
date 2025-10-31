# 1. 安装

## 1.1 CentOS安装Go

```bash
$ yum install golang
```

## 1.2 macOS安装Go

macOS 可以通过 brew 安装 Go，需要先安装 brew。

```bash
$ brew install go
```

# 2. 工具

Go的工具链通过go命令配合子命令使用。

常用命令：

```
build      编译包和依赖
clean      删除目标文件
doc        显示文档
env        显示go环境变量
fmt        格式化代码
get        下载安装包和依赖
install    编译安装包和依赖
list       列出包
run        编译运行程序
test       测试包
version    显示go版本信息
vet        代码问题分析工具
```

执行工具：

```bash
$ go [commant] // 执行工具
$ go help [commant] // 查看工具文档
```

## 2.1 env

显示Go环境变量。

```bash
$ go env # 显示所有Go环境变量
$ go env GOPATH # 显示某个环境变量

$ go env -w GO111MODULE=on # 设置Go环境变量
$ go env -u GOPROXY # 取消env配置
```

**GOPATH**

环境变量 GOPATH 表示工作空间的根目录，默认在 ~/go，其中有如下子目录：

```
GOPATH/
    src/ 包的源文件
    bin/ 生成的可执行程序
    pkg/ 编译生成的文件
        mod/ go mod下载的第三方包存放路径
```

go mod 下载的包存放在 GOPATH/pkg/mod/ 中，例如：包的 github 地址为 https://github.com/google/pprof，下载后包存放路径为 ~/go/pkg/mod/github.com/google/pprof@v0.0.0-20230705174524-200ffdc848b8，最底层路径中带有下载的版本号。

**GOROOT**

环境变量 GOROOT 指定Go发行版的根目录，也就是Go安装的目录，其中提供所有标准库的包。

**GOPROXY**

包下载的代理服务器地址，默认为 https://proxy.golang.org,direct

可以换成国内其他地址或公司内部地址，提升包的下载速度。

**GOSUMDB**

拉取包后校验和数据库的地址，确保拉取到的依赖包内容未经修改。

**GOPRIVATE**

指明私有仓库，不通过代理服务器拉取和校验，可以设置多个，通过逗号分隔。

**GOBIN**

设置 go install、go build 可执行文件的存放文件。默认情况该变量是空的，此时可执行文件会放在 $GOPATH/bin 中。

**GO111MODULE**‌

是否开启 go module 包管理。

**GOOS**

操作系统，如 windows、linux、darwin。

**GOARCH**

处理器架构，如 amd64、arm64。

**GOVERSION**

Go 安装版本。

**GOINSECURE**

用于指定特定域名使用 http 而非 https 拉取，通常在私有部署的 gitlab 只使用 http 时设置。

```bash
go env -w GOINSECURE gitlab.xxx.com
```

**GOMAXPROCS**

Go 程序可以执行的最大 CPU 数量，默认为机器的 CPU 核心数量。

## 2.2 version

显示Go版本。

```bash
# 当前环境 go 版本
$ go version

# 查看可执行文件的 go 版本
$ go version -v <exe>

# 查看可执行文件的 go 版本、依赖库版本、构建参数
$ go version -m <exe>
```

## 2.3 run

将一个或多个.go为后缀的源文件进行编译、链接，然后运行生成的可执行文件。

```bash
$ go run helloworld.go
$ go run helloworld.go one two three # 带有运行参数
```

## 2.4 build

将源文件编译输出成一个可执行的程序，该可执行文件可以直接执行。

-o 指定生成的程序文件名

-n 仅显示编译命令，但是不执行

-v 打印正在编译的包名

-x 显示正在执行的命名

-gcflags 编译器参数

- -B 禁用越界检查
- -N 禁用优化
- -l 禁用内联
- -u 禁用unsafe
- -S 输出汇编代码
- -m 输出优化信息

-ldflags 链接器参数

- -s 禁用符号表
- -w 禁用DRAWF调试信息
- -X 设置字符串全局变量值 -X ver="0.99"
- -H 这只可执行文件格式 -H windowsgui

```bash
$ go build helloworld.go
$ ./helloworld
```

## 2.5 clean

删除生成的对象文件和可执行文件。

-i 清理go install安装的文件

-r 递归清理所有依赖包

-x 显示正在执行的清理命令

-n 仅显示清理命令，但不执行

```bash
$ go clean
```

## 2.6 test

运行测试。

```bash
$ go test
```

## 2.7 list

列出可用的包。

```bash
$ go list
$ go list java... # 使用...作通配符匹配子串
$ go list -m all # 所有已安装的依赖包
$ go list -m go list -m github.com/redis/go-redis/v9 # 某个指定的依赖包
```

## 2.8 get

下载依赖包，将会被放到 GOPATH/pkg 里，目前支持从 Github、BitBucket、Google Code、Launchpad 等代码管理平台获取远程代码包。

从 Go 1.16 开始，go get 命令已经不再支持在不使用模块的情况下使用，应该使用 go install 命令。

-d 只下载不安装

-u 更新包和它的依赖包

-f 带有-u时生效，不需要验证import的每一个包都获取了

-t 同时下载运行测试所需的包

-v 显示执行的命令

```bash
# 下载最新版本
go get github.com/golang/lint/golint
# 下载指定版本
go get github.com/golang/lint/golint@v1.0.3
```

## 2.9 install

编译运行依赖包。

先将对应版本的依赖包下载到 GOPATH/pkg 下，然后进行编译安装，将可执行文件放到 GOPATH/bin 下。

```bash
# 安装最新版本
go install <package>
go install <package>@latest
# 安装指定版本
go install <package>@<version>
```

## 2.10 fmt

运行 gofmt 进行代码格式化。

```bash
$ go fmt # 格式化当前目录下所有go文件，不包含子目录内的
$ go fmt <file> # 格式化某个go文件
```

go fmt 实际执行的可执行程序是 gofmt，在不指定具体文件时，是会对每个要执行的go文件调用 gofmt的。以下是 gofmt 的用法和参数：

-n 显示要执行的命令而不实际执行

-l 将不符合格式化规范的源码文件绝对路径打印到标准输出，标准输出的go文件名就是执行了格式化的，其他未输出的就是不需要执行格式化的go文件

-w 将格式化的内容写入文件

-s 简化文件的代码

-d 只把改写前后的对比内容打印到标准输出

-e 打印所有语法错误到标准输出

-comments 是否保留注释，默认隐式使用，默认值为true

-tabwidth 设置缩进的空格数量，默认值为8

-tabs 是否使用\t代表空格表示缩进，默认隐式使用，默认值为true

```bash
$ gofmt <file> # 对go文件格式化
```

## 2.11 vet

运行代码问题分析工具，对代码进行静态分析，检查常见错误和潜在问题。

检查的项目包括有变量未使用、printf 函数格式化字符串参数类型不匹配、不必要的导入包、数组索引越界等。

```go
go vet # 检查目录下所有go文件
go vet <file> # 检查指定go文件
```

-v 显示详细检查信息

-c 不显示警告只显示错误

-all 检查所有可用的检查项目

# 3. 参考

- [《Go程序设计语言》](https://book.douban.com/subject/27044219/)
- [go tool 简介 - 知乎](https://zhuanlan.zhihu.com/p/119256899)

