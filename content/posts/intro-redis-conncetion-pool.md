---
title: "Redis go客户端连接池实现"
date: 2022-09-25T21:00:31+08:00
---

Redis 连接池是维护 Redis 连接的缓存，当需要对 Redis 发出请求时可以复用连接，减少因为新 TCP 连接的建立带来的开销，从而提高性能。我们在最近的压测中发现，我们使用的第三方库 redigo 连接池，有时候，从连接池中拿到一个连接的耗时，比 redis 命令实际执行时间还长。通过本文的源码分析可以知，阻塞点有两个：

1. 连接池大小 MaxActive 如果太小，会导致获取连接时，活跃连接达到上限，程序阻塞在 waitVacantConn() 函数；
2. 连接池设置了 TestOnBorrow（如：通过 Redis PING 命令测试连接可用性），则获取连接时，会有一次额外网络开销来执行 Redis PING。并且如果 PING 失败，redigo 会继续遍历空闲连接并再次 PING，直到找到一个可用连接返回。

同时，我们还对比了，[redigo](https://github.com/gomodule/redigo) 和 [redis v8](https://github.com/go-redis/redis) 两个 go Redis 连接池库，发现 redigo 不设置 TestOnBorrow，redis v8 和相比 redigo 性能相差不大。但生产环境一般会设置 TestOnBorrow，这样，redis v8 性能优势明显。而且，redis v8 社区活跃度高，功能也比 redigo 多，在今后的开发中，仍应优先使用 redis v8。

本文介绍两个连接池的设计和源码分析，重点指出可能导致程序阻塞的瓶颈点。

## 性能对比

首先，我们看一下两个库的性能对比大小，实验参数和方法如下：

- Redis 服务端版本 v6.2.4，机器性能48核126G;
- Redis 客户端 redigo 和 redis v8 连接池大小都设置为 256；
- 使用 go parallel benchmark 框架做并发测试；
- 客户端所在机器性能40核64G；
- go 版本 1.16.10；
- 每项实验执行压测 10s；

### 实验一：redigo 连接池不设置 TestOnBorrow

对比两种库实验结果如下：

||redigo|redis v8|
|---|---|---|
|HGETALL|8902 ns/op|8225 ns/op|
|HMSET|13572 ns/op|13618 ns/op|

分析压测数据，对比读写命令，redis v8 和 redigo 速度相差不大（单位：nanoseconds/operation。由于是并发测试，总体 ns/op 会随并发度提高而下降，这里我们固定并发度 `SetParallelism(1000)`，具体 go benchmark 框架使用细节可看官方文档）。

### 实验二：redigo 连接池设置 TestOnBorrow

```go
TestOnBorrow: func(c redigo.Conn, t time.Time) error {
    _, err := c.Do("PING")
    return err
}
```

||redigo|redis v8|
|---|---|---|
|HGETALL|13117 ns/op|8225 ns/op|
|HMSET|17140 ns/op|13618 ns/op|

此时可以看到 redigo 比 redis v8 差很多。如果埋点分析，可进一步看到 redigo 耗时更多主要原因在于连接池的竞争，而实际 redis 命令执行耗时，两种库差别不大。

## 压测代码示例

使用 redis v8 测试 HGETALL 的代码：

```go
func Benchmark_Parallel_RedisV8_HGETALL(b *testing.B) {
	pool := initRedisV8()

	b.SetParallelism(1000)
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			ctx := context.Background()
			reply := pool.HGetAll(ctx, hashKey)
			
			_, err := reply.Result()
			if reply.Err() != nil || err != nil {
				fmt.Printf("HGETALL error: %v\n", err)
			}
		}
	})

	err := pool.Close()
	if err != nil {
		fmt.Printf("Close pool error: %v\n", err)
	}
}
```

实验结果示例：

```shell
$ ./main -test.bench "Benchmark_Parallel_RedisV8_HGETALL" -test.benchtime 10s
goos: linux
goarch: amd64
pkg: main
cpu: Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz
Benchmark_Parallel_RedisV8_HGETALL-40     1422176      8225 ns/op
PASS
```

## 连接池设计

连接池需要设计包含以下功能：

1. 连接池初始化：设置连接池和底层 TCP socket 的建立参数。除此之外，像 redis v8 还会提前预热一批空闲连接（MinIdleConns）；
2. 连接的获取与释放：每次使用完连接，都需要归回到连接池。redis v8 通过双端队列来管理，而 redigo 通过双向链表来实现；
3. 建立新连接：本质上一个 Redis 的连接是封装了 go 标准库中的 `net.Conn`。底层通过调用 `net.Dial("tcp", "127.0.0.1:6379")` 来建立一次新的且可以复用的 TCP 连接。Dial() 更底层会有 DNS 查询、通关 CGO 调用 C 代码和系统调用来完成 TCP 建立；
4. 向连接读写 Redis 命令：Redis 有自己的协议，发送 Redis 命令即是向 `net.Conn` 连接 write/read 二进制数据。

对比 redigo 和 redis v8 源码，可以看到两个库，在 1、3、4 这四点上设计没有什么不同，唯一区别在于第 2 点，连接池管理的设计。

如以下是 redigo 和 redis v8 封装的连接 Conn，本质上其实都是封装了 go 标准库中的 `net.Conn`，并使用 bufio 包的 Reader 和 bufio.Writer 封装读写`net.Conn`操作（如 `bufio.NewReader(netConn)`）。这些部分都是相同的。

```go
// redigo
type conn struct {
    // Shared
    mu      sync.Mutex
    pending int
    err     error
    conn    net.Conn // 底层封装 go 标准库中的 `net.Conn`
    
    // Read
    readTimeout time.Duration // 封装 Reader
    br          *bufio.Reader
    
    // Write
    writeTimeout time.Duration
    bw           *bufio.Writer
    
    // ...省略其他字段
}

// redis v8
type Conn struct {
	usedAt  int64 // atomic
	netConn net.Conn

	rd *proto.Reader
	bw *bufio.Writer
	wr *proto.Writer

	Inited    bool
	pooled    bool
	createdAt time.Time
}
```

## Redigo Pool

接下来，我们重点看看两个库连接池管理的设计，以下是 redigo 的 Pool 定义：

```go
type Pool struct {
    MaxIdle   int
	MaxActive int
    // ...省略部分公开字段
	
    mu           sync.Mutex // 连接池互斥锁
    closed       bool // 连接池是否已关闭
    active       int
    initOnce     sync.Once
    ch           chan struct{} // channel 队列，大小等于 MaxActive，用于控制等待空闲连接。当有连接释放时，会向队列放入一个struct{}{}；当获取连接时，会向 ch 拿 struct{}{}，没有则阻塞
    idle         idleList    // 是双向链表，组织和管理连接
    waitCount    int64 // 用于监控
    waitDuration time.Duration // 用于监控
}

type idleList struct {
    count       int
    front, back *poolConn
}
```

可以看到 redigo 管理连接的方式是维护了一个双向链表 `idleList`，并提供 get()、put() 用于取和归还连接，以下代码省略部分内容：

```go
// GetContext 返回一个请求
func (p *Pool) GetContext(ctx context.Context) (Conn, error) {
    // 等到直到当前连接数小于 MaxActive，
	// 由 Pool.ch 字段（大小为 MaxActive 的 channel）控制等待与放行
    waited, err := p.waitVacantConn(ctx)
    if err != nil {
        return errorConn{err}, err
    }
    
    p.mu.Lock()
	
    // 从空闲链表首部开始遍历，不可用的连接关闭，并继续遍历，
	// 返回第一个可用连接
    for p.idle.front != nil {
        pc := p.idle.front
        p.idle.popFront()
        p.mu.Unlock()
        if (p.TestOnBorrow == nil || p.TestOnBorrow(pc.c, pc.t) == nil) &&
        (p.MaxConnLifetime == 0 || nowFunc().Sub(pc.created) < p.MaxConnLifetime) {
            return &activeConn{p: p, pc: pc}, nil
        }
        pc.c.Close()
        p.mu.Lock()
        p.active--
    }
    
    p.active++
    p.mu.Unlock()
	// 如果仍找不到，则尝试通过 dial() 建立一个新 TCP 连接
    c, err := p.dial(ctx)
    if err != nil {
        p.mu.Lock()
        p.active--
        if p.ch != nil && !p.closed {
            p.ch <- struct{}{}
        }
        p.mu.Unlock()
        return errorConn{err}, err
    }
    return &activeConn{p: p, pc: &poolConn{c: c, created: nowFunc()}}, nil
}
```

通过阅读源码，我们可以发现，当高并发请求时，可知有两个阻塞点：

1. 大量请求阻塞在上面代码中的 `p.waitVacantConn(ctx)` 函数，因为连接池设置了上限；
2. `p.TestOnBorrow()` 会测试连接的可用性，

归还连接的函数 put() 将连接放回队首：

```go
// put 归还一个请求
func (p *Pool) put(pc *poolConn, forceClose bool) error {
	p.mu.Lock()
	if !p.closed && !forceClose {
		pc.t = nowFunc()
		// 用完的连接放回队首
		p.idle.pushFront(pc)
		// 如果链表大小大于 MaxIdle，则弹出一个队尾的连接
		if p.idle.count > p.MaxIdle {
			pc = p.idle.back
			p.idle.popBack()
		} else {
			pc = nil
		}
	}

	// 如果有弹出的队尾连接，执行连接关闭
	if pc != nil {
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
	return nil
}
```


## Redis v8 ConnPool

Redis v8 库的设计我们主要看下它的连接池定义和 get() 方法实现。

```go
type ConnPool struct {
    queue chan struct{} // channel 大小等于 PoolSize，空闲连接的数量
    
    connsMu   sync.Mutex
    conns     []*Conn
    idleConns []*Conn
    poolSize     int
    idleConnsLen int
}
```

get() 方法也是先有一个 waitTurn()，这和 redigo 的 waitVacantConn() 功能、实现一样。结束等待后，popIdle() 弹出一个可用连接，功能与 redigo 的 popFront() 函数相似，实现区别是前者是切片，后者是双向链表。

```go
// Get returns existed connection from the pool or creates a new one.
func (p *ConnPool) Get(ctx context.Context) (*Conn, error) {
    // 如果连接数超过 PoolSize 则阻塞等待
	if err := p.waitTurn(ctx); err != nil {
		return nil, err
	}

	for {
		p.connsMu.Lock()
		cn, err := p.popIdle()
		p.connsMu.Unlock()
		
		return cn, nil
	}
	
	newcn, err := p.newConn(ctx, true)
	if err != nil {
		p.freeTurn()
		return nil, err
	}

	return newcn, nil
}
```

特别注意，redis v8 没有类似 TestOnBorrow 的支持。那如果这个连接不可用，是怎么处理的呢？其实，redis v8 在 [process()](https://github.com/go-redis/redis/blob/v8.11.5/redis.go#L306) 里会尝试重试，默认最大重试 MaxRetries = 3 次，重试间隔可设置 MinRetryBackoff、MaxRetryBackoff。这与 redigo TestOnBorrow 先测试连接健康，再复用该连接的思路不同。redis v8 对于从连接池中拿到的连接不做检查，如果执行失败，再重试。

```go
func (c *baseClient) process(ctx context.Context, cmd Cmder) error {
	var lastErr error
	for attempt := 0; attempt <= c.opt.MaxRetries; attempt++ {
		attempt := attempt

		retry, err := c._process(ctx, cmd, attempt)
		if err == nil || !retry {
			return err
		}

		lastErr = err
	}
	return lastErr
}

func (c *baseClient) _process(ctx context.Context, cmd Cmder, attempt int) (bool, error) {
	// 重试间隔
    if attempt > 0 {
        if err := internal.Sleep(ctx, c.retryBackoff(attempt)); err != nil {
            return false, err
        }
    }

    // ... 省略命令执行逻辑代码

	retry := shouldRetry(err, atomic.LoadUint32(&retryTimeout) == 1)
    return retry, err
}
```

而是否应该重试，由 [shouldRetry()](https://github.com/go-redis/redis/blob/v8.11.5/error.go#L28) 函数比较 err 类型来决定：

```go
func shouldRetry(err error, retryTimeout bool) bool {
	switch err {
	case io.EOF, io.ErrUnexpectedEOF:
		return true
	case nil, context.Canceled, context.DeadlineExceeded:
		return false
	}

	if v, ok := err.(timeoutError); ok {
		if v.Timeout() {
			return retryTimeout
		}
		return true
	}
	
    s := err.Error()
    if s == "ERR max number of clients reached" {
        return true
    }
	
    // ...省略部分case
	
    if strings.HasPrefix(s, "TRYAGAIN ") {
        return true
    }
    
    return false
}
```

## 总结

我们分析了 redigo 和 redis v8 连接池的工作机制和阻塞点，通过实验对比了两者性能差距，我们建议在开发中，应优先使用 redis v8 库。