# 1. JSON

JSON 全称 JavaScript Object Notation，是一种表示结构化数据的数据格式，易于阅读编写，也易于解析和生成。

各种编程语言都有丰富的 JSON 解析库，如 C++ 的 RapidJSON 和 JsonCpp 等，Go 标准库的 encoding/json 包。

JSON 在线格式化网站：https://jsoneditoronline.org/

一个 JSON 例子：

```json
{
  "array": [
    1,
    2,
    3
  ],
  "boolean": true,
  "color": "gold",
  "null": null,
  "number": 123,
  "object": {
    "a": "b",
    "c": "d"
  },
  "string": "Hello World"
}
```

JSON 有对象、数组、值三种形式，这三种形式结构可以嵌套。

**对象**

JSON 的对象是一个无序的名称/值对的集合，以左括号`{`开始以右括号`}`结束，名称是字符串，名称和值中间是一个冒号`:`，多个名称/值对用逗号`,`分隔。

**数组**

数组是值的有序集合，以左中括号`[`开始以右中括号`]`结束，值之间使用逗号`,`分隔。

**值**

值（value）包括数组（string）、数值（number）、布尔值（boolean）、null、对象（object）、数组（array）。

# 2. XML

XML 全程 Extensible Markup Language，用于结构化存储数据，但是相比于 JSON 更加冗长。

XML 在线格式化网站：https://tool.oschina.net/codeformat/xml

一个 XML 例子：

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<note>
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note>
```

首行声明 XML 的版本和编码，首行之后的内容必须由一个根元素包含起来，然后在里面包含其它的子元素。

结构如下：

```xml
<root>
  <child>
    <subchild>.....</subchild>
  </child>
</root>
```

XML 的标签是指小于号`<`和大于号`>`包括起来的内容，分为开始标签和结束标签，结束标签是开始标签名称前加一个斜杠`/`。而元素是匹配的开始标签和结束标签包括起来的内容。

XML 标签名称大小写敏感。

每个元素可以拥有文本内容和属性，属性内容放在开始标签中，文本内容放在开始标签和结束标签中间。

```xml
<book category="COOKING">hello world</book>
```

XML 中文本内容出现了以下字符，解析器会出错，需要将相应字符转码为实体引用：

| 字符 | 实体引用 | 含义   |
| ---- | -------- | ------ |
| <    | `&lt;`   | 小于   |
| >    | `&gt;`   | 大于   |
| &    | `&amp;`  | 和号   |
| '    | `&apos;` | 单引号 |
| "    | `&quot;` | 引号   |

注释：

```xml
<!-- This is a comment --> 
```

# 3. YAML

YAML 全称是 YAML Ain't Markup Language，主要用于编写配置文件。

格式验证网站：https://nodeca.github.io/js-yaml/

YAML 支持三种数据结构：对象、数组、纯量（scalars），三者可以互相组合成复合结构，如下所示。

```yaml
languages:
 - Ruby
 - Perl
 - Python
websites:
 YAML: yaml.org
 Ruby: ruby-lang.org
 Python: python.org
 Perl: use.perl.org
```

YAML 大小写敏感，使用缩进表示层级关系，但只允许使用空格而不许使用 tab，缩进空格数组不重要，只要同层级元素对齐即可。

注释是 `#` 至行末的内容。

**对象**

对象是键值对的集合，又称为映射、哈希、字典。

键值对用冒号 `:` 分隔，可以将所有键值对写成一行。

```yaml
animal: pets

hash: { name: Steve, foo: bar }
```

**数组**

数组是一组按次序排列的值，又称为序列、列表。

每行开头用减号 `-` 表示这是一个数组。

```yaml
- Cat
- Dog
- Lion
```

**纯量**

纯量是单一的不可分的值，包括类型有字符串、布尔值、整数、浮点数、null、时间、日期。

布尔值包括 true 和 false，null 用 `~` 来表示。字符串默认不用引号，但如果包含空格或特殊字符，则需要单引号包含，字符串中的单引号用两个单引号 `''` 进行转义，可以多行表示但第二行开始需要缩进，也可以用 `|` 或 `>` 保留换行符，`|+` 表示文字块末尾换行，`|-` 表示文字块末尾不换行。

```yaml
this: |
  Foo
  Bar
that: >
  Foo
  Bar

s1: |+
  Foo
s2: |-
  Foo
```

通过两个感叹号 `!!` 可以强制转换数据类型，如下，整数被转化为字符串：

```yaml
e: !!str 123
```

通过锚点 `&` 和别名 `*` 可以用于引用，就像变量声明和引用一样，这里 `<<` 表示合并到当前数据。

```yaml
defaults: &defaults
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  <<: *defaults
```

# 4. TOML

TOML 全称是 Tom's Obvious Minimal Language，主要用于编写配置文件。

TOML 基本的构成结构是键值对，用等号 `=` 分隔，左右必须在同一行。值的类型包含字符串、整数、浮点数、布尔值、日期、时间、数组、内联表。

TOML 键名大小写敏感，不需要引号包含，不过如果键包含点则需要用单引号或双引号包含。

可以用点 `.` 在一行赋值多层中的值。

```toml
name = "Orange"
physical.color = "orange"
physical.shape = "round"
site."google.com" = true
```

等价于 json 结果如下：

```json
{
  "name": "Orange",
  "physical": {
    "color": "orange",
    "shape": "round"
  },
  "site": {
    "google.com": true
  }
}
```

注释是 `#` 至行末的内容。

布尔值包括 true 和 false。字符串用双引号 `"` 包含，可以用反斜杠 `\` 来进行转义，以在字符串中表示特殊字符。三个双引号 `"""` 包裹起来的是多行字符串。

**数组**

数组可以混合不同类型的值，可以跨行。

```toml
integers = [ 1, 2, 3 ]
colors = [ "红", "黄", "绿" ]

integers = [
  1,
  2,
  3
]
```

**表**

表也称为哈希表或字典，是键值对的集合。

表的开头是表头，用方括号包含，表头也可以包含点 `.` 表示当前是几层的路径。

```toml
[table]

[t1.t2.t3]
```

在表头的下方，直至下一个表头或文件结束，都属于这个表的键值对。

```toml
[table-1]
key1 = "some string"
key2 = 123

[table-2]
key1 = "another string"
key2 = 456
```

**内联表**

内联表使用花括号包含起来，中间可以包含键值对，用逗号分隔。

```toml
name = { first = "Tom", last = "Preston-Werner" }
```

**表数组**

表数组用双方括号包含，表示键名右侧不是一个对象而是一个数组。

```toml
[[products]]
name = "Hammer"
sku = 738594937

[[products]]  # 数组里的空表

[[products]]
name = "Nail"
sku = 284758393
color = "gray"
```

等价于 JSON 中的结构：

```json
{
  "products": [
    { "name": "Hammer", "sku": 738594937 },
    { },
    { "name": "Nail", "sku": 284758393, "color": "gray" }
  ]
}
```

# 5. 参考

* [JSON](https://www.json.org/json-zh.html)
* [The Official YAML Web Site](https://yaml.org/)
* [YAML 语言教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2016/07/yaml.html)
* [TOML：Tom 的（语义）明显、（配置）最小化的语言](https://toml.io/cn/)

