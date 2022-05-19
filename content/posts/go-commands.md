---
title: "go 命令行工具"
date: 2022-05-17T23:35:55+08:00
toc: true
---

## go build

go build 用于编译代码，生成可执行文件。如执行 `go build ed.go` 会生成 ed 或 ed.exe[^1][^2]。

### 参数说明

#### -gcflags

编译器参数。一组传递给 `go tool compile` 的参数，可通过查 `go tool compile` 命令查到有哪些支持的参数[^3]，如 -N 关闭编译器优化，-l 关闭内联（inlining）：

```shell
go build -gcflags "-N -l" main.go
go build -gcflags="-N -l" main.go
go build -gcflags=-N -gcflags=-l main.go
```

#### -ldflags

链接器参数。一组传递给 `go tool link` 的参数[^4]，如 -X importpath.name=value 设置包内指定变量的值，该变量可以是未赋值或初始化了的：

```go
package main

var version = "1.0"

func main() {
	println(version)
}
```

```shell
$ go build -ldflags="-X main.version=2.0" main.go
$ ./main
2.0
```

#### -tags

一组逗号（,）隔开的标签，与标签匹配的源码才会被编译。

```go
//go:build prod
package main

func main() {
	println("test")
}
```

只有带上正确的 tag 才会编译以上代码。

```shell
go build -tags=prod .
```

#### -o

指定输出地址

## go install

go install 编译并安装可执行文件到 $GOPATH/bin 或 $HOME/go/bin 目录下，和 go build 有一样的参数[^5]。

```shell
go install [build flags] [packages]
```

## go generate

go generate 识别代码中的 `//go:generate command argument...` 指令来执行特定操作。

```go
//go:generate echo "this is a test."
package mypkg
```

执行 `go generate mypkg.go` 会打印 "this is a test."。

## go vet

[^1]: [go 工具链](https://www.jianshu.com/p/31eb601a6e95)
[^2]: [Go Command](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies)
[^3]: [go tool compile](https://pkg.go.dev/cmd/compile)
[^4]: [go tool link](https://pkg.go.dev/cmd/link)
[^5]: [go install](https://pkg.go.dev/cmd/go#hdr-Compile_and_install_packages_and_dependencies)