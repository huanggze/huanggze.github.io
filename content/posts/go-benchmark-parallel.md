---
title: "Go 并发基准测试中的坑"
date: 2022-09-18T23:11:52+08:00
---

最近在用 Go 的并发基准测试框架时，发现了一些坑。比如以下代码[^1]，执行基准测试，结果发现，随着并发度越高 `b.SetParallelism(i)`，ns/op 数值越小。然而我要做基准测试的函数，是 sleep 1ms，并无任何并发优化。要想理解为什么随着并发度翻倍，ns/op 结果减半，需要看看 Go benchmark 源码是如何计算 ns/op 的。

```go
func sleep() {
    time.Sleep(1 * time.Millisecond)
}

func BenchmarkSleepParallel(b *testing.B) {
    for i := 1; i <= 8; i *= 2 {
        b.Run(strconv.Itoa(i), func(b *testing.B) {
            b.SetParallelism(i)
            b.RunParallel(func(pb *testing.PB) {
                for pb.Next() {
                    sleep()
                }
            })
        })
    }
}
```

```shell
$ go test -bench BenchmarkSleepParallel .           
goos: darwin
goarch: amd64
pkg: main
cpu: Intel(R) Core(TM) i5-1038NG7 CPU @ 2.00GHz
BenchmarkSleepParallel/1-8          8155            156454 ns/op
BenchmarkSleepParallel/2-8         15375             79527 ns/op
BenchmarkSleepParallel/4-8         30499             40130 ns/op
BenchmarkSleepParallel/8-8         59118             20065 ns/op
PASS
ok    main  7.045s
```

Go benchmark 结果打印可见[源码](https://github.com/golang/go/blob/go1.17.1/src/testing/benchmark.go#L418)和设计文档[^2]：

```go
func (r BenchmarkResult) String() string {
    buf := new(strings.Builder)
    fmt.Fprintf(buf, "%8d", r.N)

    // Get ns/op as a float.
    ns, ok := r.Extra["ns/op"]
    if !ok {
        ns = float64(r.T.Nanoseconds()) / float64(r.N)
    }
    if ns != 0 {
        buf.WriteByte('\t')
        prettyPrint(buf, ns, "ns/op")
    }

    // ...
    
    return buf.String()
}
```

从源码中可知，ns/op 的计算方式为，总时间 / 总迭代次数（r.T.Nanoseconds() / r.N）。显然，总迭代次数会随并发数翻倍而翻倍，导致 ns/op 下降一半。所以，做并发基准测试时，需要注意有这个坑，ns/op 可能并不准确。 

[^1]: [Benchmarking Results in Go](https://www.mikenewswanger.com/posts/2018/benchmarking-in-go/)
[^2]: [Proposal: Go Benchmark Data Format](https://go.googlesource.com/proposal/+/master/design/14313-benchmark-format.md)