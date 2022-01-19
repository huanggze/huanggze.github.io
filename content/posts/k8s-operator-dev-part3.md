---
title: "K8s Operator 开发（二）：Cache 机制"
date: 2022-01-19T17:45:22+08:00
draft: true
---

## Cache 

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

### Informer


### Indexer

### List Watch 机制

## Informer 启动

