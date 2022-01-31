---
title: "Istio 多集群部署（三）：多网络多主架构"
date: 2022-01-29T13:32:02+08:00
toc: true
categories: ["Istio"]
---

## 多网络、多主架构

多网络指多集群间网络隔离，Pod 与 Pod 不互通。因此 API Server 需要暴露公网，且需要分别配置网关，使 Pod 与 Pod 通过网关通信。架构示意图如下，部署教程参考官方文档 [Install Multi-Primary on different networks](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)：

![istio-multicluster-deployment-part3-1](/images/istio-multicluster-deployment-part3-1.svg)

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
$ kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster1/ca-cert.pem \
      --from-file=cluster1/ca-key.pem \
      --from-file=cluster1/root-cert.pem \
      --from-file=cluster1/cert-chain.pem
```

### Step 2. 使用中间证书在两个集群上分别安装 Istio

创建 istio-system 命名空间，并给命名空间打上网络拓扑的标签：

```bash
kubectl label namespace istio-system topology.istio.io/network=network1
kubectl label namespace istio-system topology.istio.io/network=network2
```

安装 Istiod：

```bash
# 在 cluster1 上
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

# 在 cluster2 上
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
EOF
istioctl install -f cluster2.yaml
```

### Step 3. 安装 east-west gateway 

在 cluster 上安装网关来连接东西流量。网关需要公网可访问（或云厂商提供的其他更安全的外网访问方式）。

```bash
# 在 cluster1 上
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster1 --network network1 | \
    istioctl install -y -f -

# 在 cluster2 上    
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster2 --network network2 | \
    istioctl install -y -f -
```
东西流量（East-west traffic）：客户端和服务器之间的流量，即 server-client 流量；
南北流量（North-south traffic）：不同服务器之间的流量、数据中心或不同数据中心之间的网络流，即 server-server 流量。

注意，示例中 gen-eastwest-gateway.sh 脚本创建 gateway 的安装模板是（以 cluster1 为例）一个 IstioOperator YAML 文件。[IstioOperator API](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/) 见官方文档。我们可以看到，这个 YAML 文件安装的 gateway 是一个 Ingress Gateway。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator # 定义 Istio 组件安装的期望状态
metadata:
  name: eastwest
spec:
  revision: "" # 安装版本
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label: # 给 gateway Pod 打标签
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network1
        enabled: true # 是否安装此 gateway
        k8s: # K8s 资源相关字段
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
  # 用于传递给 Helm values.yaml 模板的值。
  # 如果 IstioOperatorSpec 有定义的地方，
  # 优先在 IstioOperatorSpec 中定义。
  values: 
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: network1
```

### Step 4. 暴露网格内的服务

由于服务网格网络隔离，因此两个网络要暴露网络内的服务，才能被其他网络访问。在两个集群上分别执行：

```bash
kubectl apply -n istio-system -f samples/multicluster/expose-services.yaml
```

expose-services.yaml 的内容如下。前面的 IstioOperator YAMl 是创建 gatway Pod 实例，这里的 Gateway 是配置网关规则。本示例中并未用到 egress gateway。

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

### Step 5. 暴露 API Server

需要提供 remote secret 给对方集群，以便对方集群能实现网格内所有服务的服务发现。但由于网络是隔离的，因此需要使用公网访问的 kubeconfig。可以从阿里云「连接信息」页中获得，拷贝到两边集群，命名未 c1.yaml 和 c2.yaml，然后分别执行：

```bash
# 在 cluster1 上
istioctl x create-remote-secret \
  --kubeconfig=c2.yaml | \
  kubectl apply -f -
  
# 在 cluster2 上
istioctl x create-remote-secret \
  --kubeconfig=c1.yaml | \
  kubectl apply -f -
```

## 测试验证

与前面文章同。