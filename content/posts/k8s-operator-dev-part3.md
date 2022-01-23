---
title: "K8s Operator 开发（三）：Cache 机制"
date: 2022-01-19T17:45:22+08:00
toc: true
categories: ["Kubernetes"]
series: ["Kubernetes-operator-开发"]
---

## Cache 

上一篇谈到 Kubebuilder 使用的是读写分离的客户端。读客户端采用了 Cache 设计。通过深入追踪 main.go 的 ctrl.NewManager 代码，我们可以看到 Kubebuilder 使用 cotroller-runtime 包构造并持有了一个 [Cache 接口实例](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.11.0/pkg/cache/cache.go#L136)。Cache 接口包含两个组件：Reader 和 Informers。而 Informers 本身内嵌了 FieldIndexer 接口。

```go
// Cache knows how to load Kubernetes objects, fetch informers to request
// to receive events for Kubernetes objects (at a low-level),
// and add indices to fields on the objects stored in the cache.
type Cache interface {
	// Cache acts as a client to objects stored in the cache.
	client.Reader

	// Cache loads informers and adds field indices.
	Informers
}
```

所以，Cache 一共提供方法如下。Reader 接口很简单，提供 Get 和 List 方法。Infomers 是新概念，Informers 保持的是一组 Informer。Cache 对象可以通过 GetInformer/GetInformerForKind 方法获得。Start 用于 Informers 的初始化和启动。WaitForCacheSync 等待缓存同步完成。

1. Get(ctx context.Context, key ObjectKey, obj Object) error
2. List(ctx context.Context, list ObjectList, opts ...ListOption) error
3. GetInformer(ctx context.Context, obj client.Object) (Informer, error)
4. GetInformerForKind(ctx context.Context, gvk schema.GroupVersionKind) (Informer, error)
5. Start(ctx context.Context) error
6. WaitForCacheSync(ctx context.Context) bool
7. IndexField(ctx context.Context, obj Object, field string, extractValue IndexerFunc) error

> Informer 的启动：\
> 实际上调用 Start()（运行在一个 [Runnable](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.10.0/pkg/manager/internal.go#L696) 后台 goroutine 中）不会真正把各个 informer 跑起来，即[源码](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.10.0/pkg/cache/internal/informers_map.go#L150-L152)中 for 循环 `go informer.Informer.Run(ctx.Done())` 不会进入，因为此时 informersByGVK 内容为空。真正 Informer.Run() 是在 [GetInformer()](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.10.0/pkg/source/source.go#L123) （以及 Get/List/GetInformerForKind）时，底层调用 [addInformerToMap()](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.10.0/pkg/cache/internal/informers_map.go#L194) 方法延迟运行。

前面提到 Informers 保持的是一组 Informer。每个 Informer 完成两个工作：1. 监听一种 K8s 资源的状态；2. 把状态保存到缓存（in-memory cache）中。资源监听是通过 ListAndWatch 机制拿到资源的最新状态，而不是轮训 kube-apiserver。后者，K8s 资源和 Informer 的缓存通过 map 映射，[以 GVK 作为 map 的 key](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.11.0/pkg/cache/internal/informers_map.go#L97)，保存在 specificInformersMap（Informer 的底层实现）的 informersByGVK 字段。获取一个具体的 Informer 缓存可以通过 Informers 的 GetInformerForKind() 方法，底层会调用 specificInformersMap 实现的 [Get()]((https://github.com/kubernetes-sigs/controller-runtime/blob/v0.11.0/pkg/cache/internal/informers_map.go#L184)) 获取。Get 方法返回一个 *MapEntry 引用对象。MapEntry 封装了一个 client-go 库的 cache 包下的 SharedIndexInformer 接口和一个 CacheReader 结构体（Reader 的接口实现）。

```go
type MapEntry struct {
	// Informer is the cached informer
	Informer cache.SharedIndexInformer

	// CacheReader wraps Informer and implements the CacheReader interface for a single type
	Reader CacheReader
}
```

> 为什么 MapEntry 两个字段一个是接口类型，一个是结构体类型？\
> cache.SharedIndexInformer 的接口实现都包内访问（首字母是小写的结构体），所以只能用接口类型来声明；而能直接使用 CacheReader 结构体的，尽量选择结构体类型声明，而不用 Reader 接口，避免不必要的反射且类型安全。

由于前面提到的懒加载，MapEntry 的初始化在 GetInformer 时调用 addInformerToMap() 才完成。可以看到 MapEntry 的两个字段 Informer 和 Reader 共享一个 Indexer。Indexer 实际上是带索引的缓存。此处 **Indexer 正是本节要讨论的所谓 Cache**（终于定位到正确的源码了）。

```go
func (ip *specificInformersMap) addInformerToMap(gvk schema.GroupVersionKind, obj runtime.Object) (*MapEntry, bool, error) {
	//...
	// 创建的 SharedIndexInformer 带有一个 Indexer
	// Indexer 可通过 SharedIndexInformer 的 GetIndexer 方法返回
	ni := cache.NewSharedIndexInformer(lw, obj, resyncPeriod(ip.resync)(), cache.Indexers{
        cache.NamespaceIndex: cache.MetaNamespaceIndexFunc,
    })
	// ...
	
    i := &MapEntry{
        Informer: ni,
        Reader: CacheReader{
            indexer:          ni.GetIndexer(),  // 共享 Indexer
            groupVersionKind: gvk,
            scopeName:        rm.Scope.Name(),
            disableDeepCopy:  ip.disableDeepCopy.IsDisabled(gvk),
        },
    }
    ip.informersByGVK[gvk] = i
	
	//...
}
```

虽然本篇重点讲 Cache 实现以及 Indexer，还未全面介绍 Informer（Cache 是 Informer 的组件），但我们在阅读 controller-runtime 源码时会看到 Informer 接口有两处定义：controller-runtime 库不仅自身定义了 [Informer 接口](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.10.0/pkg/cache/cache.go#L73)，还有一处引用了 k8s.io/client-go/tools/cache 包的 SharedIndexInformer 接口，即 MapEntry 的 Informer 字段。为什么 client-go 库和 controller-runtime 库都定义 Informer 接口（client-go 库称为 Informer，controller-runtime 库称为 SharedIndexInformer）？这两个库的区别与关系是：controller-runtime 库是 kubebuilder 框架的基础。controller-runtime 提供了很多脚手架工具。controller-runtime 库对 client-go 库进一步封装。client-go 更底层。也可以直接使用 client-go 写控制器，详见 [sample-controller](https://github.com/kubernetes/sample-controller/blob/master/main.go) 源码。通常使用 controller-runtime 更多。

|Informer|SharedIndexInformer|说明|
|---|---|---|
|**`import toolscache "k8s.io/client-go/tools/cache"`**|||
|`AddEventHandler(handler toolscache.ResourceEventHandler)`|`AddEventHandler(handler ResourceEventHandler)`|参考 AddEventHandlerWithResyncPeriod|
|`AddEventHandlerWithResyncPeriod(handler toolscache.ResourceEventHandler, resyncPeriod time.Duration)`|`AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)`|添加事件监听器。在 Informer 启动（也即 Controller 启动时）时会被调用|
|`AddIndexers(indexers toolscache.Indexers) error`|`AddIndexers(indexers Indexers) error`|添加索引器|
||`GetIndexer() Indexer`|返回带索引缓存|
||`GetStore() Store`|返回（带索引）缓存，实现上同 [GetIndexer()](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/shared_informer.go#L433-L439)|
||`GetController() Controller`|未实现功能的空方法。由上层封装实现|
||`Run(stopCh <-chan struct{})`|未实现 SharedInformer 的启动。由上层封装实现|
|`HasSynced() bool`|`HasSynced() bool`|同上|
||`LastSyncResourceVersion() string`|同上|
||`SetWatchErrorHandler(handler WatchErrorHandler) error`||

## Indexer

Indexer 是[带索引的缓存 Store](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/index.go#L35)。Indexer 接口的底层实现是 [threadSafeMap](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/thread_safe_store.go#L63)。threadSafeMap 包含三个部分：indexers、indices、items：

1. indexers：一组索引器。索引器计算用于计算待保存对象的索引； 
2. indices：一组索引。索引记录索引与对象的 ID 映射；
3. items：实际保存对象的变量。是对象 ID 与对象的映射。

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```

### Indexers

Indexers 是索引器，索引器的 IndexFunc 函数接受一个 K8s 资源对象（如 Pod）计算该对象的索引。最常见的是按 namespace 建立索引，client-go 提供了该计算函数：[MetaNamespaceIndexFunc](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/index.go#L86)。也可以自定义按 node 建立索引。

![k8s-operator-dev-part3-1](/images/k8s-operator-dev-part3-1.png)

### Indices

Indices 是真正所谓的索引。Indices 是一组索引，每个索引记录索引名，索引名下的索引键值。比如如下图，使用 namespace 作为索引，namespace 有 default、kube-system 和 istio-system 三个。default 下又有三个对象，对象 ID 为 nginx-xxxx-xxx。对象 ID 的计算方式定义在 [KeyFunc](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/store.go#L269)，默认是 [MetaNamespaceKeyFunc](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/store.go#L104)，返回对象的 \<namespace\>/\<name\> 作为 ID（这里称 Key）。

![k8s-operator-dev-part3-2](/images/k8s-operator-dev-part3-2.png)

### Items

实际元素保存在 Items 中。Items 是对象的 Key 与对象实例的映射。

![k8s-operator-dev-part3-3](/images/k8s-operator-dev-part3-3.png)

### 增删改查

Indexer 支持对索引以及缓存元素的 CRUD。那么现在的问题是缓存的数据来源是哪？K8s 控制器不是通过 Get/List 请求直接访问 kube-apiserver，而是通过 [ListAndWatch](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/reflector.go#L254) 机制，先通过 [List](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/reflector.go#L277) 请求拿到请求资源的版本（resourceVersion），然后再发送 [Watch](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/reflector.go#L414) 请求，监听资源该版本之后的变化。下一篇我们详细介绍 ListAndWatch 机制。