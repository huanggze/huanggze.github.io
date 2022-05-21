---
title: "Go 环境变量"
date: 2022-05-21T11:37:29+08:00
---

go 的 runtime 包提供一些控制 Go 程序运行时的操作，其中包括 go 环境变量[^1]。

## GOTRACEBACK

GOTRACEBACK 环境变量用于控制当 Go 程序 panic 时输出栈信息的多少。变量值可以为：

- none：不输出任何信息
- single：仅输出当前崩溃的 goroutine 信息（默认值）
- all：打印所有由用户创建的 goroutine 信息
- system：类似 all，并包含系统创建的 goroutine
- crash：类似 system，但不是直接 panic 退出，而是执行操作系统的一些操作。比如，在 Unix 系统上会生成 core dump

修改 GOTRACEBACK 有两种方式，通过 runtime/debug 包在代码里修改：

```go
package main

import (
	"runtime/debug"
	"time"
)

func main() {
	debug.SetTraceback("crash")
	
	go func() {
		for {
			println("go routine")
			time.Sleep(time.Second)
			var i *int
			*i=0
		}
	}()

	for {
		println("main func")
		time.Sleep(time.Second)
	}
}
```

或者在程序启动时修改：

```shell
GOTRACEBACK=crash go run main.go
```

注意，通过 os.Setenv 来设置是无效的，因为程序以及启动了，设置太晚了[^2]。

```go
func main() {
    os.Setenv("GOTRACEBACK", "crash")
	
    ...
}
```

[^1]: [go runtime: Environment Variables](https://pkg.go.dev/runtime#hdr-Environment_Variables)
[^2]: [Why Calling Setenv from code Not Working?](https://stackoverflow.com/questions/57176229/why-calling-setenv-from-code-not-working)