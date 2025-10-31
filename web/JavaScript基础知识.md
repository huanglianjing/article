# 1. JavaScript语法

## 1.1 在HTML中使用JavaScript

在HTML文档的head元素或body元素中，使用script标签，在其中写入JavaScript代码。

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>page name</title>
	<script>
		document.write("Hello World!");
	</script>
</head>
<body>
</body>
</html>
```

也可以把JavaScript代码保存在一个.js文件中，从script标签引用该js文件。

```html
<script src="example.js"></script>
```

script标签放在body中，会在页面加载的时候就执行，加载速度更快。而放在head中会在被调用的时候才执行，W3C建议放在head中，便于管理。

## 1.2 语法

### 1.2.1 语句

JavaScript语句末尾可以带分号也可以不带分号，但是建议都带。多条语句放在同一行必须要有分号分隔。

```javascript
statement
statement

statement;
statement;

statement; statement;
```

注释可以是如下三种方式：

```javascript
// comment

/*
comment
*/

<!-- comment
```

### 1.2.2 变量

变量声明和复制：

```javascript
// 声明
var name;

// 赋值
name = "john";

// 未声明变量在使用时自动声明
id = "356734573";
```

JavaScript中不需要声明类型，是一种弱类型语言，可以在任何时候改变变量的数据类型。

JavaScript有以下几种数据类型：字符串、数值、布尔值、数组、对象。

**字符串**

字符串由单引号或双引号包含起来。

```javascript
var a = 'fdsfsd';
var b = "dsfsg";

// 字符串拼接
a = "asd" + "33r";
a += "hehe";
```

**数值**

数值可以是整数或浮点数。

```javascript
var a = 23523;
var b = -325.666;
```

**布尔值**

布尔值有两种值：true或false。

```javascript
var a = true;
var b = false;
```

**数组**

数组（array）是变量的集合，集合中每个值是数组的一个元素（element），元素个数为数组的长度（length）。

数组中的元素数据类型可以是不同的，元素也可以是一个数组。

```javascript
// 声明数组
var arr = Array(4);

// 声明数组，不指定长度
var arr = Array();

// 声明数组，同时填充元素
var arr = Array("dsf", 333, "jj jfie");

// 填充数组
arr[0] = "34rtf435t";
arr[1] = 3432;

// 获取元素
arr[1];

// 数组长度
arr.length;
```

用字符串代替数组作为下标是关联数组，实际上创建的是数组对象的属性，不推荐使用，这种情况应该使用对象。

```javascript
var arr = Array();
arr["name"] = "mike";
arr["year"] = 2030;
```

**对象**

对象（objects）是属性的集合。

```javascript
// 声明对象
var obj = Object();

// 声明对象同时设置属性
var obj = Object(name:"mike", year:2030);

// 设置对象属性
obj.name = "mike";
obj.year = 2030;

// 获取属性
obj.name;

// 调用方法
obj.method();
```

### 1.2.3 操作

**赋值操作符**

赋值使用等号（=）。

**算术操作符**

算术操作使用加号（+）、减号（-）、乘号（*）、除号（/）、自增（++）、自减（--）、加等于（+=）。

加号和加等于也可以用于字符串拼接。

**比较操作符**

大于（>）、小于（<）、大于等于（>=）、小于等于（<=）、等于（==）、不等于（!=）。

**逻辑操作符**

逻辑与（&&）、逻辑或（||）、逻辑非（!）。

### 1.2.3 条件语句

**条件语句**

若花括号中只包含一条语句的话，可以忽略花括号。

```javascript
if (condition) {
	statement;
}

if (condition)
	statement;

if (condition) {
	statement;
} else {
	statement;
}
```

**循环语句**

```javascript
while (condition) {
	statement;
}

do {
	statement;
} while (condition);

