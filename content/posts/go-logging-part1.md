---
title: "Go 日志：klog"
date: 2022-04-06T21:23:09+08:00
toc: true
categories: ["go"]
series: ["go-logging"]
---

klog 是 K8s 社区维护的 logging 库，支持在程序命令行注册以下 flag[^1]：

- log_file：输出到日志文件；
- log_file_max_size：日志文件最大大小（单位：mb），如果超过最大值则会擦除日志文件全部内容，并从头开始写日志。未设置最大值时，无限制；
- logtostderr：是否输出到标准错误输出，如果想输出到文件，改值应设为 false；
- skip_log_headers：忽略记录日志元信息；
- v：日志等级

## 示例 1：手动/自动输出日志到文件

```go
func main() {
	fs := flag.NewFlagSet("klog", flag.ExitOnError)
	// 注册 flag
	klog.InitFlags(fs)
	fs.Set("skip_log_headers", "true")
	// 解析 flag
	fs.Parse(os.Args[1:])
	klog.Info("nice to meet you")
	klog.ErrorS(errors.New("oops"), "noooo")
	// 手动刷新日志记录到文件
	klog.Flush()
	// 或者每 5s 自动 flush
	// time.Sleep(6 * time.Second)
}
```

```bash
$ go run main.go --log_file=out.log --logtostderr=false
$ cat out.log
I0406 21:20:32.475227   88398 main.go:15] nice to meet you
E0406 21:20:32.475666   88398 main.go:16] "noooo" err="oops"
```

## 示例 2：goroutine 溢出

前面的示例 1 没有停止 Flush 因此会有 groutine leaking 的问题，解决办法是 `defer klog.StopFlushDaemon()`。

```go
func main() {
	klog.InitFlags(nil)

	// By default klog writes to stderr. Setting logtostderr to false makes klog
	// write to a log file.
	flag.Set("logtostderr", "false")
	flag.Set("log_file", "myfile.log")
	flag.Parse()

	// Info writes the first log message. When the first log file is created,
	// a flushDaemon is started to frequently flush bytes to the file.
	klog.Info("nice to meet you")

	// klog won't ever stop this flushDaemon. To exit without leaking a goroutine,
	// the daemon can be stopped manually.
	klog.StopFlushDaemon()

	// After you stopped the flushDaemon, you can still manually flush.
	klog.Info("bye")
	klog.Flush()
}

func TestLeakingFlushDaemon(t *testing.T) {
	// goleak detects leaking goroutines.
	defer goleak.VerifyNone(t)

	// Without calling StopFlushDaemon in main, this test will fail.
	main()
}
```

## 示例 3：结构化日志

使用 InfoS、ErrorS 方法

```go
// I0406 23:42:44.499187    8424 main.go:18] "Pod status updated" pod="kubedns" status="ready"
klog.InfoS("Pod status updated", "pod", "kubedns", "status", "ready")
```

## 示例 4：输出到 Buffer

```go
func main() {
	klog.InitFlags(nil)
	flag.Set("logtostderr", "false")
	flag.Set("alsologtostderr", "false")
	flag.Parse()

	buf := new(bytes.Buffer)
	// 接收一个 io.Writer 参数
	klog.SetOutput(buf)
	klog.Info("nice to meet you")
	klog.Flush()

	fmt.Printf("LOGGED: %s", buf.String())
}
```

## 示例 4：klogr

klogr 实现了 [go-logr/logr](https://github.com/go-logr/logr) 接口

```go
type myError struct {
    str string
}

func (e myError) Error() string {
    return e.str
}

func main() {
    klog.InitFlags(nil)
    flag.Set("v", "3")
    flag.Parse()
    log := klogr.New().WithName("MyName").WithValues("user", "you")
    log.Info("hello", "val1", 1, "val2", map[string]int{"k": 1})
    log.V(3).Info("nice to meet you")
    log.Error(nil, "uh oh", "trouble", true, "reasons", []float64{0.1, 0.11, 3.14})
    log.Error(myError{"an error occurred"}, "goodbye", "code", -1)
}
```

[^1]: [https://github.com/kubernetes/klog](https://github.com/kubernetes/klog/blob/v2.60.1/klog.go#L407-L421)