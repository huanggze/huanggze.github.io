---
title: "从源码中学习优雅 Go 编程"
date: 2022-09-17T16:18:32+08:00
---

## 1. 巧妙使用 Context 包的 Value 函数

Go [net 包源码](https://github.com/golang/go/blob/go1.17.1/src/net/lookup.go#L277-L283)中有一段如下代码：

```go
func (r *Resolver) lookupIPAddr(ctx context.Context, network, host string) ([]IPAddr, error) {
	// ...

	// The underlying resolver func is lookupIP by default but it
	// can be overridden by tests. This is needed by net/http, so it
	// uses a context key instead of unexported variables.
	resolverFunc := r.lookupIP
	if alt, _ := ctx.Value(nettrace.LookupIPAltResolverKey{}).(func(context.Context, string, string) ([]IPAddr, error)); alt != nil {
        resolverFunc = alt
    }
	
	// ...
}
```

他的主要目的是，不给 Resolver 增加成员变量，但同时能满足[一些 go test](https://github.com/golang/go/blob/go1.17.1/src/net/http/transport_test.go#L4859-L4861) 需要自定义 lookupIP 函数的需求：

```go
func TestTransportMaxIdleConns(t *testing.T) {
    // ...

	ctx := context.WithValue(context.Background(), nettrace.LookupIPAltResolverKey{}, func(ctx context.Context, _, host string) ([]net.IPAddr, error) {
        return []net.IPAddr{{IP: net.ParseIP(ip)}}, nil
    })


    // ...
}
```