for (condition; condition; condition) {
	statement;
}
```

### 1.2.4 函数

参数（argument）用逗号分隔，也可以不传。

```javascript
function name(argument) {
	statement;
}
```



# 2. DOM

DOM分别为文档D（document）、对象O（object）、模型M（model）。

在DOM里，文档是由节点树上的节点（node）构成的集合。

元素节点（element node）是DOM中的各种元素，标签名字就是元素的名字，元素可以包含其他的元素，html元素是节点树的根元素。

文本节点（text node）包含在元素节点的内部，是元素节点中的文本内容，但不是所有元素节点都包含文本节点。

属性节点（attribute node）是元素节点的属性。

## 2.1 文档

根据元素id获取元素：

```javascript
document.getElementById(id);
```

根据标签名字获取元素数组：

```javascript
document.getElementsByTagName(tag);
```

根据类名获取元素数组：

```javascript
document.getElementsByClassName(class);
```

向文档增加内容：

```javascript
document.write(text);
```

创建元素节点：

```javascript
document.createElement("p"); // 创建了一个p元素
```

创建文本节点，再将该节点插入元素节点作为字节点。

```javascript
document.createTextNode(text);
```

## 2.2 节点属性

根据属性名获取属性值：

```javascript
node.getAttribute(attribute);
```

设置属性：

```javascript
node.setAttribute(attribute, value);
```

元素的属性：

```javascript
node.childNodes  // 获取元素的所有子元素的数组
node.firstChild  // 子元素数组的第一个
node.lastChild   // 子元素数组的最后一个
node.nextSibling // 元素的下一个兄弟元素
node.nodeType    // 节点类型，1：元素节点，2：属性节点，3：文本节点
node.nodeValue   // 节点的值
```

节点插入新的子节点：

```javascript
node.appendChild(child);
```

在子节点的前面插入新节点：

```javascript
parentNode.insertBefore(newNode, childNode);
```

在子节点的后面插入新节点：

```javascript
parentNode.insertBefore(newNode, childNode);
```

## 2.3 节点事件

节点被点击触发onclick事件，将其与函数绑定，在点击时就会调用：

```javascript
node.onclick = functionName;
```

节点被键盘按下触发onkeypress事件，将其与函数绑定，在键盘按下时就会调用：

```javascript
node.onkeypress = functionName;
```

## 2.4 窗口

弹出确认框：

```javascript
alert(text);
```

创建新的浏览器窗口：

```javascript
window.open(url, name, features);
```

页面加载完毕后触发onload事件，将一个JavaScript函数与window对象关联，在页面加载完毕后就会调用该函数：

```javascript
window.onload = functionName;
```

若要执行多个函数，可以将这些函数包括在一个公共的函数内，进行关联。

## 2.5 调用JavaScript函数

在js文件中声明函数，可以在HTML文档中引用并调用它。

首先在showPic.js文件中声明一个函数，如以下例子表示，点击页面的链接时将图片展示到页面的某个位置而不是打开新窗口。

```javascript
function showPic(pic) {
	var source = pic.getAttribute("href");
	var placeholder = document.getElementById("placeholder");
	placeholder.setAttribute("src", source);
}
```

在HTML文档中插入该JavaScript文件：

```html
<script src="showPic.js"></script>
```

然后可以在链接的onclick属性填入该函数，在点击的时候就会调用。

```html
<a href="http://example.com" onclick="showPic(this); return false;">click</a>
```



# 3. CSS

网页信息分为三层：结构层由HTML创建，包含网页的结构以及内容，表示层由CSS完成，描述页面内容如何呈现，行为层由JavaScript和DOM负责，实现如何响应不同事件。

## 3.1 DOM读写CSS

通过DOM可以改变网页的结构，也可以给元素设定样式。文档的每个元素节点都有一个属性style，包含元素的各种样式。node.style是一个对象，节点的样式存放在该对象的属性中，可以读取或设置。

```javascript
var para = document.getElementById(id);

para.style.color // color属性
para.style.fontFamily // font-family属性，DOM需要用驼峰命名
```

## 3.2 实现动画效果

**位置信息**

CSS负责设置网页元素的位置信息：

```css
element {
	position: absolute;
	top: 500px;
	left: 100px;
}
```

可以通过DOM修改CSS中的位置信息。

**时间**

经过一定时间之后开始执行函数，第一个参数为执行的函数，第二个参数为时间，单位为毫秒。

```javascript
variable = setTimeout("func()", interval);
```

取消等待执行的函数：

```javascript
clearTimeout(variable);
```

通过每过一段时间改变元素的位置信息，实现连贯的动画效果。



# 参考

- [《JavaScript DOM编程艺术 （第2版）》](https://book.douban.com/subject/6038371/)

