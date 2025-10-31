# 1. 简介

Makefile 是一种自动化构建脚本，其中包含若干目标和对应的依赖及构建规则。

Makefile 文件默认文件名是 Makefile 或 makefile，make 命令会自动搜索本地目录中的这两个文件并执行，当然也可以通过参数来指定文件。

make 命令会自动找到当前目录的默认 Makefile 或参数指定的 Makefile，并执行相应规则的命令。

```bash
# 通用格式
[<env-vars>] make [<arg-vars>] [<options>] [<target>]

# 执行 Makefile 文件中的第一个规则
make

# 执行指定规则
make <target>

# 传递参数并执行
make argvar=foobar
```

支持的参数：

* -f file：指定 Makefile 文件；
* -C dir：指定工作目录，会将工作目录切换到该目录后执行 make，然后回到之前目录；
* -s：静默模式，不输出规则中的命令；
* -n：输出要执行的命令而不执行；
* -B：强制执行所有目标；
* -j n：指定并发数；

Makefile 中可以包含其他的 Makefile 文件，会讲包含的 Makefile 文件完全展开。包含的文件可以使用本文件中定义的变量。

```makefile
include <makefile>
```

Makefile 中的注释和 shell 一样，都是以 `#` 开头直至行末。

```makefile
# comment
```

# 2. 规则

Makefile 脚本之中包含着一条条的规则，每条规则的形式如下：

```makefile
<target>: <prerequisites>
<tab><command>
<tab><command>
...
```

* target 是需要构建的目标，可以是一个可执行文件，或者执行一系列命令的标签；
* prerequisites 是对于目标所依赖的文件或其他目标，如果依赖没有满足，则会先根据依赖对应的构建规则，生成依赖项，然后生成目标；
* command 是一个 shell 命令，以 tab 为开头，一个规则可以有多条命令，占多行或一行内以 && 分隔；

例子如下：

```makefile
main.o : main.c
	gcc -c main.c -o main.o
```

规则中可以使用 % 作为通配符，将每个 c 文件编译为目标文件，其中的 $< 和 $@ 则是 Makefile 中的自动变量。

```makefile
%.o: %.c
	gcc -c $< -o $@
```

.PHONY 用于声明伪目标，即声明某个目标是规则，而不对应实际的文件，避免存在同名文件时，判断文件存在而无法执行规则。

```makefile
.PHONY: clean all  # 声明文件中的以下目标是伪目标
```

# 3. 变量

Makefile 的变量可以用于存放命令、选项、文件名等，本质上保存了一个字符串。

变量的定义格式为：

```makefile
<var-name> <assignment> <var-value>
```

* var-name 是变量名，由字母、数字、下划线组成，不能以数字开头；
* assignment 是赋值运算符；
* var-value 是变量的值，如果包含空格则用引号括起来，注意变量赋值时最后的空格也会算为值；

变量的使用有两种方式：

```makefile
$(<var-name>)
${<var-name>}
```

例子如下：

```makefile
CC = gcc
CFLAGS = -Wall -g
MAIN_OBJS = main.o
$(MAIN_OBJS) : main.c
    $(CC) $(CFLAGS) -c main.c -o $(MAIN_OBJS)
```

变量赋值有几种方式：

`=` 赋值运算符会在 Makefile 全部展开后进行，对同一个变量多次 `=` 赋值会使其值变为最后一次的赋值。

```makefile
<var-name> = <var-value>

CC = gcc
CC2 = $(CC) # 值为 g++，因为 CC 最后的值是 g++
CC = g++
```

`:=` 赋值运算符的值和其在 Makefile 中的位置有关。

```makefile
<var-name> := <var-value>

CC := gcc
CC2 := $(CC) # 值为 gcc，根据此时的 CC 的值而定
CC := g++
```

`?=` 赋值运算符当变量未被赋值时才会赋值。

