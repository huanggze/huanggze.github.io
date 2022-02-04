---
title: "Istio 支持虚拟机集成实践"
date: 2022-01-29T14:52:28+08:00
draft: true
---

Istio 支持 K8s 集群外的虚拟机及虚拟机运行的应用加入 Istio 服务网格。这允许老的应用以及不适合容器化部署的应用也能使用 Istio 服务网格。 在单网络下，Gateway 负责虚拟机访问允许在 K8s 上的 Istio 控制平面：

![istio-virtual-machines-1](/images/istio-virtual-machines-1.svg)

多网络下，Gateway 负责同时暴露 Istio 控制平面和服务网格内的服务：

![istio-virtual-machines-2](/images/istio-virtual-machines-2.svg)

## 虚拟机集成

### Step 1. 环境准备

本文使用阿里云，在同一 VPC 下，部署一个 K8s 托管集群以及一个 ECS 实例，模拟单网络下集成虚拟机。在 K8s 节点机器上设置以下环境变量用于后续操作，并创建工作目录 `mkdir -p "${WORK_DIR}"`。

```bash
VM_APP="demo"
VM_NAMESPACE="vm"
WORK_DIR="${HOME}/vmintegration"
SERVICE_ACCOUNT="vm"
# 多网络下，K8s 集群和虚拟机所在网络分别命名，如：
# CLUSTER_NETWORK="kube-network"
# VM_NETWORK="vm-network"
# 而 CLUSTER 值赋为 cluster1
CLUSTER_NETWORK=""
VM_NETWORK=""
CLUSTER="Kubernetes"
```

### Step 2. 安装 Istio 控制平面

在 K8s 集群上安装 Istio（如果集群已安装则无须安装，但仍要暴露 Istio 控制平面）：

```yaml
cat <<EOF > ./vm-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "${CLUSTER}"
      network: "${CLUSTER_NETWORK}"
EOF

istioctl install -f vm-cluster.yaml
```

### Step 3. 安装 east-west gateway

通过 IstioOperator YAML 安装 east-west gateway：

```bash
samples/multicluster/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
```

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
        enabled: true
        k8s:
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
```

### Step 4. 暴露 Istio 控制平面

```bash
kubectl apply -n istio-system -f samples/multicluster/expose-istiod.yaml
# 多网络部署多以下步骤
# kubectl apply -n istio-system -f samples/multicluster/expose-services.yaml
```

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

### Step 5. 创建虚拟机的命名空间

```bash
kubectl create namespace "${VM_NAMESPACE}"
kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```

### Step 6. 生成虚拟机应用的配置文件

通过 `istioctl x workload entry` 命令生成虚拟机应用（workload instance）的配置文件。该命令会生成五个文件：

1. cluster.env：包含一些元信息，如：流量入口的端口号、虚拟机对应的 serviceaccount 和 namespace；
2. istio-token：JWT token，用于从 CA 获取证书；
3. mesh.yaml
4. root-cert.pem：用于认证的根证书；
5. hosts：包含 Istiod 的地址。

```bash
cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF

istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
```

cluster.env 的内容如下：

```bash
CANONICAL_REVISION='latest'
CANONICAL_SERVICE='demo'
ISTIO_INBOUND_PORTS='*'
ISTIO_LOCAL_EXCLUDE_PORTS='15090,15021,15020'
ISTIO_METAJSON_LABELS='{"app":"demo","service.istio.io/canonical-name":"demo","service.istio.io/canonical-revision":"latest"}'
ISTIO_META_CLUSTER_ID='Kubernetes'
ISTIO_META_DNS_CAPTURE='true'
ISTIO_META_MESH_ID='mesh1'
ISTIO_META_NETWORK=''
ISTIO_META_WORKLOAD_NAME='demo'
ISTIO_NAMESPACE='vm'
ISTIO_SERVICE='demo.vm'
ISTIO_SERVICE_CIDR='*'
POD_NAMESPACE='vm'
SERVICE_ACCOUNT='vm'
TRUST_DOMAIN='cluster.local'
```

mesh.yaml 的内容如下：

```yaml
defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
  meshId: mesh1
  proxyMetadata:
    CANONICAL_REVISION: latest
    CANONICAL_SERVICE: demo
    ISTIO_META_CLUSTER_ID: Kubernetes
    ISTIO_META_DNS_CAPTURE: "true"
    ISTIO_META_MESH_ID: mesh1
    ISTIO_META_NETWORK: ""
    ISTIO_META_WORKLOAD_NAME: demo
    ISTIO_METAJSON_LABELS: '{"app":"demo","service.istio.io/canonical-name":"demo","service.istio.io/canonical-revision":"latest"}'
    POD_NAMESPACE: vm
    SERVICE_ACCOUNT: vm
    TRUST_DOMAIN: cluster.local
  tracing:
    zipkin:
      address: zipkin.istio-system:9411
```

hosts 包含 Istiod 的地址：

```text
8.134.<xxx>.<xxx> istiod.istio-system.svc
```


### Step 6. 配置虚拟机

将前面生成的配置，从 K8s 集群节点机器上，传输到要加入服务网格的虚拟机上。然后依次将配置文件安装到虚拟机上：

```bash
# 首先，将根证书安装到 /etc/certs 目录下
mkdir -p /etc/certs
cp root-cert.pem /etc/certs/root-cert.pem

# 接着，将 istio-token 安装到 /var/run/secrets/tokens 目录下
mkdir -p /var/run/secrets/tokens
cp istio-token /var/run/secrets/tokens/istio-token

# 然后，将 cluster.env 安装到 /var/lib/istio/envoy/ 目录下
cp cluster.env /var/lib/istio/envoy/cluster.env

# 将 mesh 配置文件安装到 /etc/istio/config/ 目录下，并重命名为 mesh
cp mesh.yaml /etc/istio/config/mesh

# 最后，更新 hosts 文件
cat hosts >> /etc/hosts
```

并将 /etc/certs/ 等目录的所有者设置为 istio-proxy：

```bash
mkdir -p /etc/istio/proxy
chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
```

### Step 7. 安装 Istio 虚拟机集成运行时：

因阿里云 ECS 的操作系统 Alibaba Cloud Linux 是基于 CentOS 的，因此选择 CentOS 8 的安装包：

```bash
curl -LO https://storage.googleapis.com/istio-release/releases/1.12.2/rpm/istio-sidecar.rpm
rpm -i istio-sidecar.rpm
```

运行 Istio agent：

```bash
systemctl start istio
```

检查是否安装成功 `cat /var/log/istio/istio.log`，日志中无 failed 字样。至此，可以访问 K8s 服务网格中的 Service，如：`curl helloworld.sample.svc:5000/hello`。