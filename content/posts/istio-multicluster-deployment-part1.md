---
title: "Istio 多集群部署（一）：单一网络多主架构"
date: 2022-01-28T15:28:19+08:00
toc: true
categories: ["Istio"]
---

## 单网格、单网络、多主架构

单网格、单网络、多主架构部署对应官方文档 [Install Multi-Primary](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/)。单网格、单网络、多主架构部署指单个 Istio 服务网格（service mesh）运行在单个完全互联的网络上。网络内有多个集群，同时存在多个主集群（primary cluster）运行 Istio 控制平面。示例架构如下图：

![istio-multicluster-deployment-part1-1](/images/istio-multicluster-deployment-part1-1.svg)

单一网络模型，即所有工作负载实例（workload instances，指 pods）都可以直接相互访问、完全互联，而无需 Istio 网关。

> 注意：这里「可以直接相互访问」指的是 Pod 与 Pod 间互通（可互 ping），包括跨集群的 Pod 通信。不是指 Service 之间 Cluster IP 互相可 ping，Service 的 ClusterIP 不支持跨集群访问。ClusterIP 是虚拟 IP，没有对应实体，而跨集群 Pod IP 能互 ping 是因为路由表中存在对应网段的下一跳节点。

多主架构指多个集群下，存在多个单独部署的 Istio 控制平面。我们知道，Istio 控制平面通过向工作负载实例的 Envoy 代理下发服务端点信息实现流量管理。因此**单网格**下，Istio 控制平面需要拿到所有集群的服务端点信息。服务端点发现需要配置 Istio 控制平面使其能访问每个集群的 kube-apiserver[^1]。

## 环境准备

本文使用阿里云托管 K8s 服务，在同一 VPC 下，部署两个集群（命名 cluster1 和 cluster2，本示例部署的是单 worker node 集群），模拟单网络、多集群。注意，在创建托管 K8s 界面里应设置 Pod CIDR 为不同网段，如 10.210.0.0/16 和 10.211.0.0/16。创建完后，检查跨集群 Pod 是否可以互相通信（互 ping Pod IP）。同一 VPC 下部署的集群 Pod 互通是因为 VPC 路由表存在对应的网段下一跳节点（通过阿里云控制台「专有网络 > 路由表」查看）。

在两个集群上，下载安装 istioctl（1.12.2 版本）。由于 Istio 官网的下载脚本拉的是海外包，会超时，改用 ghproxy.com 代理下载。

```bash
curl -O https://ghproxy.com/https://github.com/istio/istio/releases/download/1.13.2/istioctl-1.13.2-linux-amd64.tar.gz
tar zxvf istioctl-1.13.2-linux-amd64.tar.gz
mv istioctl /usr/local/bin/
```

最后，为了启用 kubectl 命令，拷贝一份 kubeconfig（通过阿里云控制台，容器服务 > 集群列表 > 集群信息 > 连接信息，获取）到各自集群 worker node 机器上的 .kube/ 目录下，文件命名为 config。

## 部署 Istio 控制平面

分别在 cluster1 和 cluster2 上安装部署 Istio 控制平面，实现前面图中的架构。实践中发现，在阿里云环境部署与官方文档相比，有少许额外工作需要做，请特别注意。因此，我们不能完全按照官方文档来，要理解原理，底层逻辑都是一样的。

### Step 1. 证书配置

