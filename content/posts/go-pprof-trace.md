---
title: "Go PProf Trace 分析"
date: 2022-10-22T23:16:42+08:00
draft: true
---

## Trace

![](/images/go_pprof_trace_1.png)

### 1. STATS 区域

- Goroutines：显示执行期间每个 Goroutine 运行阶段有多少个协程在运行。点击某一时间点，可以在下边信息栏看到详细说明。

![](/images/go_pprof_trace_2.png)

![](/images/go_pprof_trace_3.png)

面板里展示了三种状态的 Goroutine 的数量，分别是 GCWaiting（GC等待）， Runnable（可运行），Running（运行中）。有 7 个处于 GCWaiting，3 个处于 Running，543 个处于 Runnable。

实际上，[trace go 代码](https://github.com/golang/go/blob/go1.16.10/src/cmd/trace/trace.go#L465-L469)里有五种状态，另外两种是 Dead 和 Waiting：

```go
// src/cmd/trace/trace.go

package main

type gState int

// 五种 goroutine 状态常亮
const (
	gDead gState = iota
	gRunnable
	gRunning
	gWaiting
	gWaitingGC

	gStateCount
)

// 记录 goroutine 信息
type gInfo struct {
	state      gState // current state
	name       string // name chosen for this goroutine at first EvGoStart
	isSystemG  bool
	start      *trace.Event // most recent EvGoStart
	markAssist *trace.Event // if non-nil, the mark assist currently running.
}
```

但是在生成 trace 日志的时候，并没有输出 Dead 和 Waiting，而只保留了图片中的三种。

```go
// src/cmd/trace/trace.go

package main

func generateTrace(params *traceParams, consumer traceConsumer) error {
	defer consumer.flush()

	ctx := &traceContext{traceParams: params}
	ctx.frameTree.children = make(map[uint64]frameNode)
	ctx.consumer = consumer
	
	// ...

	// 更新 gstates 中记录的 goroutine 状态
	// Since we make many calls to setGState, we record a sticky
	// error in setGStateErr and check it after every event.
	var setGStateErr error
	setGState := func(ev *trace.Event, g uint64, oldState, newState gState) {
		info := getGInfo(g)
		
		// ...
		
		ctx.gstates[info.state]--
		ctx.gstates[newState]++
		info.state = newState
	}

	for _, ev := range ctx.parsed.Events {
		// Handle state transitions before we filter out events.
		switch ev.Type {
		case trace.EvGoStart, trace.EvGoStartLabel:
			setGState(ev, ev.G, gRunnable, gRunning)
			info := getGInfo(ev.G)
			info.start = ev
		// ...	
		case trace.EvGoBlockGC:
			setGState(ev, ev.G, gRunning, gWaitingGC)
		}
		
		
		// Emit any counter updates.
		ctx.emitThreadCounters(ev)
		ctx.emitHeapCounters(ev)
		ctx.emitGoroutineCounters(ev)
	}
	
	// ...
	
	return nil
}

type goroutineCountersArg struct {
	Running   uint64
	Runnable  uint64
	GCWaiting uint64
}

// ctx.emit 将 trace 记录放到 traceContext 中
func (ctx *traceContext) emitGoroutineCounters(ev *trace.Event) {
	if ctx.prevGstates == ctx.gstates {
		return
	}
	if tsWithinRange(ev.Ts, ctx.startTime, ctx.endTime) {
		// 代码里只打印了 Running、Runnable、GCWaiting 三种状态
		ctx.emit(&traceviewer.Event{Name: "Goroutines", Phase: "C", Time: ctx.time(ev), PID: 1, Arg: &goroutineCountersArg{uint64(ctx.gstates[gRunning]), uint64(ctx.gstates[gRunnable]), uint64(ctx.gstates[gWaitingGC])}})
	}
	ctx.prevGstates = ctx.gstates
}
```

那么这三种状态，分别代表什么意思呢？

首先，处于 Running 的 goroutine 表示正在运行在内核线程上（也就是 P 上）[^1]。

![](https://alanzhan.dev/2022-01-24-golang-goroutine/2022-01-24-gmp-detail.jpg)

Runnable 是放置在本地或全局 `runq` 队列上的 goroutine[^2]（相关代码可以看 runtime 包 [p](https://github.com/golang/go/blob/go1.16.10/src/runtime/runtime2.go#L598-L601) 和 [schedt](https://github.com/golang/go/blob/go1.16.10/src/runtime/runtime2.go#L741-L743) 数据结构），是接下来可以被调度运行 goroutine。

而 GCWaiting 是各种 Waiting 中的一种（goroutine 阻塞，即 gopark，原因可见源码列表 [waitReason](https://github.com/golang/go/blob/867babe1b1587ab6961c1d6274be2426e90bf5d4/src/runtime/runtime2.go#L1045-L1082)，其他常见的阻塞原因有 select 语句、channel 收发、mutex锁阻塞）,[trace 记录并统计了](https://github.com/golang/go/blob/go1.16.10/src/runtime/mgcmark.go#L618-L619)因为 waitReasonGCAssistWait 而阻塞的 goroutine，trace 记录事件为 [traceEvGoBlockGC](https://github.com/golang/go/blob/go1.16.10/src/runtime/trace.go)：

```go
// src/runtime/mgcmark.go

package runtime

func gcParkAssist() bool {
	// ...
	
	// Park.
	goparkunlock(&work.assistQueue.lock, waitReasonGCAssistWait, traceEvGoBlockGC, 2)
	return true
}
```

```go
// src/runtime/runtime2.go

package runtime

// A waitReason explains why a goroutine has been stopped.
// See gopark. Do not re-use waitReasons, add new ones.
type waitReason uint8

const (
	// ...
	waitReasonGCAssistWait    // "GC assist wait"
)
```

```go
// src/runtime/trace.go

package runtime

// Event types in the trace, args are given in square brackets.
const (
	// ...
	traceEvGoBlockGC = 42    // goroutine blocks on GC assist [timestamp, stack]
)
```

GCWaiting 顾名思义，他是指 goroutine 需要等待 GC 完成，堆内存上才有足够的空间来完成新创建的对象的分配。通过追踪 gcParkAssist 前后代码，可以知道 GCWaiting 的时机是这么个顺序：

1. [mallocgc()](https://github.com/golang/go/blob/go1.16.10/src/runtime/malloc.go#L948-L961)：当 goroutine 要在堆上分配内存的时候，比如创建切片，会进入到 mallocgc。mallocgc 会检查当前 goroutine 的 gcAssistBytes 并减去要创建的对象的内存大小。gcAssistBytes 记录的是 goroutine 通过辅助 GC 积累了多少字节的信用量[^3]。如果扣除后的差值为负数，就要调用 gcAssistAlloc() 执行辅助 GC；

2. [gcAssistAlloc()](https://github.com/golang/go/blob/go1.16.10/src/runtime/mgcmark.go#L387)：首先根据 gcAssistBytes 和最小辅助 GC 工作量（gcOverAssistWork） 来计算本次应达到的工作量 scanWork。同时也允许 goroutine 偷一部分后台 GC 工作协程 gcController 的信用额度 bgScanCredit 来减少 scanWork 量；

若 scanWork 为正，则开使辅助 GC，实际辅助 GC 的工作量 [workDone](https://github.com/golang/go/blob/go1.16.10/src/runtime/mgcmark.go#L542-L549)。如果工作量不够、且又不是因为被抢占（是的话，允许继续辅助GC），那就只能挂起了 gopark；

3. [gcParkAssist()](https://github.com/golang/go/blob/go1.16.10/src/runtime/mgcmark.go#L482)：goparkunlock 里就是我们在前面代码片段里展示的函数，这里会调用 goparkunlock 记录阻塞原因（waitReasonGCAssistWait）和 trace 事件（traceEvGoBlockGC）。goparkunlock 里还会继续调用 gopark 真正执行 goroutine 的挂起；

4. [gopark()](https://github.com/golang/go/blob/go1.16.10/src/runtime/proc.go#L319)：gopark 调用了 mcall，传入了函数指针 park_m。park_m() 中先记录 trace 事件然后完成挂起，完成 goroutine 的切换。

```go
// park continuation on g0.
func park_m(gp *g) {
	_g_ := getg()

	if trace.enabled {
		traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
	}

	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	if fn := _g_.m.waitunlockf; fn != nil {
		ok := fn(gp, _g_.m.waitlock)
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
		if !ok {
			if trace.enabled {
				traceGoUnpark(gp, 2)
			}
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
	schedule()
}
```

至此，我们知道了 GCWaiting 是什么含义。至于为什么单独留了 GCWaiting，而不显示其他所有 Waiting，根据当时的 [commit message](https://github.com/golang/go/commit/6da83c6fc006019f6fe0503099d165e19f465b1b)，作者的用意是，GCWaiting 并不像其他 Waiting 一样没有事情要做，如 select、channel、mutex 的等待，要等待网络 I/O 之类的，而是 GCWaiting 有任务要做，有点类似于 Runnable 状态，所以单独拿出来了，是有 GC 分析的价值。

参考资料：

[^1] [Golang Goroutine 與 GMP 原理全面分析](https://alanzhan.dev/2022-01-24-golang-goroutine/2022-01-24-gmp-detail.jpg)

[^2] [深度探索 Go 语言，P189-191]()

[^3] [深度探索 Go 语言，P319、P345]()

https://about.sourcegraph.com/blog/go/an-introduction-to-go-tool-trace-rhys-hiltner

gcwaiting
https://github.com/golang/go/commit/6da83c6fc006019f6fe0503099d165e19f465b1b
代码
https://github.com/golang/go/blob/go1.16.10/src/runtime/mgcmark.go#L619

G2444075

https://groups.google.com/g/golang-nuts/c/5dOKzJQHSRY

https://www.ardanlabs.com/blog/2015/02/scheduler-tracing-in-go.html

https://blog.gopheracademy.com/advent-2017/go-execution-tracer/

The "Network", "Timers", and "Syscalls" traces indicate events in
the runtime that cause goroutines to wake up.
https://github.com/golang/go/blob/master/src/cmd/trace/main.go


https://github.com/campoy/go-tooling-workshop/tree/master/3-dynamic-analysis


可视化工具
https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/

https://pkg.go.dev/runtime/trace#hdr-Tracing_runtime_activities


trace的用途
https://github.com/golang/go/issues/16293

latency-sensitive

https://making.pusher.com/golangs-real-time-gc-in-theory-and-practice/