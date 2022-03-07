---
title: "Intro Pprof"
date: 2022-02-28T16:26:32+08:00
draft: true
---

```go
package main

import (
	"net/http"
	_ "net/http/pprof"
	"runtime"
	"time"
)

// https://jishuin.proginn.com/p/763bfbd6b80d
// https://github.com/DataDog/go-profiler-notes/blob/main/block.md
func main() {
	//runtime.SetBlockProfileRate(1)
	//runtime.SetBlockProfileRate(1000 * 1000 * 1000 * 1)
	// rate=2秒，一半的概率能捕捉到阻塞事件
	runtime.SetBlockProfileRate(1000 * 1000 * 1000 * 2)
	//runtime.SetBlockProfileRate(1000 * 1000 * 1000 * 10)

	// 阻塞 goroutine，阻塞 1 秒
	go func() {
		select {
		case <-time.After(1 * time.Second):
			println("done")
		}
	}()

	// 打印 pprof
	http.ListenAndServe(":9909", nil)
}

```
