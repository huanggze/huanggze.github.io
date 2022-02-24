---
title: "管理 Go 命令行工具"
date: 2022-02-24T19:21:30+08:00
categories: ["go"]
---

有时候，在我们的项目中需要使用一些 Go 命令行工具，比如 ginkgo 测试工具、CI 工具、代码自动生成工具 client-gen 等。因此，需要确保 Go 命令行工具在 CI 服务器上等不同环境版本一致。

解决办法是把依赖的工具加到 go module 中。创建 tools.go，在文件中引用依赖工具包：

```go
// +build tools

package main

import (
  _ "github.com/onsi/ginkgo/v2/ginkgo"
)
```

> 编译标签：\
> 注意 tools.go 中我们使用了编译标签（build tag）[^1]`// +build tools`，作用是 go build 时会过滤掉带有标签的 go 文件，除非使用 go build -tags 指定标签编译。因为 tools.go 带有标签，不会被编译，仅用于指定 go 二进制安装版本。

这样我们就可以使用 `go install github.com/onsi/ginkgo/v2/ginkgo` 安装到指定版本的 ginkgo 工具。go install 会把安装到目录下。

> go install 使用：\
> 只有在包含 go module 的项目中使用 go install 不需要指定版本。如果在项目外运行 go install 会报错：go install: version is required when current directory is not in a module。此时需要 `go install github.com/onsi/ginkgo/v2/ginkgo@<version>`

我们也可以在 Makefile 文件中，提供便捷安装命令[^2]：

```makefile
install-tools:
@echo Installing tools from tools.go
@cat go list -f '{{range .Imports}}{{.}} {{end}}' tools.go | xargs go install
```

[^1]: [Using Go's build tags](https://wawand.co/blog/posts/using-build-tags/)
[^2]: [Manage Go tools via Go modules](https://marcofranssen.nl/manage-go-tools-via-go-modules)