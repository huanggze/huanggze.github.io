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

参考官方文档部署 [Install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/) 即可，前期工作与上一篇类似。有两点需要注意：

### 1. 配置 CA 中间证书

[Configure Trust
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

### 2. 通过 Gateway 暴露控制平面

在主从架构中，工作负载实例间可以互相通信，但从集群工作负载代理访问主集群的 Istio 控制平面需要通过 gateway。以下 YAML 来自 expose-istiod.yaml。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway # 描述位于服务网格边缘用于接受进出网格的连接的负载均衡器
metadata:
  name: istiod-gateway
spec:
  # 关联实现负载均衡器网关的 Pods
  selector:
    istio: eastwestgateway
  servers: # 网关上暴露的一组服务端口
    - port:
        name: tls-istiod # 端口别名
        number: 15012 # 网关上暴露的端口号
        protocol: tls # TLS 表示连接会使用 SNI 头部来指示目的地址用于路由
      tls: # TLS 选项
        mode: PASSTHROUGH # 通过 SNI 路由
      # hosts：一个或多个由此网关暴露的、服务网格内的 host。
      # VirtualService 必须绑定一个 Gateway，
      # 且 VirtualService 的 host 必须与 Gateway 的至少一个 host 匹配
      hosts:
        - "*"
    - port:
        name: tls-istiodwebhook
        number: 15017
        protocol: tls
      tls:
        mode: PASSTHROUGH          
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService # 描述路由规则
metadata:
  name: istiod-vs
spec:
  hosts: # 客户端请求服务器时使用的地址名
  - "*"
  gateways: # 关联的 gateway
  - istiod-gateway
  tls: # TLS 和 HTTPS 请求的路由规则
  - match: # 匹配规则
    - port: 15012
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local # 目标服务的真实地址
        port:
          number: 15012
  - match:
    - port: 15017
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 443
```
SNI：服务器名称指示。允许服务器在相同的IP地址和TCP端口号上呈现多个证书，并且因此允许在相同的IP地址上提供多个安全（HTTPS）网站（或其他任何基于TLS的服务），而不需要所有这些站点使用相同的证书[^1]。

[^1]: [什么是 SNI？](https://www.cnblogs.com/pam-sh/p/13409492.html)