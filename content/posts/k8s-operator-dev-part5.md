---
title: "K8s Operator 开发（四）：Manager 启动"
date: 2022-03-21T21:56:27+08:00
draft: true
---

runnable
/Users/game-netease/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.11.0/pkg/manager/runnable_group.go
```go
type runnables struct {
	Webhooks       *runnableGroup
	Caches         *runnableGroup
	LeaderElection *runnableGroup
	Others         *runnableGroup
}
```


setupwith 会 add 第一个 runnable--default leaderelection
/Users/game-netease/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.11.0/pkg/controller/controller.go
```go
// New returns a new Controller registered with the Manager.  The Manager will ensure that shared Caches have
// been synced before the Controller is Started.
func New(name string, mgr manager.Manager, options Options) (Controller, error) {
	c, err := NewUnmanaged(name, mgr, options)
	if err != nil {
		return nil, err
	}

	// Add the controller as a Manager components
	return c, mgr.Add(c)
}
```

/Users/game-netease/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.11.0/pkg/manager/internal.go
callback
```go
Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(_ context.Context) {
        if err := cm.startLeaderElectionRunnables(); err != nil {
            cm.errChan <- err
            return
        }
        close(cm.elected)
    },
    OnStoppedLeading: func() {
        if cm.onStoppedLeading != nil {
            cm.onStoppedLeading()
        }
        // Make sure graceful shutdown is skipped if we lost the leader lock without
        // intending to.
        cm.gracefulShutdownTimeout = time.Duration(0)
        // Most implementations of leader election log.Fatal() here.
        // Since Start is wrapped in log.Fatal when called, we can just return
        // an error here which will cause the program to exit.
        cm.errChan <- errors.New("leader election lost")
    },
```

---
succeeded = le.tryAcquireOrRenew(ctx)
是否成功


跑起一个实现了 leader-elec 的 controller runnable
```go
func (cm *controllerManager) startLeaderElectionRunnables() error {
	return cm.runnables.LeaderElection.Start(cm.internalCtx)
}
```


```bash
[root@iZ7xvg39u5exti9uz9gfrkZ ~]# k logs -n gas gas-operator-657899c8b8-rmw2l -p
2022-03-22T09:18:25.392116Z	info	klog	Waited for 1.037743826s due to client-side throttling, not priority and fairness, request: GET:https://172.16.0.1:443/apis/storage.alibabacloud.com/v1alpha1?timeout=32s
1.6479407057939525e+09	INFO	controller-runtime.metrics	Metrics server is starting to listen	{"addr": "127.0.0.1:8080"}
1.6479407057942128e+09	INFO	controller-runtime.builder	Registering a mutating webhook	{"GVK": "networking.my.domain/v1alpha1, Kind=ExternalService", "path": "/mutate-networking-my-domain-v1alpha1-externalservice"}
1.647940705794281e+09	INFO	controller-runtime.webhook	Registering webhook	{"path": "/mutate-networking-my-domain-v1alpha1-externalservice"}
1.6479407057943192e+09	INFO	controller-runtime.builder	Registering a validating webhook	{"GVK": "networking.my.domain/v1alpha1, Kind=ExternalService", "path": "/validate-networking-my-domain-v1alpha1-externalservice"}
1.6479407057943592e+09	INFO	controller-runtime.webhook	Registering webhook	{"path": "/validate-networking-my-domain-v1alpha1-externalservice"}
1.64794070579442e+09	INFO	setup	starting manager
1.6479407057946649e+09	INFO	Starting server	{"path": "/metrics", "kind": "metrics", "addr": "127.0.0.1:8080"}
1.647940705794696e+09	INFO	Starting server	{"kind": "health probe", "addr": "[::]:8081"}
1.6479407057946427e+09	INFO	controller-runtime.webhook.webhooks	Starting webhook server
1.6479407057949371e+09	INFO	controller-runtime.certwatcher	Updated current TLS certificate
1.6479407057950022e+09	INFO	controller-runtime.webhook	Serving webhook server	{"host": "", "port": 9443}
1.6479407057950592e+09	INFO	controller-runtime.certwatcher	Starting certificate watcher
2022-03-22T09:18:25.796566Z	info	klog	attempting to acquire leader lease gas/541aa8d9.my.domain...
2022-03-22T09:18:25.808057Z	info	klog	successfully acquired lease gas/541aa8d9.my.domain
1.6479407058081634e+09	DEBUG	events	Normal	{"object": {"kind":"ConfigMap","namespace":"gas","name":"541aa8d9.my.domain","uid":"5376e6d2-c3ff-4c2a-9985-681afec36ebf","apiVersion":"v1","resourceVersion":"50436267"}, "reason": "LeaderElection", "message": "gas-operator-657899c8b8-rmw2l_4f09c667-d125-4813-8c53-22710ee9b6ba became leader"}
1.6479407058082526e+09	DEBUG	events	Normal	{"object": {"kind":"Lease","namespace":"gas","name":"541aa8d9.my.domain","uid":"961a81ac-c0a6-4fcc-9031-b26d81d1fea6","apiVersion":"coordination.k8s.io/v1","resourceVersion":"50436268"}, "reason": "LeaderElection", "message": "gas-operator-657899c8b8-rmw2l_4f09c667-d125-4813-8c53-22710ee9b6ba became leader"}
1.6479407058082745e+09	INFO	controller.rebootpolicy	Starting EventSource	{"reconciler group": "autoreboot.my.domain", "reconciler kind": "RebootPolicy", "source": "kind source: *v1alpha1.RebootPolicy"}
1.6479407058083148e+09	INFO	controller.rebootpolicy	Starting Controller	{"reconciler group": "autoreboot.my.domain", "reconciler kind": "RebootPolicy"}
1.647940705808863e+09	INFO	controller.externalservice	Starting EventSource	{"reconciler group": "networking.my.domain", "reconciler kind": "ExternalService", "source": "kind source: *v1alpha1.ExternalService"}
1.6479407058089213e+09	INFO	controller.externalservice	Starting Controller	{"reconciler group": "networking.my.domain", "reconciler kind": "ExternalService"}
1.6479407059090834e+09	INFO	controller.rebootpolicy	Starting workers	{"reconciler group": "autoreboot.my.domain", "reconciler kind": "RebootPolicy", "worker count": 1}
1.6479407059101803e+09	INFO	controller.externalservice	Starting workers	{"reconciler group": "networking.my.domain", "reconciler kind": "ExternalService", "worker count": 1}
2022-03-22T09:27:14.178024Z	info	klog	failed to renew lease gas/541aa8d9.my.domain: timed out waiting for the condition
1.6479412341780827e+09	ERROR	setup	problem running manager	{"error": "leader election lost"}
main.main
	/workspace/main.go:121
runtime.main
	/usr/local/go/src/runtime/proc.go:255
```