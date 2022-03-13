---
title: "Go pprof 性能分析"
date: 2022-02-28T16:26:32+08:00
toc: true
categories: ["go"]
---

## PProf

Golang 提供了程序性能分析工具 pprof。性能分析（profiling）可以收集运行期间的程序性能情况，弥补了其他静态测试方法的不足。pprof 支持三种方式采样和生成性能报告（profile）：

1. 通过 runtime/pprof 包将采集结果保存到文件；
2. 通过 net/http/pprof 打开 /debug/pprof HTTP 接口暴露数据；
3. `go test -cpuprofile=cpu.profile` 命令采集测试用例运行的性能数据。

```go
// 分析报告导出为文件
func main() {
	f, _ := os.Create("cpu.profile")
	defer f.Close()
	
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()
	
	for i := 0; i < 3; i++ {
		go sum()
	}
}
```

```go
func sum() {
	d := 0
	for i := 0; i < 10_000_000_000; i++ {
		d += i
	}
}

func main() {
	for i := 0; i < 3; i++ {
		go sum()
	}

	// 使用 go tool pprof -web 127.0.0.1:8080 http://127.0.0.1:9090/debug/pprof/profile\?seconds\=2
	// 可查看 2 秒内，CPU 性能分析报告
	http.ListenAndServe(":9090", nil)
}
```

## Profile 类型

Profile 分析统计记录的是一组调用栈以及相关的事件，比如 CPU 使用时间、内存分配、阻塞时间等。Profile 除了 CPU 比较特殊以为，还有以下几种：
- goroutine：当前所有 goroutine 以及调用栈；
- heap：堆内存以及存活对象内存分配统计
- block：记录阻塞事件
- mutex：记录锁竞争

### CPU

CPU 是最基础的性能分析，开启采样需要在代码中调用 pprof.StartCPUProfile 和 pprof.StopCPUProfile，或调用 /debug/pprof?seconds=\<duration\> HTTP 接口。pprof 默认采样间隔是 100ms。另外，仅 CPU profile 支持 label 给不同 goroutine 打标签帮助细分采样来源[^1]。

### Heap

pprof 主要关注堆内存的情况。pprof 采样会记录上一次 GC 到采样的堆内存分配状况。pprof 默认每分配 512kb 的内存触发一次采样[^2]。heap profile 又分为四种：inuse_space（默认，当前用量 bytes），inuse_objects（当前存活对象数），alloc_space 和 alloc_objects，记录当前内存分配以及过去所有内存分配汇总情况，这样我们不仅可以知道一个程序运行时当前分配的内存情况，以及哪段代码过往内存分配很大。如果要保存 alloc_\* 信息，则将代码中的 pprof.WriteHeapProfile 替换为 pprof.Lookup("allocs").WriteTo(file, 0)。

```go
func main() {
	f, _ := os.Create("mem.profile")
	defer f.Close()

	runtime.GC()
	defer pprof.WriteHeapProfile(f)

	s := make([]byte, 0)
	for i:= 0; i < 1024 * 1024 * 10; i++{
		s = append(s, 1)
	}

	// s = nil
	// runtime.GC()
	// 如果再调用一次 GC，则采样结果会为空
}
```

以下是内存分配采样底层实现简化代码，可以看到 go 是如何采样埋点的[^3]：

```go
func malloc(size):
  object = ... // allocation magic

  if poisson_sample(size):
    s = stacktrace()
    mem_profile[s].allocs++
    mem_profile[s].alloc_bytes += size
    track_profiled(object, s)

  return object
```

### Block

Block profile 记录阻塞事件持续时间以及源头（stack trace 信息）。pprof 记录的阻塞事件包括：select 语句、chan 接收/发送、获取锁，但不包括 time.Sleep、GC 停顿、syscall（网络、文件 I/O）[^4]。采用频率可以通过 runtime.SetBlockProfileRate 设置，设置为 1 上是开启 block 采样。BlockProfileRate 的值（单位 ns）影响采样概率，比如设置 rate 纳秒，而实际阻塞 duration 纳秒，那么这次阻塞有 duration/rate 的概率被选中记录下来，否则丢弃。

```go
func main() {
	// 设置 rate = 2s，一半的概率记录阻塞持续 1s 的事件，
	// 一半概率丢弃
	runtime.SetBlockProfileRate(1000 * 1000 * 1000 * 2)

	// 阻塞 goroutine，阻塞 1s
	go func() {
		select {
		case <-time.After(1 * time.Second):
		}
	}()

	http.ListenAndServe(":9090", nil)
}
```

以下是阻塞事件采样底层实现简化代码，有助于我们理解 BlockProfileRate 的含义：

```bash
func chansend(channel, msg):
  if ready(channel):
    send(channel, msg)
    return

  start = now()
  wait_until_ready(channel) // Off-CPU Wait
  duration = now() - start

  // 埋点采样
  if random_sample(duration, rate):
    s = stacktrace()
    // note: actual implementation is a bit trickier to correct for bias
    block_profile[s].contentions += 1
    block_profile[s].delay += duration

  send(channel, msg)
```

```bash
func random_sample(duration, rate):
  if rate <= 0 || (duration < rate && duration/rate > rand(0, 1)):
    return false
  return true
```

### Mutex

Mutex profile 记录锁竞争的情况，和 Block profile 还不一样。运行下面代码，会发现 Block 记录的是 6 秒，Mutex 记录的是 4 秒。这是因为，Block 记录的是 Lock 锁等待阻塞的时间，第一个 goroutine 无须等待，第二个 goroutine 等待前一个 goroutine 释放，耗时 2 秒，第三个 goroutine 等待前两个锁占用释放，耗时 4 秒，因此 0+2+4=6；而 Mutex 记录的是每个锁请求者，从 Lock 到 Unlock 的时间，而最后一个 goroutine 因为不存在锁竞争，无须纳入统计，因此 2+2=4。