单网格、多集群架构下，主集群之间要相互建立信任。[官方文档](https://istio.io/latest/docs/setup/install/multicluster/before-you-begin/#configure-trust)提供了用于快速测试的证书配置方法：配置一个自签名的根证书，然后通过根证书为两个集群单独生成中间证书，如下图。Istio 使用中间证书安装。如果没有单独配置根证书，Istio 会默认生成自签名的 CA 证书用于签发工作负载实例的证书，导致网格隔离、不同控制平面签发的证书无法互认。另外注意，在已有 Istio 集群上测试会因为需要配置新证书而重新安装 Istio。

![istio-multicluster-deployment-part1-2](/images/istio-multicluster-deployment-part1-2.svg)

首先，在 cluster1 上 clone istio 代码仓库，里面包含创建证书相关的脚本：

```bash
yum install git -y

git clone https://ghproxy.com/https://github.com/istio/istio.git
cd istio
git checkout tags/1.12.2 -b 1.12.2
```

在 cluster1 机器上，创建根证书。以下命令会生成 root-cert.pem 等四个文件。

```bash
mkdir -p certs
pushd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca
```

为 cluster1、cluster2 生成中间证书和密钥：

```bash
make -f ../tools/certs/Makefile.selfsigned.mk cluster1-cacerts
make -f ../tools/certs/Makefile.selfsigned.mk cluster2-cacerts
```

将 cluster2 的证书发送到 cluster2 机器上（IP 地址是 192.168.0.92）：

```bash
# 在 cluster1 机器上
scp cluster2/* 192.168.0.92:~

# 在 cluster2 机器上
mkdir cluster2
mv *.pem cluster2
```

### Step 2. 使用中间证书在两个集群上分别安装 Istio

在 cluster1 机器上安装 Istio。IstioOperator YAML 是安装配置文件。

```bash
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster1/ca-cert.pem \
      --from-file=cluster1/ca-key.pem \
      --from-file=cluster1/root-cert.pem \
      --from-file=cluster1/cert-chain.pem


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

类似地，在 cluster2 上执行：

```bash
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster2/ca-cert.pem \
      --from-file=cluster2/ca-key.pem \
      --from-file=cluster2/root-cert.pem \
      --from-file=cluster2/cert-chain.pem


cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network1
EOF
istioctl install -f cluster2.yaml
```

### Step 3. 设置服务发现

为了能让两个主集群互相了解对方集群上部署的服务以及后端实例，须使 Istio 控制平面能互相访问对方 kube-apiserver。首先，把各自 kubeconfig 文件发给对方。

```bash
# 在 cluster1 机器上，把自己的 kubeconfig 发给 cluster2 机器（IP 地址 192.168.0.92）
scp .kube/config 192.168.0.92:~/.kube/cluster1.yaml

# 在 cluster2 机器上，把自己的 kubeconfig 发给 cluster1 机器（IP 地址 192.168.0.87）
scp .kube/config 192.168.0.87:~/.kube/cluster2.yaml
```

在各自集群机器上分别执行创建 remote secret（这里，我没有用到官方文档提到的环境变量 CTX_CLUSTER1，因为是直接在各自集群分开操作）：

```bash
# 在 cluster2 上配置访问 cluster1
istioctl x create-remote-secret \
    --kubeconfig=.kube/cluster1.yaml | \
    kubectl apply -f -

# 在 cluster1 上配置访问 cluster2
istioctl x create-remote-secret \
    --kubeconfig=.kube/cluster2.yaml | \
    kubectl apply -f -
```

> `istioctl x create-remote-secret` 创建的 remote secret 在哪里被使用？\
> secret 并非通过 volumemount 的形式挂载到 istiod Pod 中使用，而是 istiod 有 secret API 的权限，详见 istiod 的 Role 和 Rolebinding。只有安装了 remote secret，`istioctl remote-clusters` 才会正确打印出远端集群。

至此，单网格、单网络、多主架构已部署好。

## 测试验证

可参考[官方文档](https://istio.io/latest/docs/setup/install/multicluster/verify/)测试连通性。同样，如果手动分别在不同集群上直接操作，不需要 `--context="${CTX_CLUSTER1}"`。

```bash
kubectl create namespace sample
kubectl label namespace sample istio-injection=enabled
kubectl apply -f samples/helloworld/helloworld.yaml -l service=helloworld -n sample

# 在 cluster1 上
kubectl apply -f samples/helloworld/helloworld.yaml -l version=v1 -n sample
# 在 cluster2 上
kubectl apply -f samples/helloworld/helloworld.yaml -l version=v2 -n sample

kubectl apply -f samples/sleep/sleep.yaml -n sample
kubectl exec -n sample -c sleep \
    "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
```

[^1]: [Endpoint discovery with multiple control planes](https://istio.io/latest/docs/ops/deployment/deployment-models/#endpoint-discovery-with-multiple-control-planes)
