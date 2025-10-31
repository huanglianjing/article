# 1. CSS简介

CSS全称层叠样式表（Cascading Style Sheets），用于描述HTML文档样式。可以写在HTML文档的style标签中，也可以保存在.css文件中被引用。

## 1.1 语法

CSS规则集由选择器和声明块组成，选择器指向需要设置样式的HTML元素，声明块包含若干条用分号分割的声明，每条声明由一个CSS属性和一个值组成。

![css_format](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/web/css_format.gif)

以上图片例子对应如下代码：

```css
p {
	color: red;
	font-size: 14px;
}
```

注释用`/*`和`*/`包裹，可以跨多行。

```css
/*comment*/
```

## 1.2 选择器

选择器用于查找要设置样式的HTML元素，分为以下五类：

- 简单选择器：根据标签、id、类选取
- 组合器选择器：根据它们之间的特定关系选取
- 伪类选择器：根据特定状态选取
- 伪元素选择器：选取元素的一部分设置样式
- 属性选择器：根据属性或属性值选取

以下为基本的简单选择器：

**元素选择器**

根据标签名称选择元素。

```css
p {}
```

**id选择器**

选择特定id的元素。

```css
#id {}
```

**类选择器**

选择特定类的元素。

```css
.class {}
```

也可以指定特定标签中的特定类。

```css
p.class {}
```

**通用选择器**

选择所有元素。

```css
* {}
```

**分组选择器**

选择多种条件组合应用同样的样式，减少重复代码。

```css
p, .c1, #id1 {}
```

**后代选择器**

选择指定元素后代中的所有标签。

```css
div p {}
```

**子选择器**

选择属于指定元素子元素的所有标签。

```css
div > p {}
```

:hover选择器

鼠标指针移到元素上时的样式。

```css
a:hover {
	background-color:yellow;
}
```



# 2. 颜色

颜色可以用多种方式来指定，以下为Tomato对应的不同颜色表示方法和值：

| 表示形式 | 值                      |
| -------- | ----------------------- |
| 颜色名称 | Tomato                  |
| RGB值    | rgb(255, 99, 71)        |
| HEX值    | \#ff6347                |
| HSL值    | hsl(9, 100%, 64%)       |
| RGBA值   | rgba(255, 99, 71, 1.0)  |
| HSLA值   | hsla(9, 100%, 64%, 1.0) |