```go
// 分别查看 blcok 和 mutex profile
// go tool pprof -http 127.0.0.1:8080 http://127.0.0.1:9090/debug/pprof/block
// go tool pprof -http 127.0.0.1:8081 http://127.0.0.1:9090/debug/pprof/mutex
func main() {
    runtime.SetBlockProfileRate(1)
    runtime.SetMutexProfileFraction(1)
	
    m := &sync.Mutex{}
    for i := 0; i < 3; i++ {
        go func() {
            m.Lock()
            time.Sleep(2 * time.Second)
            m.Unlock()
        }()
    }

    http.ListenAndServe(":9090", nil)
}
```

Block 结果：

![intro-pprof-1](/images/intro-pprof-1.png)

Mutex 结果：

![intro-pprof-2](/images/intro-pprof-2.png)

以下是锁竞争采样底层实现简化代码，有助于我们理解 Block 与 Mutex 的区别：

```bash
func semacquire(lock):
  if lock.take():
    return

  start = now()
  waiters[lock].add(this_goroutine(), start)
  wait_for_wake_up()
```

```bash
func semrelease(lock):
  next_goroutine, start = waiters[lock].get()
  if !next_goroutine:
    // If there weren't any waiting goroutines, there is no contention to record
    return

  duration = now() - start
  if rand(0,1) < 1 / rate:
    s = stacktrace()
    mutex_profile[s].contentions += 1
    mutex_profile[s].delay += duration

  wake_up(next_goroutine)
```

### Goroutine

统计当前 goroutine 情况。

## 可视化

Profile 可视化有两种：调用图和火焰图。

### 调用图

调用图（callgraph）[^5][^6]展示栈内的函数调用关系。一个调用图中包含若干节点（node）和边（edge）。红灰颜色区分代表正值和近乎零值，节点字体大小采样值的大小，边的宽度代表调用过程资源的消耗情况。

![intro-pprof-3.png](/images/intro-pprof-3.png)

![intro-pprof-4.png](/images/intro-pprof-4.png)

### 火焰图

火焰图（flame graph）[^7]

### 文本

?debug=0  pb数据类型
?debug=1  legacy text 类型
// The debug parameter enables additional output.
// Passing debug=0 writes the gzip-compressed protocol buffer described
// in https://github.com/google/pprof/tree/master/proto#overview.
// Passing debug=1 writes the legacy text format with comments
// translating addresses to function names and line numbers, so that a
// programmer can read the profile without tools.

## Profile 格式

```bash
protoc --decode perftools.profiles.Profile  --proto_path /go/pkg/mod/github.com/google/pprof/proto  /go/pkg/mod/github.com/google/pprof/proto/profile.proto < block.profile
```

> 注意，不可以省略 --proto_path，否则会报错：\
> /go/pkg/mod/github.com/google/pprof/proto/profile.proto: File does not reside within any path specified using --proto_path (or -I).  You must specify a --proto_path which encompasses this file.  Note that the proto_path must be an exact prefix of the .proto file names -- protoc is too dumb to figure out when two paths (e.g. absolute and relative) are equivalent (it's harder than you think).

每一个 sample 代表一次采样?什么是 sample
A sample is a measurement. This measure is made at a certain time during the profiling process.
(sample 记录一个完整调用栈)
sample vs. location【function】

https://www.polarsignals.com/blog/posts/2021/08/03/diy-pprof-profiles-using-go/

## Profile 分析工具

24种 output
node()

flat time:function的自己耗时
cum time：包括等待调用返回的时间
sum: 占总时间的百分比

https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/


```text
pprof list
(pprof) list func1
Total: 2.67s
ROUTINE ======================== main.main.func1 in /Users/xxx/go/src/awesomeProject/main.go
2.65s   2.67s (flat, cum)   100% of Total
.          .     20:   for i := 0; i < 3; i++ {
.          .     21:           idx := i
.          .     22:           wg.Add(1)
.          .     23:           go pprof.Do(context.Background(), pprof.Labels("idx", fmt.Sprintf("%d", idx)), func(ctx context.Context) {
.          .     24:                   sum := 0
2.65s   2.65s    25:                   for d := 0; d < 3_000_000_000; d++ {
.          .     26:                           sum += d
.          .     27:                   }
.       20ms     28:                   wg.Done()
.          .     29:           })
.          .     30:   }
.          .     31:
.          .     32:   wg.Wait()
.          .     33:}
```

[^1]: [Using profiler labels](https://www.jetbrains.com/help/go/using-profiler-labels.html#viewing-labels-in-goland)
[^2]: [go runtime 包](https://pkg.go.dev/runtime#pkg-variables)
[^3]: [DataDog go pprof notes](https://github.com/DataDog/go-profiler-notes/tree/main/guide#memory-profiler)
[^4]: [Block Profiling in Go](https://github.com/DataDog/go-profiler-notes/blob/main/block.md)
[^5]: [Practical Go Lessons - Chapter 36: Program Profiling](https://www.practical-go-lessons.com/chap-36-program-profiling#display-profile-in-a-web-browser)
[^6]: [Interpreting the Callgraph](https://github.com/google/pprof/blob/master/doc/README.md)
[^7]: [The Flame Graph](https://queue.acm.org/detail.cfm?id=2927301)