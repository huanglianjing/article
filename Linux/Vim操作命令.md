# 1. Vim介绍

Vim是一个命令行端的文本编辑器，由Vi发展而来，通过各种命令能很方便地对文本进行编辑。

我通常习惯加上`alias vi='vim'`，然后直接用vi来打开vim。



# 2. 打开

使用Vim的第一步是打开它：

```bash
# 打开空文件
$ vi

# 打开一个文件
$ vi file

:e [path] 文件浏览器，可以指定打开路径，或直接打开某个文件
```



# 3. 退出

对于很多第一次使用Vim的人，往往会有不知道如何退出编辑器的尴尬。

以下是几种常用的退出方式：

```
:w   保存，但不退出
:q   直接退出，若有改动则会报错
:qa  退出多个窗口，a表示all，也可用于其他退出命令
:wq  保存并退出
:q!  退出且不保存，会舍弃掉未保存的改动
:wq! 强制保存并退出
:x   退出，若有改动时才保存
```



# 4. 配置

### 保存配置

Vim的配置保存在**/etc/vim/vimrc**或**~/.vimrc**中，这样每次打开Vim之后保存的配置就生效了。

以下是我的Vim配置：

```properties
syntax on
set number
set nocompatible
set showcmd
set t_Co=256
set tabstop=4
set softtabstop=4
set shiftwidth=4
set cursorline
set showmatch
set hlsearch
set incsearch
set ignorecase
set nowrapscan
set noswapfile
set autochdir
set list
set listchars=tab:>-,trail:-
set shortmess=atI
set backspace=indent,eol,start
if &diff
    colors blue
endif
```

### 输入配置

在命令模式下直接按`:`加上命令输入配置，配置仅在打开Vim期间生效，关闭后失效。

```
:set settingname 设置某个配置
```

一些常用配置：

```
:set number 显示行号
:set nonumber 关闭行号
:set paste 粘贴模式，不缩进
:set nopaste 关闭粘贴模式
```



# 5. 通用命令

一些适用于所有命令的通用命令：

```
esc 取消正在输入的命令
. 重复上一命令
数字+命令 重复该数字次数的命令，如3j表示3次下，10dd表示删除10行
```



# 6. 光标移动

### 基本移动

```
j 下
k 上
h 左
l 右
space 后一个字符
backspace 前一个字符
w 后一个词
b 前一个词
0 行首
^ 本行第一个非空字符
$ 行末
```

### 换行

```
H 屏幕内首行
L 屏幕内末行
M 屏幕内中间行
:n 第n行
:$ 最后一行
enter 下一行
gg 第一行
G 最后一行
ngg 第n行
nG 第n行
n| 第n列
```

### 换页

```
ctrl+f 下一页
crtl+b 上一页
```

### 跳转

```
[[ 上一个函数
]] 下一个函数
{ 上一段，按空行分隔
} 下一段，按空行分隔
```



# 7. 编辑

### 编辑模式

从命令模式进入编辑模式后，按任何按键就是输入内容而非命令了，左下角会显示`-- INSERT --`。

直到按esc回到命令模式。

```
i 插入模式
a 后一位插入
I 行首插入
A 行末插入

R 替换模式，输入内容替换当前字符而为插入
```

### 撤销和恢复

```
u 撤销
ctrl+r 恢复
```

### 复制粘贴

```
yy 复制行，可以在前面加数字复制多行
Y 复制行
p 粘贴在本行后
P 粘贴在本行前，可以在前面加数字粘贴多遍

J 下一行合并到本行
```

### 删除

```
x 删除后一个字符
X 删除前一个字符
d0 删除从光标到行首
d^ 删除从光标到行首非空字符
d$ 删除从光标到行末

dd 删除行，可以在前面加数字删除多行
dgg 删除本行到第一行
dG 删除本行到最后一行
ggdG 删除所有行
Gdgg 删除所有行
```

### 缩进

```
>> 增加缩进
<< 减少缩进
== 根据情况缩进
```



# 8. 查找和替换

### 查找

```
/key 从上往下查找，可以使用正则表达式
?key 从下往上查找
* 查找当前单词
n 下一个匹配
N 上一个匹配
```

### 替换

```
:%s/old/new/g 全文替换
:%s/old/new/gi 全文替换，忽略大小写
:%s/old/new/gc 全文替换，需要确认

:s/old/new/ 本行替换一次
:s/old/new/g 本行替换
:n1,n2s/old/new/g 第n1到n2行替换
:1,$s/old/new/g 全部行替换

:g/string/d 删除所有含string的行
:v/string/d 删除所有不含string的行
```

### 数字增减

```
ctrl+a 当前数字+1
ctrl+x 当前数字-1
```



# 9. 选择

进入选择模式，选择一块区域的文本，左下角会显示`-- VISUAL --`。

```
v 字符选择
V 行选择
ctrl+v 长方形区域选择
```

然后作出操作

```
y 复制
d 删除
u 转小写
U 转大写
```



# 10. 多文件

### 多文件

```
$ vi file1 file2 打开多个文件，同时只显示其中一个文件
:n 编辑下一个文件
:N 编辑上一个文件
:files 列出打开的文件
```

### 多窗口

```
$ vi -o files 上下排列打开多个文件
$ vi -O files 左右排列打开多个文件

:sp 在上方打开同一文件
:sp file 在上方打开文件
:vsp 在左方打开同一文件
:vsp file 在左方打开文件

ctrl+w,ctrl+w 光标移到下一窗口
ctrl+w,h/j/k/l 光标移到旁边窗口
ctrl+w,= 使窗口大小平均
ctrl+w,+/- 当前窗口调整大小
```

### 文本比较

```
# 打开多个文件比较
$ vi -d file1 file2
$ vimdiff file1 file2

[c 上一个差异
]c 下一个差异
dp 差异点复制到对面
do 差异点从对面复制
zo 展开折叠的相同行
zc 折叠相同行

:diffupdate 重新比较差异
```

