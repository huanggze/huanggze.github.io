---
title: "K8s Operator 开发（五）：Leader Election 机制"
date: 2022-01-19T17:47:13+08:00
draft: true
---

那如果我的元件執行在 Kubernetes 叢集裡，有沒有什麼方式可以不依靠第三方系統讓我們也能做到分散式資源鎖呢？

答案是有的，可以透過Kubernetes的ResourceLock來達成，ResourceLock 大致上可以分為endpoints,configmaps,leases三種。

关于分布式锁的实现很多，可以自己从零开始制造。当然更简单的是基于现有中间件，比如有基于 Redis 或数据库的实现方式，最近 Zookeeper/ETCD 也提供了相关功能。但 K8s 的实现并没有使用这些方式，而是另辟蹊径使用了资源锁的概念，简单来说就是通过创建 K8s 的资源（当前的实现中实现了 ConfigMap 和 Endpoint 两种类型的资源）来维护锁的状态。


返回 resourcelock.Interface
```go
return resourcelock.New(options.LeaderElectionResourceLock,
		options.LeaderElectionNamespace,
		options.LeaderElectionID,
		corev1Client,
		coordinationClient,
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: recorderProvider.GetEventRecorderFor(id),
		})
```
/Users/game-netease/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.11.0/pkg/leaderelection/leader_election.go

/Users/game-netease/go/pkg/mod/k8s.io/client-go@v0.23.1/tools/leaderelection/resourcelock/interface.go

https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/
https://blog.jjmengze.website/posts/kubernetes/resourcelock/



