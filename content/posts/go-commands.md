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

go vet 检查代码并报告可疑错误。通过 go doc cmd/vet 可以查看支持的命令。

```go
package main

func main() {
	printf("test")
}
```

```shell
$ go vet .
# awesomeProject
vet: ./main.go:4:3: undeclared name: prinf
```

## go list

go list 逐行列出包名。

### 用法

```shell
go list [-f format] [-json] [-m] [list flags] [build flags] [packages]
```

示例：

```shell
$ tree .
.
├── gen
│   ├── a
│   │   └── a.go
│   ├── b
│   │   └── b.go
│   └── gen.go
└── main.go

$ go list ./...
awesomeProject
awesomeProject/gen
awesomeProject/gen/a
awesomeProject/gen/b
```

go 命令行工具中，（...）表示通配符，比如 `net/...` 同时匹配 `net` 包和 `net/http`[^6]。

另外，（.）以及（..）分别表示当前目录和上一级目录：

```shell
$ go list go list ./gen/a/../...
awesomeProject/gen

$ go list ./gen/a/../... 
awesomeProject/gen
awesomeProject/gen/a
awesomeProject/gen/b
```

多个 go 命令行工具可以组合使用：

```shell
$ go vet $(go list ./...)
# awesomeProject/gen/a
vet: gen/a/a.go:4:2: undeclared name: printf
```

### 参数说明

#### -m

打印模块名，而不是包名。all 是特殊用法，表示所有在使用模块。

```shell
$ go list -m all
awesomeProject
cloud.google.com/go v0.26.0
github.com/BurntSushi/toml v0.3.1
github.com/NYTimes/gziphandler v0.0.0-20170623195520-56545f4a5d46
github.com/PuerkitoBio/purell v1.1.1
...
```

#### -json

使用 json 格式打印

#### -u

提示可升级信息

```shell
$ go list -m -u all
k8s.io/api v0.21.0 [v0.24.0]
k8s.io/apimachinery v0.21.0 [v0.24.0]
k8s.io/gengo v0.0.0-20200413195148-3a45101e95ac [v0.0.0-20220307231824-4627b89bbf1b]
k8s.io/klog/v2 v2.8.0 [v2.60.1]
...
```

[^1]: [go 工具链](https://www.jianshu.com/p/31eb601a6e95)
[^2]: [Go Command](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies)
[^3]: [go tool compile](https://pkg.go.dev/cmd/compile)
[^4]: [go tool link](https://pkg.go.dev/cmd/link)
[^5]: [go install](https://pkg.go.dev/cmd/go#hdr-Compile_and_install_packages_and_dependencies)
[^6]: [Package lists and patterns](https://pkg.go.dev/cmd/go@go1.18.2#hdr-Package_lists_and_patterns)