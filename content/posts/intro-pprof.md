---
title: "Intro Pprof"
date: 2022-02-28T16:26:32+08:00
draft: true
---

为什么需要 profile？
The codebase’s static analysis can be insufficient to detect why the program behaves badly. Benchmarks test an isolated function’s performance; they are insufficient to understand the whole picture
https://www.practical-go-lessons.com/chap-36-program-profiling

解析 profile
protoc --decode perftools.profiles.Profile  --proto_path /Users/huanggze/go/pkg/mod/github.com/google/pprof@v0.0.0-20210407192527-94a9f03dee38/proto  /Users/huanggze/go/pkg/mod/github.com/google/pprof@v0.0.0-20210407192527-94a9f03dee38/proto/profile.proto < block.pb

注意：
/Users/huanggze/go/pkg/mod/github.com/google/pprof@v0.0.0-20210407192527-94a9f03dee38/proto/profile.proto: File does not reside within any path specified using --proto_path (or -I).  You must specify a --proto_path which encompasses this file.  Note that the proto_path must be an exact prefix of the .proto file names -- protoc is too dumb to figure out when two paths (e.g. absolute and relative) are equivalent (it's harder than you think).

每一个 sample 代表一次采样?什么是 sample
A sample is a measurement. This measure is made at a certain time during the profiling process.
(sample 记录一个完整调用栈)
sample vs. location【function】

https://www.polarsignals.com/blog/posts/2021/08/03/diy-pprof-profiles-using-go/

trace vs. profile
Trace files can be generated with:

    - runtime/trace.Start
    - net/http/pprof package
    - go test -trace


24种 output
node()

flat time:function的自己耗时
cum time：包括等待调用返回的时间
sum: 占总时间的百分比

1。 Labels
仅用于 CPU
https://www.jetbrains.com/help/go/using-profiler-labels.html#viewing-labels-in-goland
https://www.polarsignals.com/blog/posts/2021/04/13/demystifying-pprof-labels-with-go/

2。 Profile
A Profile is a collection of stack traces showing the call sequences that led to instances of a particular event, such as allocation.
Each Profile has a unique name. A few profiles are predefined:

goroutine    - stack traces of all current goroutines
heap         - a sampling of memory allocations of live objects
allocs       - a sampling of all past memory allocations
threadcreate - stack traces that led to the creation of new OS threads
block        - stack traces that led to blocking on synchronization primitives
mutex        - stack traces of holders of contended mutexes

有哪些 profile 信息可以收集(profile type)：
- cpu
- delay
- inuse_space
- alloc_space
- inuse_objects
- alloc_space

3。CPU profile 是计算时间
30-second CPU profile:

4. Block profile 需要先 setBlockrate

5。 execution stack？execution trace？stack vs. trace?

A stack is a pile of objects. In real life, you can make a stack with woods, with glasses of champagne, or with everything that can be stacked.

In computer science, we are stacking function calls. When a program executes, it starts with a function. The main function is the first function executed. And then we will call other functions, that will call other functions...



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


https://slcjordan.github.io/posts/pprof/
https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/
https://medium.com/happyfresh-fleet-tracker/danny-profiling-1c60a19d30de