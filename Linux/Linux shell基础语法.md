# 1. 简介

shell 是 Linux 中用于访问和控制操作系统的程序，提供了交互式的界面，也可以通过编写 shell 脚本文件，直接执行文件。

以下主要介绍 Linux 众多 shell 中的 bash（Bourne Again Shell），它的位置在 /bin/bash 或 /usr/bin/bash。

查看当前的 shell：

```bash
echo $0
```

如果系统登陆的默认 shell 不是 bash，可以输入 `bash` 命令直接切换到 bash。

# 2. 脚本

我们编写的脚本文件习惯扩展名 sh，如 `test.sh`，但不是必须的，只要它是有执行(x)权限的文本文件，就可以被当作 shell 脚本执行。

```bash
# 执行脚本
./test.sh

# 指定 shell 执行脚本
bash test.sh
```

shell 脚本通常习惯在开头加上以下内容，表示执行脚本选择的 shell。

```bash
#!/bin/bash
```

默认情况下，以上的执行方式，执行 shell 脚本会启动一个新的 shell 子进程来执行脚本的命令，脚本内无法看到当前的变量。

可以在执行的时候不启动 shell 子进程，直接在当前环境执行，就可以访问和修改当前环境的所有变量、环境、工作目录。

```bash
source test.sh
. test.sh # . 是 source 的简写
```

也可以通过将变量导出为环境变量的方式，使变量被当前 shell 和所有子进程可见。

```bash
export var=123
```

脚本调用可以传参，并在脚本中获取。

```bash
# 调用传参
./test.sh good 12

# 获取参数
$# # 参数个数
$0 # 程序名称
$1 # 第1个参数
$2 # 第2个参数，依次类推，获取不到返回空
$* # 所有参数
$@ # 所有参数，加双引号输出参数
```

脚本中还可以获取脚本执行的信息和状态。

```bash
$$ # 当前进程id
$! # 后台运行的最后一个进程id
$- # shell当前选项
$? # 上一条命令的退出状态，0表示成功，其他表示错误
```

# 3. 注释

单行注释以 # 开头，直到一行的结尾。

```bash
# 注释
echo $var # 注释
```

多行注释，开始和结束标识符相等的范围内为注释，标识符不是固定的。

```bash
:<<EOF
注释
注释
EOF

:<<'
注释
注释
'

:<<!
注释
注释
!

# 简化版本，表示符必须为单引号
: '
注释
注释
'
```

# 4. 变量

变量通过等号赋值的方式定义，等号左右不能有空格。

字符串变量：

```bash
var=good
var='good' # 支持内部空格
var="good" # 支持内部空格和引用变量，推荐使用

# 获取字符串长度
${#var}

# 获取字符串子串，指定offset和limit
${var:1:4}
```

整数变量：

```bash
declare -i var=123
```

数组变量：

```bash
# 一起定义
var=(1 2 3 4 5)

# 一个个下标定义
var[0]=43
var[1]=3453
var[2]=100

# 读取元素
${var[1]}
${var[@]} # 获取所有元素

# 从a到b的整数组成的数组
{a..b}
$(seq a b)
```

关联数组变量：

```bash
# 定义
declare -A site=(["google"]="www.google.com" ["taobao"]="www.taobao.com")

# 设置键值
site["baidu"]="www.baidu.com"

# 读取元素
${site["baidu"]}
```

使用变量时，在变量名前加上 $，花括号是可选的。

```bash
$var
${var} # 可选，可以和后面的内容区分
```

变量默认为本地变量，本地变量只能在当前 shell 使用。

环境变量是导出的变量，对当前 shell 和所有子进程可见，但子进程中的修改不会影响父 shell。

```bash
# 赋值并导出
export var=123

# 先定义后导出
var=123
export var
```

环境变量可以取消导出。

```bash
# 彻底删除变量
unset var

# 取消导出，但仍是本地变量
export -n var
```

将变量定义为只读变量后，无法更改它的值，也无法删除。

```bash
# 直接定义
readonly var=123

# 先定义后设为只读
var=123
readonly var
```

declare 命令用于声明变量和设置变量属性。

```bash
# 整数
declare -i num=100

# 索引数组
declare -a arr=("a" "b")

# 关联数组
declare -A dict=([key1]="val1")

# 只读
declare -r const=12345

# 环境变量
declare -x var="global"

# 在函数内声明全局变量
declare -g var="global"

# 变量值被赋值时自动转为小写
declare -l var="hello"

# 变量值被赋值时自动转为大写
declare -u var="hello"

# 声明为引用，修改ref会同时修改var
declare -n ref=var

# 为变量添加trace属性，调试用
declare -t var

# 显示变量的定义
declare -p var
```

# 5. 运算符

通过 expr 计算表达式并求值，表达式可以是算数运算、字符串操作、正则匹配。以下出现的部份运算符需要加 \ 转义。

成功时输出结果到标准输出，失败时返回非零状态码，通过 $? 检查 0 或 1 表示逻辑真假。

```bash
expr <expression>
```

将 expr 表达式计算的值赋值给变量：

```bash
var=$(expr 1 + 2)
```

算数运算，仅支持整数运算，运算符和数字之间要用空格隔开。

