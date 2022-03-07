---
title: "Kubernetes 编程风格"
date: 2022-01-22T16:25:12+08:00
draft: true
---

# 类型检查

sigs.k8s.io/controller-runtime@v0.10.0/pkg/source/source.go

```go
var _ inject.Cache = &Kind{}

// InjectCache is internal should be called only by the Controller.  InjectCache is used to inject
// the Cache dependency initialized by the ControllerManager.
func (ks *Kind) InjectCache(c cache.Cache) error {
	if ks.cache == nil {
		ks.cache = c
	}
	return nil
}

```