```makefile
<var-name> ?= <var-value>

CC ?= gcc # 赋值
CC ?= g++ # 已有值，不再赋值
```

`+=` 赋值运算符在已有值后面追加内容，中间补空格。

```makefile
<var-name> += <var-value>

CC = gcc
CC += -Wall # gcc -Wall
```

Makefile 支持如下自动变量：

`$@` 表示构建目标。

```makefile
# $@ 表示 main.o
main.o: main.c
	gcc -c main.c -o $@
```

`$*` 表示构建目标去掉后缀的部份。

```makefile
# $* 表示 main
main.o : main.c
	gcc -c $* -o $@
```

`$<` 表示第一个依赖文件。

```makefile
# $< 表示 main.o
main: main.o func.o
	gcc $< -o main
```

`$^` 表示所有依赖文件。

```makefile
# $^ 表示 main.o func.o
main: main.o func.o
	gcc $^ -o main
```

`$?` 表示比目标文件更新的所有依赖文件。

```makefile
# $? 表示所有依赖文件中比 main 更新的所有文件
main: main.o func.o
	gcc $? -o main
```

# 4. 函数

Makefile 提供了一些函数，函数的格式如下，函数和参数之间用空格分隔，参数之间用逗号分隔。

```makefile
$(<func-name> <arg1>, <arg2>, ...)
```

`abspath` 函数用于获取文件的绝对路径。

```makefile
BUILD_DIR := $(abspath ./build)
```

`addprefix` 函数给一系列字符串添加前缀。

```makefile
OBJS := main.o func.o
BUILD_DIR := ./build
BUILD_OBJS := $(addprefix $(BUILD_DIR)/, $(OBJS)) # 给OBJS的每个字符串都添加BUILD_DIR前缀
```

`addsuffix` 函数给一系列字符串添加后缀。

```makefile
OBJS := main func
BUILD_OBJS := $(addsuffix .o, $(OBJS)) # 给OBJS的每个字符串都添加.o后缀
```

`basename` 函数用于去掉文件名的后缀。

```makefile
OBJS = ./build/main.o ./build/func.o
BASE_OBJS = $(basename $(OBJS)) # 去掉.o后缀
```

`dir` 函数用于获取文件名中的目录部分。

```makefile
OBJS = ./build/main.o ./build/func.o
DIR_OBJS = $(dir $(OBJS)) # 获取OBJS各个文件的所在目录
```

`notdir` 函数用于获取文件名中的非目录部分。

```makefile
OBJS = ./build/main.o ./build/func.o
NOTDIR_OBJS = $(notdir $(OBJS)) # 获取OBJS各个文件的文件名
```

`shell` 函数用于执行 shell 命令。

```makefile
CWD = $(shell pwd) # 执行pwd命令并将输出值赋值变量
```

`wildcard` 函数用于获取符合通配符的所有文件。

```makefile
CFILES = $(wildcard *.c) # 获取当前目录所有.c文件
```

`patsubst` 函数用于对每个文件进行模式替换。

```makefile
SRCS = main.c utils.c helper.c
OBJS = $(patsubst %.c,%.o,$(SRCS)) # 将每个.c文件替换为.o文件
```

# 5. 条件分支

Makefile 可以使用 `ifeq`、`ifneq`、`ifdef`、`ifndef` 等关键字来控制条件分支。

* ifeq：是否相等；
* ifneq：是否不等；
* ifdef：是否有值；
* ifndef：是否没有值；

以下例子中，ifeq 判断变量和值是否相等，来决定走哪个条件分支，以 endif 结束 if 语句。

```makefile
ifeq ($(CC), gcc)
	LIBS = $(LIBS_FOR_GCC)
else
	LIBS = $(NORMAL_LIBS)
endif
```

# 6. 参考

* [Makefile Tutorial By Example](https://makefiletutorial.com/)
* [Makefile 常用语法 - USTC CECS 2023](https://soc.ustc.edu.cn/CECS/lab1/makefile/)