```bash
# 加法
expr 5 + 3

# 减法
expr 10 - 2

# 乘法
expr 2 \* 3

# 除法
expr 10 / 2

# 取模
expr 10 % 3
```

字符串操作。

```bash
# 计算长度
expr length $var
expr $var : '.*'

# 提取子串，指定offset和limit，下标从1开始
expr substr $var 2 10

# 查找子串位置，下标从1开始
expr index $var "substr"
```

逻辑比较，返回 0 或 1。

```bash
# 字符串相等
expr "a" = "a"

# 字符串不等
expr "a" != "b"

# 字符串大小比较
expr "a" \< "b"
expr "a" \> "b"

# 逻辑与
expr 1 \& 1

# 逻辑或
expr 0 | 1
```

正则匹配，仅用于简单场景。

```bash
# 计算从开头匹配的长度
expr "abc123" : '[a-z]*'

# 提取匹配的子串，两个$为固定前后包围
expr "abc123" : '$[a-z]*$'
```

expr 适用于兼容较旧的环境，现代新版本系统建议使用替换方案：

```bash
# 整数运算
let "var = 3 + 5"
(( var = 3 + 5 )) // let的简化版
var=$((3 + 5))

# 字符串操作
${#var} # 计算长度
${var:1:4} # 获取子串
```

数字运算符。

```bash
# 相等
if [ $a == $b ]
if [ $a -eq $b ]

# 不等
if [ $a != $b ]
if [ $a -ne $b ]

# 大于
if [ $a -gt $b ]
(( $a > $b ))

# 小于
if [ $a -lt $b ]
(( $a < $b ))

# 大于等于
if [ $a -ge $b ]

# 小于等于
if [ $a -le $b ]

# 自增
let num++
((num++))

# 自减
let num--
((num--))
```

字符串运算符。

```bash
# 相等
if [ $a = $b ]

# 不等
if [ $a != $b ]

# 是否为空
if [ -z $a ]

# 是否非空
if [ -n $a ]
if [ $a ]

# 正则表达式匹配
if [ $a =~ [0-9]+ ]
```

布尔运算符，返回 true 或 false。

```bash
# 非运算
if [ ! false ]

# 与运算
if [ $a -gt 50 -a $b -lt 100 ]
if [ $a -gt 50 && $b -lt 100 ]

# 或运算
if [ $a -lt 50 -o $b -gt 100 ]
if [ $a -lt 50 || $b -gt 100 ]
```

文件运算符。

```bash
# 文件存在
if [ -e $file ]

# 普通文件
if [ -f $file ]

# 目录
if [ -d $file ]

# 文件不为空
if [ -s $file ]

# 文件是socket
if [ -S $file ]

# 文件是符号链接
if [ -L $file ]

# 文件可读
if [ -r $file ]

# 文件可写
if [ -w $file ]

# 文件可执行
if [ -x $file ]
```

# 6. 语句

if 条件判断语句，可以根据情况搭配 elif、else，最后必须要用 fi 结束。

```bash
if condition
then
  command
fi

if condition
then
  command1
else
  command2
fi

if condition1
then
  command1
elif condition2 
then 
  command2
else
  command3
fi

# 可以写成一行，原本每一行的结尾加上;
if [ condition ]; then command; fi
```

for 循环语句遍历数组或条件并执行命令。

```bash
for var in array
do
  command
done

# 遍历某一范围的整数
for i in $(seq 0 99)
for i in {0..99}
for ((i=0; i<100; i++))

# 无限循环
for (( ; ; ))
```

while 循环语句重复执行命令，直到条件不成立。

```bash
while condition
do
  command
done

# 无限循环
while :
while true
```

until 循环语句执行命令直至条件为真。

```bash
until condition
do
  command
done
```

判断语句有几种写法，if、for、while、until 同理。

```bash
if [ $a -gt $b ]
if (( $a > $b ))
if test $a -gt $b
```

break 语句可以跳出以上各个循环语句，continue 语句可以跳过当前循环。

case 多选择语句，匹配变量对应值的条件并执行，;; 表示跳出语句。

```bash
case $var in
  1)
  command1
  ;;
  2)
  command2
  ;;
esac
```

# 7. 函数

定义函数：

```bash
function func()
{
  command
}

# function可忽略
func()
{
  command
}
```

函数内通过 $1 $2 等获取参数。

函数可以通过 return 返回值，如果不返回，则以最后一条命令的运行结果作为返回值。

函数调用：

```bash
# 调用无参数函数
func

# 调用函数并带参数
func "good" "morning"
```

# 8. 重定向

文件描述符中，0表示标准输入（STDIN），1表示标准输出（STDOUT），2表示错误输出（STDERR）。

命令的输入可以重定向从文件输入，输出可以重定向到文件。

```bash
# 输入重定向到文件
command < file

# 输出重定向到文件，将覆盖文件原有内容
command > file

# 输出重定向到文件，追加到末尾
command >> file

# 重定向输入和输出
command < file1 > file2

# 只将标准错误重定向到文件
command 2>> file

# 将标准输出和标准错误一起重定向到文件
command 2>&1 >> file

# 将输出丢弃
command 2>&1 > /dev/null
```

