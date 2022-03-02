---
title: "数据约束"
date: 2022-02-28T09:42:28+08:00
draft: true
---

lattice:


cue:
- validation 约束
- reducing boilerplate
- disallows overrides

CUE’s focus is data validation whereas Jsonnet focuses on data templating (boilerplate removal). Jsonnet was not designed with validation in mind.

scalability:
- samll: validate
- mediun: reduce boilerplate (using import and trim tools)
- large: advanced tooling, automation

比较：
- Jsonnet
- Json schema
- open api (缺点 very verbose)
- Rego
- 

定义
\#V3：# 打头，不会输出


Folding of Single-Field Structs

https://cuelang.org/docs/tutorials/tour/types/bottom/
top: _
Bottom: _|_
These all follow from the definition of unification:

The unification of a with itself is always a.
The unification of values a and b where a ⊑ b is always a.
The unification of a value with bottom is always bottom.
https://cuelang.org/docs/references/spec/#unification

unification: a & b


多行字符串
"""

开放字段
...

```cue
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

别名
let A = a

包的使用
https://cuetorials.com/zh/first-steps/modules-and-packages/

ls_tool.cue
https://github.com/cue-lang/cue/blob/v0.4.2/doc/tutorial/kubernetes/README.md#listing-object

语法：
扩展巴科斯范式
EBNF, Extended Backus–Naur Form

默认值：
* >=5 | int

bytes
s: bytes
s: '\x30\x30\x30'
a: len(s)

模式约束
[pattern]: value
A pattern constraint, denoted [pattern]: value, defines a pattern, which is a value of type string, and a value to unify with fields whose label match that pattern. When unifying structs a and b, a pattern constraint [p]: v declared in a defines that the value v should unify with any field in the resulting struct c whose label unifies with pattern p.

函数
close struct等同于 definition
#S: {
    field1: string
}

S1: #S & { field2: "foo" }

属性
@if()
@tag()


repeat:
s: "etc. "*3  // "etc. etc. etc. "

引用循环、结构循环
https://cuelang.org/docs/references/spec/#cycles

[1^]: (Cue tutorials: 模块和包)[https://cuetorials.com/zh/first-steps/modules-and-packages/]