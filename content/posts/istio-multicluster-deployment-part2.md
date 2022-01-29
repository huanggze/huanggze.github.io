---
title: "Istio 多集群部署（二）：单一网络主从架构"
date: 2022-01-28T23:24:39+08:00
toc: true
categories: ["Istio"]
---

## 主从架构

主从架构指主集群安装 Istio 控制平面，从集群（remote cluster）连接主集群 Istio 控制平面。在主从架构中，从集群需要通过专门的 gateway 访问主集群上的 Istio 控制平面。**简言之，如果集群内部署有 Istio 控制平面，该集群内的工作负载实例访问集群内的控制平面（主集群模式），否则访问外部 Istio 控制平面（从集群模式）**。下图展示的是单网格、单网络、主从架构部署：

![istio-multicluster-deployment-part2-1](/images/istio-multicluster-deployment-part2-1.svg)

## 部署测试

参考官方文档部署即可。唯一需要注意的是，[Configure Trust
](https://istio.io/latest/docs/setup/install/multicluster/before-you-begin/#configure-trust) 必不可少，需要正确配置。否则会出现证书问题，istio-ingressgateway 无法启动，报错如下：

```bash
$ kubectl get po -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-ingressgateway-b68f578f6-p8j4p   0/1     Running   0          5m58s
istiod-7cd5464766-hr8t4                1/1     Running   0          6m2s


$ kubectl describe po -n istio-system istio-ingressgateway-b68f578f6-p8j4p
Events:
  Type     Reason     Age                      From               Message
  ----     ------     ----                     ----               -------
  Normal   Scheduled  9m32s                    default-scheduler  Successfully assigned istio-system/istio-ingressgateway-b68f578f6-p8j4p to cn-guangzhou.192.168.0.92
  Normal   Pulled     9m32s                    kubelet            Container image "docker.io/istio/proxyv2:1.12.2" already present on machine
  Normal   Created    9m32s                    kubelet            Created container istio-proxy
  Normal   Started    9m32s                    kubelet            Started container istio-proxy
  Warning  Unhealthy  4m30s (x156 over 9m31s)  kubelet            Readiness probe failed: Get "http://10.211.0.4:15021/healthz/ready": dial tcp 10.211.0.4:15021: connect: connection refused
```

```bash
$ kubectl logs -n istio-system istio-ingressgateway-b68f578f6-p8j4p
2022-01-29T11:54:18.257751Z     warning envoy config    StreamAggregatedResources gRPC config stream closed since 672s ago: 14, connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority"
2022-01-29T11:54:46.065125Z     warning envoy config    StreamAggregatedResources gRPC config stream closed since 700s ago: 14, connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority"
2022-01-29T11:55:00.926243Z     warning envoy config    StreamAggregatedResources gRPC config stream closed since 715s ago: 14, connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority"
```

在主从架构中，工作负载实例间可以互相通信，但从集群工作负载代理访问主集群的 Istio 控制平面需要通过 gateway。
