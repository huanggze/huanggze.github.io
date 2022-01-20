---
title: "K8s Operator 开发（二）：Cache 机制"
date: 2022-01-19T17:45:22+08:00
draft: true
---

## Cache 

上一篇谈到 Kubebuilder 使用的是读写分离的客户端。读客户端采用了 Cache 设计。通过追踪 Operator 生成项目 main.go 入口里的 ctrl.NewManager 方法，我们可以看到 Kubebuilder 使用 cotroller-runtime 包 NEW 了一个 [Cache 接口实例](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.11.0/pkg/cache/cache.go#L136)。Cache 接口包含两个组件：Reader 和 Informers。而 Informers 本身内嵌了 FieldIndexer 接口与。

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

// Informers knows how to create or fetch informers for different
// group-version-kinds, and add indices to those informers.  It's safe to call
// GetInformer from multiple threads.
type Informers interface {
	// Interface methods
	// ...
	
    client.FieldIndexer
}
```

所以，Cache 一共提供方法如下。Reader 接口很简单，提供 Get 和 List 方法。Infomers 是新概念，。

1. Get(ctx context.Context, key ObjectKey, obj Object) error
2. List(ctx context.Context, list ObjectList, opts ...ListOption) error
3. GetInformer(ctx context.Context, obj client.Object) (Informer, error)
4. GetInformerForKind(ctx context.Context, gvk schema.GroupVersionKind) (Informer, error)
5. Start(ctx context.Context) error
6. WaitForCacheSync(ctx context.Context) bool
7. IndexField(ctx context.Context, obj Object, field string, extractValue IndexerFunc) error

Informers 记录的是一组 Informer。每个 Informer 完成两个工作：1. 监听一种 K8s 资源的状态；2. 把状态保存到内存中。前者通过 List and Watch 机制，后者。K8s 资源和 Informer 的缓存通过 map 映射，[以 GVK 作为 map 的 key](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.11.0/pkg/cache/internal/informers_map.go#L97)。

### Informer


### Indexer

### List Watch 机制

## Informer 启动