具体支持的颜色名称和对应颜色值可以参考 [HTML 颜色名](https://www.w3school.com.cn/html/html_colornames.asp) 。

## 2.1 RGB颜色

rgb(red, green, blue)中的红、绿、蓝参数定义从0到255之间的颜色强度。

rgb(0, 0, 0)为黑色，rgb(255, 255, 255)为白色，rgb(255, 0, 0)为红色。

## 2.2 HEX颜色

#rrggbb中的红、绿、蓝参数分别是00到ff之间的十六进制值。

#000000为黑色，#ffffff为白色，#ff0000为红色。

## 2.3 HSL颜色

hsla(hue, saturation, lightness)的参数分别为色相、饱和度、明度。

色相是色轮上0到360的度数，饱和度是一个百分比，从0%阴影到100%全色，亮度是一个百分比，从0%黑色到100%白色。



# 3. 背景

背景属性用于定义元素的背景效果。

## 3.1 背景色

**background-color**

指定元素的背景色。

```css
div {
	background-color: lightblue;
}
```

**opacity**

指定元素的透明度。取值范围为0.0 - 1.0，值越低越透明。

```css
div {
	background-color: green;
	opacity: 0.3;
}
```

## 3.2 背景图像

**background-image**

指定用作元素背景的图像。

默认在水平和垂直方向上都重复。

```css
div {
	background-image: url("paper.gif");
}
```

**background-repeat**

设置水平和垂直方向是否重复，值`repeat-x`表示只在水平方向重复，值`repeat-y`表示只在垂直方向重复，值`no-repeat`表示不重复。

```css
div {
	background-image: url("tree.png");
	background-repeat: no-repeat;
}
```

**background-position**

指定背景图像的位置。

```css
div {
	background-image: url("tree.png");
	background-repeat: no-repeat;
	background-position: right top;
}
```

**background-attachment**

指定背景图像是应该滚动还是固定的，值`fixed`表示固定，值`scroll`表示滚动。

```css
div {
	background-image: url("tree.png");
	background-repeat: no-repeat;
	background-position: right top;
	background-attachment: fixed;
}
```

**background-size**

指定背景图像的尺寸。

```css
div {
	background: url(img_flwr.gif);
	background-size: 80px 60px;
}
```

**background**

background属性可以指定多个边框属性，`background-color`、`background-image`、`background-repeat`、`background-attachment`、`background-position`。

```css
div {
	background: #ffffff url("tree.png") no-repeat right top;
}
```



# 4. 边框

**border-style**

指定要显示的边框类型。

```css
div {
	border-style: dotted;
}
```

值可以为：

- dotted 点线边框
- dashed 虚线边框
- solid 实线边框
- double 双边框
- groove 3D坡口边框
- ridge 3D脊线边框
- inset 3D inset边框
- outset 3D outset边框
- none 无边框
- hidden 隐藏边框

可以设置一到四个值，用于上边框、右边框、下边框和左边框。

如果设置1个值，则同时表示四个边框。如果设置2个值，则分别表示上下和左右边框，如果设置三个值，则分别表示上、左右和下边框。

```css
div {
	border-style: dotted dashed solid double;
}
```

**border-top-style**
**border-right-style**
**border-bottom-style**
**border-left-style**

分别设置四个边框。

```css
div {
	border-top-style: dotted;
	border-right-style: solid;
	border-bottom-style: dotted;
	border-left-style: solid;
}
```

**border-width**

设置边框宽度。

值的单位px表示像素，%表示百分比例，也可以是thin、medium或thick。

```css
div {
	border-style: solid;
	border-width: 5px;
}
```

可以设置一到四个值，用于上边框、右边框、下边框和左边框。

**border-color**

设置边框颜色。

```css
div {
	border-style: solid;
	border-color: red;
}
```

**border-radius**

定义圆角边框。

```css
div {
	border: solid;
	border-radius: 5px;
}
```

**border**

border属性可以指定多个边框属性，包含`border-width`、`border-style`、`border-color`。

```css
div {
	border: 5px solid red;
}
```



# 5. 边距和宽高

CSS框模型的各种属性如下图：

黑色实线处为边框，分为外边距和内边距，中间是实际元素的高度和宽度。

![css_border_model](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/web/css_border_model.gif)

**margin-top**
**margin-right**
**margin-bottom**
**margin-left**

设置每一侧的外边距。

单位可以是px或%，值为auto表示自动设置居中。

```css
div {
	margin-top: 100px;
	margin-bottom: 100px;
	margin-right: 150px;
	margin-left: 80px;
}
```

**margin**

可以设置一至四个值，表示的方向和边框类型一致。

```css
div {
	margin: 25px 50px 75px 100px;
}
```

**padding-top**
**padding-right**
**padding-bottom**
**padding-left**

设置每一侧的内边距。

单位可以是px或%。

```css
div {
	padding-top: 50px;
	padding-right: 30px;
	padding-bottom: 50px;
	padding-left: 80px;
}
```

**padding**

可以设置一至四个值，表示的方向和边框类型一致。

```css
div {
	padding: 25px 50px 75px 100px;
}
```

**height**

**width**

设置元素的高度和宽度。

单位可以为px和%。

```css
div {
	height: 200px;
	width: 50%;
}
```

设置高度宽度的最大最小值：

* max-height 最大高度
* max-width 最大宽度
* min-height 最小高度
* min-width 最小宽度



# 6. 轮廓

轮廓是在元素的边框外绘制的，以凸显元素，是可能与其他内容重叠的。

**outline-style**

轮廓样式，和border-style可以设置的值一样。

```css
div {
	border-style: dotted;
	outline-style: dotted;
}
```

**outline-width**

指定轮廓的宽度。

**outline-color**

指定轮廓的颜色。

**outline**

outline属性可以指定多个轮廓属性，包含`outline-width`、`outline-style`、`outline-color`。

```css
div {
	outline: 5px solid yellow;
}
```



# 7. 文本

**color**

设置文本的颜色。

```css
div {
	color: blue;
}
```

**text-align**

设置文本水平对齐方式，可以是以下值：

- center 居中对齐
- left 左对齐
- right 右对齐
- justify 拉伸每一行使得具有相等宽度，且左右边距是直的

```css
div {
	text-align: center;
}
```

**vertical-align**

设置元素垂直对齐方式，可以是以下值：

- top
- middle
- bottom

**text-decoration**

设置或删除文本装饰。

值为none表示无装饰，常用于删除链接的下划线，underline为下划线，overline为上划线，line-through为删除线。

**text-transform**

指定文本中的大写和小写字母。

**text-indent**

指定文本第一行的缩进。

**letter-spacing**

指定文本中字符之间的间距。

**line-height**

指定行之间的间距。

**word-spacing**

指定文本中单词之间的间距。

**text-shadow**

为文本添加阴影。



# 8. 字体

有五个通用字体族：

- serif 衬线字体，字母每个边缘有一个小的笔触
- sans-serif 无衬线字体，线条简洁
- monospace 等宽字体
- cursive 草书字体，模仿人类笔记
- fantasy 幻想字体，装饰性

**font-family**

设置文本字体。

可以设置多个字体作为后备，以逗号分隔。如果字体名称不止一个单词，则必须用引号引起来。

```css
div {
	font-family: "Times New Roman", Times, serif;
}
```

**font-style**

设置字体样式，可以是以下值：

- normal 正常字体
- italic 斜体
- oblique 倾斜

**font-weight**

指定字体的粗细。

**font-size**

设置文本的大小。

**font**

font属性可以指定多个字体属性，包含`font-style`、`font-variant`、`font-weight`、`font-size`、`font-family`。

```css
div {
	font: italic small-caps bold 12px/30px Georgia, serif;
}
```



# 参考

- [W3school - CSS 教程](https://www.w3school.com.cn/css/index.asp)

