# 1. 介绍

文档地址：https://pkg.go.dev/encoding/json

Go 标准库 encoding/json 用于进行json的编码和解码。

结构体成员变量的标签中的 json 标签表示转为 JSON 后对应 key 的名称，如果没有则会用变量名称作为 key 名称。

结构体编码为字符串示例：

```go
type People struct {
	Name string `json:"name"`
	Age  int64  `json:"age"`
}

p := People{Name: "Moondo", Age: 10}
marshal, err := json.Marshal(p)
fmt.Printf("%s\n", marshal)
```

字符串解码为结构体示例：

```go
type People struct {
	Name string `json:"name"`
	Age  int64  `json:"age"`
}

str := "{\"name\":\"Moondo\",\"age\":10}"
var p People
err := json.Unmarshal([]byte(str), &p)
fmt.Printf("%s\n", p.Name)
```

# 2. 类型

## 2.1 RawMessage

类型定义：

```go
// RawMessage json字符串
type RawMessage []byte
```

方法：

```go
// MarshalJSON 编码并返回
func (m RawMessage) MarshalJSON() ([]byte, error)

// UnmarshalJSON 从data赋值自身
func (m *RawMessage) UnmarshalJSON(data []byte) error
```

## 2.2 Decoder

类型定义：

```go
// Decoder 从输入流读取和解码json
type Decoder struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// NewDecoder 创建对象
func NewDecoder(r io.Reader) *Decoder

// Buffered 获取对应的Reader
func (dec *Decoder) Buffered() io.Reader

// Decode 从输入流读取json并解码至v
func (dec *Decoder) Decode(v any) error

// 获取输入流offset
func (dec *Decoder) InputOffset() int64

// More 是否还有待处理
func (dec *Decoder) More() bool
```

## 2.3 Encoder

类型定义：

```go
// Encoder 编码写到输出流
type Encoder struct {
	// contains filtered or unexported fields
}
```

方法：

```go
// NewEncoder 创建对象
func NewEncoder(w io.Writer) *Encoder

// Encode 将v编码写到输出流
func (enc *Encoder) Encode(v any) error

// SetEscapeHTML 设置HTML转码
func (enc *Encoder) SetEscapeHTML(on bool)

// SetIndent 设置缩进
func (enc *Encoder) SetIndent(prefix, indent string)
```

# 3. 函数

```go
// Marshal 将变量编码为json字符串
// 结构体成员通过json标签设置转为json的key，如`json:"Name"`
// `json:"Name,omitempty"`表示当成员为空值时不输出json key
// `json:"Name,string"` 表示输出json的value为字符串
// `json:"-"`表示该成员不输出json key
// `json:"-,"`表示该成员json key为"-"
func Marshal(v any) ([]byte, error)

// MarshalIndent 将变量编码为json字符串，带缩进
func MarshalIndent(v any, prefix, indent string) ([]byte, error)

// Unmarshal 将json字符串解码至变量
// json的value类型对应接收的go类型：
// boolean -> bool
// number  -> float64
// string  -> string
// array   -> []interface{}
// object  -> map[string]interface{}
// null    -> nil
func Unmarshal(data []byte, v any) error

// Valid 是否合法json
func Valid(data []byte) bool

// Compact 将src的json字符串去掉空字符串，写入dst
func Compact(dst *bytes.Buffer, src []byte) error

// Indent 将src的json字符串添加缩进，写入dst
func Indent(dst *bytes.Buffer, src []byte, prefix, indent string) error

// HTMLEscape 将src的json字符串进行HTML转码，如<转为\u003c，写入dst
func HTMLEscape(dst *bytes.Buffer, src []byte)
```

