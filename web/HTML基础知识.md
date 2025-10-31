# 1. HTML简介

HTML是用来描述网页的一种语言，全称超文本标记语言(Hyper Text Markup Language)，它是一种标记语言，使用标记标签来描述网页，它的语法规则可以定义图片、表格、链接等。

## 1.1 一个基础例子

一个基础的HTML例子：

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>page name</title>
</head>
<body>
	<h1>title 1</h1>
	<h2>sub title</h2>
	<p>paragraph text</p>
	<h1>title 2</h1>
	<p>paragraph text</p>
</body>
</html>
```

整个文件由html标签包着，其中由分为首部和主体两部分，分别用head标签和body标签包裹。

首部告诉浏览器关于网页的信息，如页面标题，首部包括\<head\>和\</head\>之间的所有内容。其中的title标签定义了网页的标题，meta标签指定了字符编码。

主体包含网页的所有内容和结构，也就是在浏览器直接看到的部分，主体包括\<body\>和\</body\>之间的所有内容。其中的h1标签和h2标签分别是一级标题和二级标题，p标签则是段落。

保存为HTML文件，在浏览器打开后，页面效果如下：

![html_example](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/web/html_example.png)

此外，还可以在HTML的首部里增加一些样式，就是style标签。style标签有一个可选的属性type，一般指定为"text/css"。

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>page name</title>
	<style type="text/css">
		body {
			background-color: #d2b48c;
			margin-left: 20%;
			margin-right: 20%;
			border: 2px dotted black;
			padding: 10px 10px 10px 10px;
			font-family: sans-serif;
		}
	</style>
</head>
<body>
	<h1>title 1</h1>
	<h2>sub title</h2>
	<p>paragraph text</p>
	<h1>title 2</h1>
	<p>paragraph text</p>
</body>
</html>
```

style标签中的body表示这段配置应用于主体中的body标签。

保存刷新后看到页面效果：

![html_example_1](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/web/html_example_1.png)

可以看到通过增加style标签，给页面加了背景颜色、边框、字体。

## 1.2 标签

HTML标签是由尖括号包围、成对出现的关键词，如\<p> 和 \</p\>，分别为开始标签和结束标签。HTML标签大小写不敏感，但推荐使用小写。

## 1.3 元素

HTML元素是开始标签到对应的结束标签的所有代码，HTML元素可以嵌套包含其他HTML元素。

html元素定义整个HTML文档，head元素定义文档的首部，body元素定义文档的主体。空元素没有内容，在开始标签中关闭，如br元素应当在开始标签添加斜杠：\<br /\>。

## 1.4 属性

HTML属性在开始标签中设置，以名称/值对name="value"的形式出现。属性和属性值大小写不敏感，但推荐小写。

以下是大多数HTML元素的属性：

1. class：规定元素的类名（classname），不同的元素可以有相同的类名
2. id：规定元素的id，元素的id在HTML文档中是唯一的，其他a标签可以通过`href="url#label"`定位到此网页的该元素处
3. style：规定元素的行内样式（inline style）
4. title：规定元素的额外信息

以下是几个属性的实例：

1. HTML链接的地址：

   ```html
   <a href="http://www.google.com">This is a link</a>
   ```

2. HTML标题的对齐方式：

   ```html
   <h1 align="center">
   ```

3. HTML表格的边框信息：

   ```html
   <table border="1">
   ```

## 1.5 字符实体

若要在HTML文本中展示<和>，又不想和标签区分开，可以转化为字符实体，用来表示特殊字符。

| 特殊字符 | 字符实体 | 说明                                 |
| -------- | -------- | ------------------------------------ |
| 空格     | \&nbsp;  |                                      |
| <        | \&lt;    |                                      |
| >        | \&gt;    |                                      |
| &        | \&amp;   | 在&用于实体字符时转化，一般可直接用& |
| "        | \&quot;  |                                      |
| '        | \&apos;  |                                      |

## 1.6 标准

