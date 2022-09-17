---
title: "Go DNS 查询源码解析：LookupIP"
date: 2022-09-17T17:48:53+08:00
---

GO DNS lookup 代码位于 [net/lookup.go](https://github.com/golang/go/blob/go1.17.1/src/net/lookup.go#L206) 下 `func (r *Resolver) LookupIPAddr(ctx context.Context, host string) ([]IPAddr, error)` 函数中。这个函数有几个设计和特性，非常值得学习：

1. 能对相同 Host 的查询请求进行合并，避免重复请求；
2. 使用 channel + select 语句，实现优雅的同步请求；
3. 使用 Cancel、Deadline Context，实现终止请求；
4. Context 取消，不会影响到其他合并的请求得到结果；
5. 底层使用 cgo 来调用 C 语言代码执行 DNS 查询。

以下我们将逐步介绍 lookupIPAddr 是核心代码：

```go
// LookupIPAddr looks up host using the local resolver.
// It returns a slice of that host's IPv4 and IPv6 addresses.
func (r *Resolver) LookupIPAddr(ctx context.Context, host string) ([]IPAddr, error) {
return r.lookupIPAddr(ctx, "ip", host)
}
```

## Resolver

首先，该函数是 Resolver 的成员方法，Resolver 结构体是一个 DNS 解析器，它包含以下字段：

```go
type Resolver struct {
    PreferGo bool // 是否使用 Go 实现的 DNS 解析器，还是用 CGO 调用 C 语言的代码
	Dial func(ctx context.Context, network, address string) (Conn, error) // Go 实现的 DNS 解析器会调用该函数来，来发起 DNS 请求

    lookupGroup singleflight.Group // 用于合并相同 host 的 DNS 查询
}
```

注意，因为 lookupGroup 字段，才得以实现对相同 Host 的查询请求进行合并，避免重复请求。`singleflight.Group` 其实是一个 map，key 无疑是 host 字符串，value 是发起的请求：

```go
package singleflight

// Group 用于去重
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

// call is an in-flight or completed singleflight.Do call
// call 代表一次 DNS 查询请求
type call struct {
    // ...
	
	dups  int // 记录重复请求次数
	chans []chan<- Result // 存放查询结果
}

// Result holds the results of Do, so they can be passed
// on a channel.
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}
```

## lookupIPAddr

LookupIPAddr 实际请求的是 lookupIPAddr 内部方法，该方法代码骨架如下：

```go
var dnsWaitGroup sync.WaitGroup

func (r *Resolver) lookupIPAddr(ctx context.Context, network, host string) ([]IPAddr, error) {
	// ...

	// DNS 查询函数
	resolverFunc := r.lookupIP
	
	// 真正执行 DoChan 的时候，不是直接用 ctx，而是再创建一个 lookupGroupCtx
	// 这是为了避免，ctx 被取消，影响到其他并发的查询。
	// 可以看到 withUnexpiredValuesPreserved 中另起了一个 context.Background()
	lookupGroupCtx, lookupGroupCancel := context.WithCancel(withUnexpiredValuesPreserved(ctx))
	
	lookupKey := network + "\000" + host
	// 发起请求加入 WaitGroup 中等待执行完成
	dnsWaitGroup.Add(1)
	// 如果查询成功，结果会写入 ch；如果是重复请求，called 返回 false
	// getLookupGroup 没有什么特别的，就是返回 singleflight.Group
	ch, called := r.getLookupGroup().DoChan(lookupKey, func() (interface{}, error) {
	    defer dnsWaitGroup.Done()
	    return testHookLookupIP(lookupGroupCtx, resolverFunc, network, host)
	})
	if !called {
	    dnsWaitGroup.Done()
	}

    select {
    case <-ctx.Done():
		// 由于 Context 取消，请求结束并报错
	    if r.getLookupGroup().ForgetUnshared(lookupKey) {
            lookupGroupCancel()
        } else {
            go func() {
                <-ch
                lookupGroupCancel()
            }()
        }
	    err := mapErr(ctx.Err())
        return nil, err
    case r := <-ch:
        return lookupIPReturn(r.Val, r.Err, r.Shared)
    }
}
```

首先，dnsWaitGroup 运行并记录多个 DNS 查询。当 lookupIPAddr 被并发调用时，dnsWaitGroup 会多次 `dnsWaitGroup.Add(1)`。

实际执行查询的，是通过 DoChan 方法，我们下面再详细介绍。在之前，我们看看最后 select 的语句。这个 select 语句有两个 case。第一个 case 判断 ctx 是否发生了取消。如果发生了，那么就执行取消 DNS 查询。但注意，我们开头说过，Go 这里是做了多个并发请求合并，那么要在 ForgetUnshared() 里进一步判断，是否只有自己在查询。只有自己就直接取消，不是的话，仍等待查询完成，因为其他并发的请求，还要等待结果。

最后，结果由 lookupIPReturn() 函数包装后返回。

## DoChan

现在，我们重点看下 DoChan。DoChan 执行一次 DNS 查询，实际执行时调用入参函数 fn：

```go
func (g *Group) DoChan(key string, fn func() (interface{}, error)) (<-chan Result, bool) {
    ch := make(chan Result, 1)
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
        c.dups++
        c.chans = append(c.chans, ch)
        g.mu.Unlock()
        return ch, false
    }
    c := &call{chans: []chan<- Result{ch}}
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()
    
    go g.doCall(c, key, fn)
    
    return ch, true
}

// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
    c.val, c.err = fn()
    c.wg.Done()
    
    g.mu.Lock()
    delete(g.m, key)
	// 给并发请求中的每个 ch 都发一次数据，
	// 因为，所有的结果等待 channel 都在这里，每个都要发一份数据
    for _, ch := range c.chans {
        ch <- Result{c.val, c.err, c.dups > 0}
    }
    g.mu.Unlock()
}
```

DoChan 的执行结果会放到 ch 中，ch 是一个长度为，存放 Result 结果的 channel。为避免并发请求，g.m[key] 被用于去重。如果是重复的，则把 call 的结果暂存 channel c.chans 加一（`c.chans = append(c.chans, ch)`）。不是重复的，则起一个协程去查询 `go g.doCall(c, key, fn)`。doCall 实际调用 fn()，也就是前面代码中 `r.lookupIP` 这个函数。

## lookupIP

可以预料到，lookupIP 会调用系统调用来查询 DNS。那么它源码具体怎么实现的呢？Go 既有自己实现方式（goLookupIP），也有通过 CGO 调用 C 语言代码实现（cgoLookupIP）。一般只在特殊需求的 debug 时，才会让 PreferGo 为 true。

```go
func (r *Resolver) lookupIP(ctx context.Context, network, host string) (addrs []IPAddr, err error) {
	if r.preferGo() {
		return r.goLookupIP(ctx, network, host)
	}
	order := systemConf().hostLookupOrder(r, host)
	if order == hostLookupCgo {
		if addrs, err, ok := cgoLookupIP(ctx, network, host); ok {
			return addrs, err
		}
		// cgo not available (or netgo); fall back to Go's DNS resolver
		order = hostLookupFilesDNS
	}
	ips, _, err := r.goLookupIPCNAMEOrder(ctx, network, host, order)
	return ips, err
}
```

order 定义了以什么模式执行查询。由于 mac 电脑 order 一定会等于 hostLookupCgo。后面 cgoLookupIP() 涉及 CGO 知识，不再深入介绍。cgoLookupIP() 最终返回查询结果。

```go
// systemConf returns the machine's network configuration.
func systemConf() *conf {
    confOnce.Do(initConfVal)
    return confVal
}

func initConfVal() {
	// ...
	
    // Darwin pops up annoying dialog boxes if programs try to do
    // their own DNS requests. So always use cgo instead, which
    // avoids that.
    if runtime.GOOS == "darwin" || runtime.GOOS == "ios" {
        confVal.forceCgoLookupHost = true
        return
    }
	
	// ...
}
```