---
title: "Intro Cue"
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
Bottom: _|_

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