现在最新的标准是HTML5，可以通过 [The W3C Markup Validation Service](https://validator.w3.org/) 检查网页是否符合HTML标准。



# 2. 标签

## 2.1 首部

html标签包裹整个HTML文件，其中又分为head元素和body元素表示首部和主体。

```html
<html>
<head>
</head>
<body>
</body>
</html>
```

以下是几个专门用于首部的标签。

**标题**

title标签指定网页标题。

```html
<title>page name</title>
```

**默认链接**

base标签为所有链接规定默认地址。

```html
<base href="http://www.google.com/" />
<base target="_blank" />
```

**外部资源**

link标签定义外部资源。

```html
<link rel="stylesheet" type="text/css" href="mystyle.css" />
```

**样式**

style标签定了HTML文档的样式信息。

```html
<style type="text/css"></style>
```

**数据**

meta标签定义元数据，如页面描述、修改时间、字符编码等。

```html
<meta charset="utf-8">
```

## 2.2 基础标签

**注释**

```html
<!-- This is a comment -->
```

**标题**

通过 h1 - h6 标签进行定义不同等级的标题，h1最大，h6最小。

```html
<h1>This is a heading</h1>
<h2>This is a heading</h2>
<h3>This is a heading</h3>
```

**段落**

浏览器会忽略HTML文档中的制表符、回车和大部分空格，根据标记来确定换行或分段。

```html
<p>This is a paragraph.</p>
```

**换行**

显示页面时，浏览器会移除源代码多余的空格和空行，连续的空格或空行会被算作一个空格，因此需要用此标签来换行。

```html
<br />
```

**水平线**

创建水平线用于分割内容。

```html
<hr />
```

**链接**

href属性指定链接地址，标签中间为显示的文本。

target属性定义链接文档的显示位置，默认在当前窗口打开，`target="_blank"`表示在新窗口打开。

id属性或name属性定义锚（anchor），id="label"，在同一文档创建链接`<a href="#label">`，或者其他页面创建链接`<a href="url#label">`，点击后直接跳转到该锚。

将显示文本替换为img元素，可以展示一个可以点击的图像。

```html
<a href="url">text</a>
```

**图像**

属性指定了图像的名称和尺寸。

src属性指定图片地址，可以是本地图片地址或网络图片地址。

width、height属性指定图片宽高，默认为按实际图片大小显示，可以为具体像素值也可以为百分比，如`width="300"`、`width="300px"`、`width="50%"`。

alt属性定义替换的文本，无法载入图片时显示。

```html
<img src="abc.jpg" width="104" height="142" />
```

## 2.3 文本格式

**粗体文本**

```html
<b>text</b>
```

**斜体字**

```html
<i>text</i>
```

**强调**

```html
<em>text</em>
```

**大号字**

```html
<big>text</big>
```

**小号字**

```html
<small>text</small>
```

**下标字**

```html
<sub>text</sub>
```

**上标字**

```html
<sup>text</sup>
```

**删除线**

```html
<del>text</del>
```

**下划线**

```html
<ins>text</ins>
```

**代码格式**

```html
<code>a = 1;</code>
```

## 2.4 引用

**短引用**

单行的引用，会用引号包围。

```html
<q>text</q>
```

**长引用**

会对整个元素进行缩进。

```html
<blockquote cite="http://www.worldwildlife.org/who/index.html">
五十年来，WWF 一直致力于保护自然界的未来。
WWF 工作于 100 个国家，并得到美国一百二十万会员及全球近五百万会员的支持。
</blockquote>
```

**缩略词**

对缩写进行标记，为浏览器、搜索引擎提供信息。

```html
<abbr title="World Health Organization">WHO</abbr>
```

**联系信息**

定义文档或文章的联系信息（作者/拥有者）。

```html
<address>
	Written by Donald Duck.<br> 
	Visit us at:<br>
	Example.com<br>
	Box 564, Disneyland<br>
	USA
</address>
```

**著作标题**

```html
<cite>The Scream</cite>
```

## 2.5 表格

一个表格用table标签定义，每一行用tr标签定义，行的每个单元格用td标签定义。第一行表头可用th标签定义，显示为粗体居中。

数据单元格可以包含文本、图片、列表、段落、表单、水平线、表格等等。

table标签的border属性定义边框。

一个表格的例子：

```html
<table>
	<tr>
		<th>Heading</th>
		<th>Another Heading</th>
	</tr>
	<tr>
		<td>row 1, cell 1</td>
		<td>row 1, cell 2</td>
	</tr>
	<tr>
		<td>row 2, cell 1</td>
		<td>row 2, cell 2</td>
	</tr>
</table>
```

## 2.6 列表

不同的列表可以嵌套使用，即在li元素中定义新的列表。

li = list item

ul = unordered list

ol = ordered list

**无序列表**

用圆点列出每个项目。

```html
<ul>
	<li>Coffee</li>
	<li>Milk</li>
</ul>
```

也可以直接用li标签包裹每个元素：

```html
<li>Coffee</li>
<li>Milk</li>
```

**有序列表**

用数字列出每个项目。

```html
<ol>
	<li>Coffee</li>
	<li>Milk</li>
</ol>
```

**定义列表**

用给定文本列出每个项目，列表的每一项都有dt标签和dd标签。

```html
<dl>
	<dt>Coffee</dt>
	<dd>Black hot drink</dd>
	<dt>Milk</dt>
	<dd>White cold drink</dd>
</dl>
```

## 2.7 块

使用div元素定义块，用于组合其它HTML元素，以对文档布局，将文档分割为独立的分区或节。

用id或class属性标记div元素，和css一起使用，用于对内容块设置样式。

```html
<div style="color:#00FF00">
	<h3>This is a header</h3>
	<p>This is a paragraph.</p>
</div>
```

用span元素来组合行内的文本，再通过样式来格式化它们。span标签本身没有固定的格式表现，需要对它应用样式。

```html
<p><span>some text.</span>some other text.</p>
```

## 2.8 内联框架

iframe元素用于在网页内显示网页。

属性width和height规定框架的宽度和高度，默认单位为像素，也可以是百分比。

```html
<iframe src="url"></iframe>
```

## 2.9 JavaScript

在head元素或body元素内，使用script标签定义JavaScript脚本。

```html
<script>
	document.write("Hello World!");
</script>
```

也可以通过src属性，使用外部脚本文件。

```html
<script src="./js/example.js"></script>
```

## 2.10 表单

使用form标签定义HTML表单，收集用户输入。

表单元素内，input标签定义输入，其中`<input type="text">`是输入文本框，`*<input type="radio">`是单选按钮，`<input type="submit">`是提交按钮。

如下例子：

```html
<form>
	input 1:<br>
	<input type="text" name="input 1"><br />
	input 2:<br>
	<input type="text" name="input 2"><br />
	<input type="radio" name="sex" value="male" checked>Male<br />
	<input type="radio" name="sex" value="female">Female<br />
	<input type="submit" value="submit">
</form>
```

在网页中显示如下：

![html_form](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/web/html_form.png)

点击提交按钮，会讲上面文本框和单选按钮输入的key=value对，传给网页参数发送请求，例如会发送一个请求`a.html?input+1=val1&input+2=val2&sex=male`。

form标签的action属性定义提交表单时执行的动作，如`<form action="action_page.php">`将会指定某个服务器脚本来处理被提交的表单，不指定action属性则设置为当前网页处理表单。

form标签的method属性规定提交表单用的HTTP方法为GET或POST，如`<form action="action_page.php" method="GET">`指定为GET方法。

## 2.11 画布

canvas元素使用JavaScript在网页上绘制图像。



# 3. HTML5

## 3.1 语义元素

非语义元素div和span，无法提供关于内容的信息，通过id和class包含信息，如`<div class="header">`和`<div id="footer">`。

HTML5提供的语义元素能清楚地描述意义，明确表示网页的不同部分，可以将上面的换为`<header>`和`<footer>`。

HTML5语义元素对应的网页结构：

![html5_semantic_element](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/web/html5_semantic_element.png)

**header**

文档的页眉。

**nav**

定义导航链接集合。

**main**

定义文档的主内容。

**section**

文档中的节，是有主题的内容组，通常具有标题。

**article**

独立的自包含的内容，常用于论坛、博客、新闻。

section元素和article元素是可以包含对方或者自包含的。

**aside**

除了页面主内容外的内容，与周围内容相关，如侧栏。

**footer**

文档或节的页脚，通常包含作者、版权信息、联系信息等。



# 4. 样式

可以通过首部的style标签，将样式添加到HTML元素中，或者通过CSS进行定义。

## 4.1 内部样式表

在HTML首部添加style标签，给指定元素定义样式。以下例子分别给HTML主体中的body元素和p元素指定了样式。

```html
<head>
	<style type="text/css">
		body {background-color: red}
		p {margin-left: 20px}
	</style>
</head>
```

## 4.2 外部样式表

当同一个样式需要被应用到很多页面的时候，可以选择外部样式表，定义一个公共的css文件来杯多个HTML文件引用，在每个HTML文件首部引用该css文件。

```html
<head>
	<link rel="stylesheet" type="text/css" href="mystyle.css">
</head>
```

## 4.3 内联样式

当特殊的样式需要应用到个别元素时，就可以使用内联样式，为对应的元素添加style属性，只应用于该元素。

```html
<p style="background-color: green;">This is a paragraph.</p>
```

## 4.4 指定元素使用样式

style标签指定对特定类型标签应用样式：

```html
<head>
	<style type="text/css">
		p {
			background-color: #a0a0a0;
		}
	</style>
</head>
<body>
	<h1>title</h1>
	<p class="text" id="t1">paragraph text</p>
	<p class="text" id="t2">paragraph text</p>
	<p class="message" id="t3">paragraph text</p>
</body>
```

style元素内的css分为选择器和声明块两部分，如`p {background-color: #a0a0a0;}`中p为选择器，指定HTML文档主体中特定的元素，{}中的部分为声明块，为若干个设置。

css写成`p {}`表示对所有p元素设置样式。css还可以写成`.text {}`指定对所有类text设置样式，或者用`p.text {}`对p元素的类text设置样式。写成`#t1 {}`指定对特定id t1设置样式。



# 参考

- [《Head First HTML与CSS（第2版）》](https://book.douban.com/subject/25752357/)
- [W3school - HTML 教程](https://www.w3school.com.cn/html/index.asp)
- [The W3C Markup Validation Service](https://validator.w3.org/)

