# 1. Markdown介绍

Markdown是一种轻量级标记语言，使用纯文本编写有简单格式的文档。Markdown可以支持标题、图片引用、图表、数学表达式，甚至可以用来画流程图。

Markdown文件后缀为.md或.markdown，常用于Github的readme文件和文档编写等，编辑软件推荐Typora，或者Visual Studio Code安装插件Markdown All in One也非常好用。



# 2. 文本与标题

纯文本直接输入，用\转义特殊字符。

### 一至六级标题

```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
正文
```

效果如下：

![markdown_grammar_title](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_title.png)

### 目录

根据Markdown文件中的标题结构产生层级目录

```
[toc]
```

效果如下：

![markdown_grammar_toc](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_toc.png)

### 段落

```
行末用两个以上空格表示换行
line1<space><space>
line2

用空行换行
line1

line2
```

### 分隔线

```
***
---
___
```

效果如下：

![markdown_grammar_line](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_line.png)

### 文本格式

```
斜体
*text*
_text_

粗体
**text**
__text__

粗斜体
***text***
___text___

删除线
~~text~~

下划线
<u>text</u>
```

效果如下：

![markdown_grammar_font_style](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_font_style.png)

### 注释

```
text可以为任何文本，同一行后面的文本为注释
[text]:commenttext

本行后没有文本时，下一行为注释
[text]:
commenttext
```

### 脚注

```
引用脚注，指向脚注名字时会显示文本
text [^1]

声明脚注，带有一个名字和一些文本
[^1]: some comment
```

效果如下：鼠标移动至1会显示文本

![markdown_grammar_footnote](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_footnote.png)

# 3. 列表

### 无序列表

```
* text
+ text
- text
```

### 有序列表

```
1. text
2. text
3. text
```

### 列表嵌套

可以将无序列表和有序列表混用

```
* text
	1. text
	2. text
		* text
		* text
* text
```

效果分别如下：

![markdown_grammar_list](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_list.png)

### 勾选框

```
* [ ] text
* [x] text
```

效果如下：

![markdown_grammar_checklist](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_checklist.png)

# 4. 代码

### 引用

```
> text
>> 第一层嵌套
>>> 第二层嵌套
```

效果如下：

![markdown_grammar_quote](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_quote.png)

### 代码

```
`code`
```

### 代码段

前面需要一个空行，每行开头要缩进一层

```

	code
	code
```

可选：指定语言高亮

````
```c++
code
```
````

效果如下：

![markdown_grammar_code](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_code.png)



# 5. 链接

### 链接

```
显示的文本和指向的链接
[text](url)

将链接分开放到后面编号
[text][1]
[1]:url
```

效果如下：点击文本会打开链接

![markdown_grammar_link](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_link.png)

### 图片

```
图片地址可以是本机路径或网络地址
![text](address)

使用其他HTML标签属性如放大率
<img src="photo address" width="50%">
```



# 6. 表格

```
|title|title|title|
|-|-|-|
|text|text|text|
|text|text|text|
```

效果如下：

![markdown_grammar_table](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/encode/markdown_grammar_table.png)

### 表格对齐

在第二行加，控制对应列的属性

```
|:-| 左对齐
|-:| 右对齐
|:-:| 居中对齐
```

