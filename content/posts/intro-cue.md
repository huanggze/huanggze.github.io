---
title: "开源数据约束语言 CUE"
date: 2022-02-28T09:42:28+08:00
mermaid: true
toc: true
categories: ["配置"]
---

CUE（Configure Unify Execute）是一种数据验证语言，有自己的推理引擎，可作为配置语言使用。CUE 语言的特点是把数据类型（type）和值（value）看作同一概念：Types are Values。如下图展示了 CUE 的核心思想，左边是数据，中间是类型，右边（.cue）是数据与类型的混合，约束了任何 largeCapital 数据必须包含字符串类型的 name 字段，大于 5M 的 pop 字段以及值为 true 的 capital 字段。

![intro-cue-1](/images/intro-cue-1.png)

CUE 的主要功能包含：
- 数据验证：客户侧数据校验
- 代码导入/导出：支持从 Go 代码、Protobuf、YAML、JSON 等源文件转换成 CUE 文件
- 配置管理：是 JSON 的超集，支持类型检查
- 辅助工具：`cue trim` 自动修剪冗余

## CUE 示例

下面 cue 文件定义了三个字段，a 是 int 类型，值为 1；b 是对象类型；c 是 string 类型，未赋具体指。

```text
// demo.cue
a: int
a: 1
b: {
	c: "abc"
}
d: string
```

执行验证命令 `cue eval demo.cue`，可以发现 `a: int` 和 `a: 1` 合并（Unification），c 因为没有具体值，保留 `d: string`：

```text
a: 1
b: {
    c: "abc"
}
d: string
```

由于 d 没有具体指，不完整，因此 cue 不能导出 JSON 格式。修改 `d: "hi"`，执行 `cue export demo.cue`，JSON 格式正确导出了：

```json
{
    "a": 1,
    "b": {
        "c": "abc"
    },
    "d": "hi"
}
```

## CUE 原理：格

CUE 支持多个文件合并，正如前面示例中对同一字段 a 的合并。合并（Unification）是 CUE 的一大亮点和设计思想，CUE 中的 U 即代表合并的意思。CUE 合并操作的理论支持来自数学中的格（Lattice）。由于合并操作是顺序无关的，所以 CUE 功能非常强大。下图是合并的逻辑示意图以及对应 cue 文件：

{{< mermaid >}}
graph TD
A("_ (top)") --> N(number)
N --> I("int")
N --> GEH(">=0.5")
N --> LTC("<10")
I --> Z("0")
I --> One("1")
IFI("1.1")
GEH --> One
GEH --> IFI
GEH --> CCF("20.0")
LTC --> One
LTC --> IFI
Z --> E
One --> E
IFI --> E
CCF --> E
E("_|_ (bottom)") 
{{< /mermaid >}}

```text
a: _
a: number
a: float
a: >= 0.5 & < 10
a: 1.1
a: 20.0 // error
```

以上 cue 文件合并操作会报错，这是因为在 `a: 1.1` 和 `a: 20.0` 这里合并冲突，两个都是原子值（atom）。CUE 中，`_` 表示任意值，是合并逻辑中的上确界（top），`_|_` 表示错误，是下确界（bottom）。CUE 使用格理论能很好地处理无序合并逻辑。合并操作也可以使用 & 操作符（a & b）显示地表示，底层逻辑是[^1]：

1. a 与自身合并总等于自身 a；
2. a 与 b 合并，如果 a ⊑ b，则合并结果总为 a；
3. a 与 bottom（\_|\_）合并等于 bottom。

## CUE 语法

#### 1. 定义模板

\# 打头的字段是定义模板（definition），用于定义数据格式。\# 字段不是被打印，仅被引用。如果引用合并了定义不包含的字段，则合并操作会报错。

```text
#MyStruct: {
    sub: field:    string
    sub: enabled?: bool
}

myValue: #MyStruct & {
    sub: feild:   2     // error, feild not defined in #MyStruct
    sub: enabled: true  // okay
}
```

### 2. 多行字符串

使用 """ 来引用多行字符串。

```text
"""
    hello:
    world
"""
```

### 3. 开放字段

使用 ... 表示允许追加额外字段或数组元素。? 表示可选字段，但如果是开放数组 arr: [...]，不必加?。

```text
#MyStruct: {
    sub: {
    	field:    string
    	enabled?: bool
    	...
    }
}

myValue: #MyStruct & {
	  sub: {
	  	field: "hi"
	  	feild: 2       // no error this time
	  	enabled: true
	  }
}
```

### 4. 别名

别名的三种写法，X 都可以作为别名使用：
1. `let X = expr`，expr 值作为 X 的值；
2. 在标签中，`X=label: expr`，expr 值作为 X 的值
3. 在模式限制中，`[X=expr]: value`，把模式匹配（expr）到的值作为 X 的值；

### 5. 模式限制

模式限制（pattern constraint）使用 `[pattern]: value` 格式来字段名匹配 `pattern` 的字段的值需要被 `value` 约束。

```bash
// Name 这里用了别名
job: [Name=_]: {
    name:     Name
    replicas: uint | *1
    command:  string
}

job: list: command: "ls"

job: nginx: {
    command:  "nginx"
    replicas: 2
}
```

`cue export` 输出结果是：

```json
{
    "list": {
        "name": "list",
        "replicas": 1,
        "command": "ls"
    },
    "nginx": {
        "name": "nginx",
        "command": "nginx",
        "replicas": 2
    }
}
```

再看一个例子：

```text
a: {
    foo:    string    // foo is a string
    [=~"^i"]: int     // all other fields starting with i are integers
    [=~"^b"]: bool    // all other fields starting with b are booleans
    ...string         // all other fields must be a string
}

b: a & {
    i3:    3
    bar:   true
    other: "a string"
}
```

### 6. 循环字段

引用循环可以被处理，结构循环会报错。

```text
// 引用循环
a: b & { x: 1 }   // a: { x: 1, y: 2, z: 3 }
b: c & { y: 2 }   // b: { x: 1, y: 2, z: 3 }
c: a & { z: 3 }   // c: { x: 1, y: 2, z: 3 }

// 结构循环
a: b: a  // error
```

### 7. 隐藏字段

\_ 打头的字段是隐藏字段，不会被打印。

### 8. 循环控制

```text
a: [1, 2, 3, 4]
b: [ for x in a if x > 1 { x+1 } ]  // [3, 4, 5]
```

### 9. 条件控制

```text
price: number

// Require a justification if price is too high
if price > 100 {
    justification: string
}

price: 200
```

### 10. 包

同一 `package <name>` 打头的 cue 文件在同一包下。

### 11. 默认值

\* 标记的值是默认值

### 12. 属性

属性写法如示 `@go(Field)`，用于表示字段属性[^2]。

### 13. 模块

CUE 也支持 module 和 import。比如导入内置包后[^3]，可以使用内置包的函数：

```text
import (
	"encoding/json"
	"math"
)

// data: { a: 2.6457513110645907 }
data: json.Marshal({ a: math.Sqrt(7) })
```

甚至可以自己创建包，并引用。如 KubeVela 项目自己创建了 kube api 包：

```text
import (
   apps "kube/apps/v1"
)

parameter: {
    name:  string
}

output: apps.#Deployment
output: {
    metadata: name: parameter.name
}
```

[^1]: [CUE Unification](https://cuelang.org/docs/references/spec/#unification)
[^2]: [CUE Attributes](https://cuetorials.com/zh/deep-dives/attributes/)
[^3]: [Cuetorials](https://cuetorials.com/overview/standard-library/)