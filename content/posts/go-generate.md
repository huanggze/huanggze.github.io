---
title: "go generate 命令"
date: 2022-05-18T22:12:02+08:00
series: ["go command"]
---

go generate 识别代码中的 `//go:generate command argument...` 指令来执行特定操作。

```go
//go:generate echo "this is a test."
package mypkg
```

执行 `go generate mypkg.go` 会打印 "this is a test."。


[^2]: [Go Command](https://pkg.go.dev/cmd/go#hdr-Generate_Go_files_by_processing_source)
