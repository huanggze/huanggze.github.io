---
title: "Istio 多集群部署（四）：多网络主从架构"
date: 2022-01-31T11:16:04+08:00
toc: true
categories: ["Istio"]
---

## 多网络主从架构

多网络、主从架构安装可参考官方文档 [Install Primary-Remote on different networks
](https://istio.io/latest/docs/setup/install/multicluster/primary-remote_multi-network/)。主从架构中，主集群的网关不仅暴露网格内的服务，还有暴露 Istio 控制平面的作用。架构示意图如下：

![istio-multicluster-deployment-part4-1](/images/istio-multicluster-deployment-part4-1.svg)

## 环境准备

本文使用阿里云托管 K8s 服务，在两个**不同** VPC 下，分别部署一个 K8s 集群（命名 cluster1 和 cluster2，本示例部署的是单 worker node 集群），模拟多网络、多主集群。注意，创建集群时，「集群配置 > API Server 访问」一栏中，勾选「使用 EIP 暴露 API Server」。这使的 kube-apiserver 可以公网访问，用于公网连接的 kubeconfig 在集群页「连接信息 > 公网访问」中拿到。

在两个集群上，下载安装 istioctl（1.12.2 版本）：

```bash
curl -O https://ghproxy.com/https://github.com/istio/istio/releases/download/1.12.2/istioctl-1.12.2-linux-amd64.tar.gz
tar zxvf istioctl-1.12.2-linux-amd64.tar.gz
mv istioctl /usr/local/bin/
```

最后，为了启用 kubectl 命令，拷贝一份内网访问 kubeconfig（通过阿里云控制台，容器服务 > 集群列表 > 集群信息 > 连接信息，获取）到各自集群 worker node 机器上的 .kube/ 目录下，文件命名为 config。

## 部署 Istio 控制平面

### Step 1. 证书配置

配置根证书、中间证书的方式和第一篇文章一样。只不过 cluster2 的中间证书需要手动拷贝到 cluster2 集群。这是因为两个集群不在一个内网、网络隔离，无法像第一篇一样通过 scp 和内网地址传输 cluster2 的中间证书。

安装 cacerts 密钥（以 cluster1 为例）：

```bash
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster1/ca-cert.pem \
      --from-file=cluster1/ca-key.pem \
      --from-file=cluster1/root-cert.pem \
      --from-file=cluster1/cert-chain.pem
```

### Step 2. 使用中间证书在 cluster1 上安装 Istiod

给 istio-system 命名空间打上网络拓扑标签：

```bash
# 在 cluster1 上
kubectl label namespace istio-system topology.istio.io/network=network1
# 在 cluster2 上
kubectl label namespace istio-system topology.istio.io/network=network2
```

在 cluster1 主集群上安装 Istiod：

```bash
cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
istioctl install -f cluster1.yaml
```

### Step 3. 在 cluster1 上安装 east-west gateway

安装 east-west gateway，并记下 gateway 的 IP 地址：

```bash
cd istio
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster1 --network network1 | \
    istioctl install -y -f -
    
# 记下 gateway 的公网 IP 地址
kubectl get svc -n istio-system  istio-eastwestgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

gen-eastwest-gateway.sh 脚本生成的 gateway 安装配置文件如下：
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest
spec:
  revision: ""
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network1
        enabled: true
        k8s:
          env:
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: network1
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: network1
```

### Step 4. 通过 cluster1 的 easy-west gateway 暴露 Istio 控制平面和集群内服务

可以看到，前一步安装的 Istio ingress gateway 在这一步被两个 Gateway YAML 配置文件使用。一个 Gateway YAML 负责暴露 Istiod，一个负责暴露集群内的服务。

```bash
kubectl apply -n istio-system -f samples/multicluster/expose-istiod.yaml
kubectl apply -n istio-system -f samples/multicluster/expose-services.yaml
```

expose-istiod.yaml 的内容如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istiod-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        name: tls-istiod
        number: 15012
        protocol: tls
      tls:
        mode: PASSTHROUGH        
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
kind: VirtualService
metadata:
  name: istiod-vs
spec:
  hosts:
  - "*"
  gateways:
  - istiod-gateway
  tls:
  - match:
    - port: 15012
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
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

expose-services.yaml 的内容如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-network-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
```

### Step 5. 暴露 cluster2 的 API Server 给 cluster1

需要提供 cluster2 集群 kubeconfig 作为 remote secret 给集群 cluster1，以便 cluster1 能实现网格内所有服务的服务发现。但由于网络是隔离的，因此需要使用 cluster2 的公网访问 kubeconfig。可以从阿里云「连接信息」页中获得，拷贝到 cluster1，命名为 c2.yaml，然后执行（注意 --name 标志一定要带上）：

```bash
istioctl x create-remote-secret \
  --kubeconfig=c2.yaml \
  --name=cluster2 | \
  kubectl apply -f -
```

### Step 6. cluster2 作为从集群安装

```bash
cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
      remotePilotAddress: ${DISCOVERY_ADDRESS} # 这里填刚才记下的 cluster1 east-west gateway 地址
EOF
istioctl install -f cluster2.yaml
```

### Step 7. 在 cluster2 上安装 east-west gateway，并暴露集群内的服务

```bash
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster2 --network network2 | \
    istioctl install -y -f -

kubectl apply -n istio-system -f \
    samples/multicluster/expose-services.yaml
```

expose-services.yaml 的内容和 cluster1 的步骤一样：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-network-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
```

## 测试验证

同前文。

## 总结

在实践四种方式安装多集群时，踩来不少坑，总结出以下几点需要注意：

1. 一定要按官方文档的操作顺序来。否则 Istio 相关的 Pod 会起不来。比如中间重新生成了密钥，这时候得从头安装 Istio；
2. `kubectl create secret` 如果 from-file 的文件有问题，创建 secret 的时候不会报错的。所以要格外小心。

|部署方案|同一CA签发证书|网络拓扑标签|安装east-west gateway|暴露 Istiod|暴露网格服务|暴露 API Server|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|单网络多主|✅|-|-|-|-|✅|
|单网络主从|✅|-|主集群✅|主集群✅|-|从集群✅|
|多网络多主|✅|✅|✅|-|✅|✅|
|多网络主从|✅|✅|✅|主集群✅|✅|从集群✅|

✅：表示该步骤与所有集群都相关