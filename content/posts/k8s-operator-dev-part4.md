---
title: "K8s Operator 开发（四）：List-Watch 机制"
date: 2022-01-19T17:46:40+08:00
draft: true
---

## Watch API

前文提到 Indexer 实现缓存同步是通过 [ListAndWatch](https://github.com/kubernetes/client-go/blob/v0.22.1/tools/cache/reflector.go#L254) 机制返回。我们在第二篇 K8s API 中已经了解到 List API 和 Watch API 的请求 URI。比如，请求某个命名空间下所有 Deployment 会返回 DeploymentList 对象：

```yaml
# kubectl proxy --port 8080
# curl 127.0.0.1:8080/apis/apps/v1/namespaces/default/deployments
{
    "kind": "DeploymentList",
    "apiVersion": "apps/v1",
    "metadata": {
        "resourceVersion": "1348795"
    },
    "items": [
        {
            "metadata": { 
                "name": "nginx",
                ...
            },
            "spec": {
                ...
            },
            "status": {
                ...        
            },
       },
    ]
}
```

List 是基于 HTTP 短链接，而 Watch 要复杂一些。这里我们首先重点讨论。Watch 请求使用的是 HTTP 中的 GET 动作。这是一个长连接的流式 HTTP（使用 HTTP 2.0 协议）。以下示例输出来自 Watch 默认命名空间下所有 Deployment 的变更。Watch 请求返回的是一个个 [WatchEvent](https://github.com/kubernetes/apimachinery/blob/v0.22.1/pkg/watch/watch.go#L57) 对象。WatchEvent 数据由两个字段 type 和 object，type 表示事件类型（Added/Modified/Deleted），object 是当前监听返回的对象。可以看到发起 Watch 请求时，第一个返回是 ADDED，得到已经存在的 nginx Deployment；然后我们手动创建第二个 Deployment nginx-2，可以看到再收到一个 ADDED 事件以及一系列 Modified 事件。

```bash
$ kubectl proxy --port 8080

$ curl -i "127.0.0.1:8080/apis/apps/v1/namespaces/default/deployments?watch=true"
HTTP/1.1 200 OK
Cache-Control: no-cache, private
Content-Type: application/json
Date: Sun, 23 Jan 2022 14:54:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED","object":{"kind":"Deployment","apiVersion":"apps/v1","metadata":{"name":"nginx","namespace":"default","uid":"786d3456-db91-47b8-9829-fcce567b11f6","resourceVersion":"1348784","generation":1,"creationTimestamp":"2022-01-23T14:11:53Z","labels":{"app":"nginx"},"annotations":{"deployment.kubernetes.io/revision":"1"},"managedFields":[{"manager":"kubectl-create","operation":"Update","apiVersion":"apps/v1","time":"2022-01-23T14:11:53Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:progressDeadlineSeconds":{},"f:replicas":{},"f:revisionHistoryLimit":{},"f:selector":{},"f:strategy":{"f:rollingUpdate":{".":{},"f:maxSurge":{},"f:maxUnavailable":{}},"f:type":{}},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"nginx\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}}},{"manager":"kube-controller-manager","operation":"Update","apiVersion":"apps/v1","time":"2022-01-23T14:12:04Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:deployment.kubernetes.io/revision":{}}},"f:status":{"f:availableReplicas":{},"f:conditions":{".":{},"k:{\"type\":\"Available\"}":{".":{},"f:lastTransitionTime":{},"f:lastUpdateTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Progressing\"}":{".":{},"f:lastTransitionTime":{},"f:lastUpdateTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}},"f:observedGeneration":{},"f:readyReplicas":{},"f:replicas":{},"f:updatedReplicas":{}}}}]},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"nginx"}},"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","securityContext":{},"schedulerName":"default-scheduler"}},"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":"25%","maxSurge":"25%"}},"revisionHistoryLimit":10,"progressDeadlineSeconds":600},"status":{"observedGeneration":1,"replicas":1,"updatedReplicas":1,"readyReplicas":1,"availableReplicas":1,"conditions":[{"type":"Available","status":"True","lastUpdateTime":"2022-01-23T14:12:04Z","lastTransitionTime":"2022-01-23T14:12:04Z","reason":"MinimumReplicasAvailable","message":"Deployment has minimum availability."},{"type":"Progressing","status":"True","lastUpdateTime":"2022-01-23T14:12:04Z","lastTransitionTime":"2022-01-23T14:11:53Z","reason":"NewReplicaSetAvailable","message":"ReplicaSet \"nginx-6799fc88d8\" has successfully progressed."}]}}}
{"type":"ADDED","object":{"kind":"Deployment","apiVersion":"apps/v1","metadata":{"name":"nginx-v2","namespace":"default","uid":"7f925568-de95-441c-99fe-fb91812a6ad0","resourceVersion":"1350344","generation":1,"creationTimestamp":"2022-01-23T14:26:23Z","labels":{"app":"nginx-v2"},"managedFields":[{"manager":"kubectl-create","operation":"Update","apiVersion":"apps/v1","time":"2022-01-23T14:26:23Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:progressDeadlineSeconds":{},"f:replicas":{},"f:revisionHistoryLimit":{},"f:selector":{},"f:strategy":{"f:rollingUpdate":{".":{},"f:maxSurge":{},"f:maxUnavailable":{}},"f:type":{}},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"nginx\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}}}]},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"nginx-v2"}},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"nginx-v2"}},"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","securityContext":{},"schedulerName":"default-scheduler"}},"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":"25%","maxSurge":"25%"}},"revisionHistoryLimit":10,"progressDeadlineSeconds":600},"status":{}}}
{"type":"MODIFIED","object":{"kind":"Deployment","apiVersion":"apps/v1","metadata":{"name":"nginx-v2","namespace":"default","uid":"7f925568-de95-441c-99fe-fb91812a6ad0","resourceVersion":"1350346","generation":1,"creationTimestamp":"2022-01-23T14:26:23Z","labels":{"app":"nginx-v2"},"annotations":{"deployment.kubernetes.io/revision":"1"},"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"apps/v1","time":"2022-01-23T14:26:23Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:deployment.kubernetes.io/revision":{}}},"f:status":{"f:conditions":{".":{},"k:{\"type\":\"Progressing\"}":{".":{},"f:lastTransitionTime":{},"f:lastUpdateTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}}}}},{"manager":"kubectl-create","operation":"Update","apiVersion":"apps/v1","time":"2022-01-23T14:26:23Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:progressDeadlineSeconds":{},"f:replicas":{},"f:revisionHistoryLimit":{},"f:selector":{},"f:strategy":{"f:rollingUpdate":{".":{},"f:maxSurge":{},"f:maxUnavailable":{}},"f:type":{}},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"nginx\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}}}]},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"nginx-v2"}},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"nginx-v2"}},"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","securityContext":{},"schedulerName":"default-scheduler"}},"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":"25%","maxSurge":"25%"}},"revisionHistoryLimit":10,"progressDeadlineSeconds":600},"status":{"conditions":[{"type":"Progressing","status":"True","lastUpdateTime":"2022-01-23T14:26:23Z","lastTransitionTime":"2022-01-23T14:26:23Z","reason":"NewReplicaSetCreated","message":"Created new replica set \"nginx-v2-798f8fdcf7\""}]}}}
```

> WatchEvent 和 K8s Event 的区别？\
> 两个不是同一概念。WatchEvent 是 Watch API 返回的数据类型；K8s Event 是一个 K8s 资源，kubectl describe pod 时打印的 Event 信息就来自 K8s Event。K8s Event 也可以通过 kubectl get event 获取。

## List-Watch 设计理念

K8s 作为优秀的分布式系统设计时，充分考虑到了异步消息系统的以下要求[^1]：

1. 消息可靠性：List 是请求全量，Watch 是请求增量。通过定时重新 List 纠正状态不一致。
2. 消息实时性：kube-apiserver 在资源产生状态变更事件时会及时推送给客户端
3. 消息顺序性：K8s 资源有 resourceVersion 记录资源的版本（存储在 etcd 的版本），resourceVersion 是递增的。可以根据 resourceVersion 保序。

## Reflector 与 DeltaFIFO 队列

[^1]: [理解 K8S 的设计精髓之 List-Watch机制和Informer模块](https://zhuanlan.zhihu.com/p/59660536)