---
title: "Istio 客户端源 IP 保持"
date: 2022-12-14T21:56:37+08:00
toc: true
categories: ["istio"]
---

## 问题背景

发现默认部署的 Istio 服务网格下，服务端收到请求 IP 源地址是 127.0.0.6，而不是请求方的真实 IP 地址。

## 环境复现

环境复现总共需要准备的内容清单：

- 服务端 Pod：打印客户端 IP 地址；
- 客户端 Pod：向服务端发送请求；
- K8s 环境：使用阿里云 ACK 部署 K8s 环境；
- Istio：安装 Istio。

各部署相关的注意事项见下面分小节的内容。

### 1. 服务端 demo

在 18080 端口接收客户端请求，解析IP地址的方式设置为两种：读取 XFF 首部，或 request 的 RemoteAddr 字段。

```go
package main

import (
	"fmt"
	"net/http"
)

func handle(w http.ResponseWriter, r *http.Request) {
	// 打印客户端IP
	fmt.Printf("X-Forwarded-For=%s, RemoteAddr=%s\n", r.Header.Get("X-Forwarded-For"), r.RemoteAddr)

	w.Write([]byte(`{"code": 0, "status": "fail", "user": {"id": "test_common"}}`))
}

func main() {
	http.HandleFunc("/common/sauth", handle)

	err := http.ListenAndServe(":18080", nil)
	if err != nil {
		fmt.Println(err)
	}
}
```

镜像构建 Dockerfile 和命令：

```shell
docker buildx build --platform=linux/amd64 -t vasth/simple-server .
```

```dockerfile
FROM golang:alpine
RUN mkdir /app
COPY . /app
WORKDIR /app
RUN go build -o main .
CMD ["/app/main"]
```

K8s 部署 YAML：

```yaml
# server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-server
  namespace: demo-to
spec:
  selector:
    matchLabels:
      web: server
  template:
    metadata:
      labels:
        web: server
    spec:
      containers:
      - name: demo
        image: vasth/simple-server
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: simple-server-svc
  namespace: demo-to
spec:
  selector:
    web: server
  ports:
  - port: 18080
    targetPort: 18080
```

### 2. 客户端 demo

向服务端发送 GET 请求

```go
package main

import (
   "fmt"
   "io"
   "net/http"
   "time"
)

func main() {
   for {
      time.Sleep(time.Second)

      resp, err := http.Get("http://simple-server-svc.demo-to:18080/common/sauth")
      if err != nil {
         fmt.Printf("Get err: %v\n", err)
         continue
      }

      raw, err := io.ReadAll(resp.Body)
      if err != nil {
         fmt.Printf("ReadAll err: %v\n", err)
         continue
      }

      fmt.Println(string(raw))

      resp.Body.Close()
   }
}
```

Dockerfile和服务端一样，K8s 部署 YAML 如下：

```yaml
# client.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-client
  namespace: demo-from
spec:
  selector:
    matchLabels:
      web: client
  template:
    metadata:
      labels:
        web: client
    spec:
      containers:
      - name: demo
        image: vasth/simple-client
        imagePullPolicy: Always
```

### 3. 准备 K8s 环境

在阿里云 ACK 容器服务下，创建一个集群。注意，选择的节点操作系统要是 Alibaba Cloud Linux 2，不能用 CentOS，有内核问题。 我的测试环境是部署了两个工作节点。

### 4. 安装 Istio

Istio 1.11.3 版本文档：[https://istio.io/v1.11/docs/setup/getting-started](https://istio.io/v1.11/docs/setup/getting-started)

```shell
# 下载 1.11.3 版本的 istio
curl -O https://ghproxy.com/https://github.com/istio/istio/releases/download/1.11.3/istioctl-1.11.3-linux-amd64.tar.gz
tar zxvf istioctl-1.11.3-linux-amd64.tar.gz
mv istioctl /usr/local/bin/

# 安装 istio
istioctl install --set profile=demo -y

# 准备 namespace
kubectl create ns demo-from
kubectl create ns demo-to
kubectl label namespace demo-from istio-injection=enabled
kubectl label namespace demo-to istio-injection=enabled

# 部署客户端、服务端Pod
kubectl apply -f server.yaml
kubectl apply -f client.yaml
```

### 5. 复现

现在，可以打印服务端日志，看到打印的客户端IP是127.0.0.6。本文就是介绍：

1.为什么不是打印客户端的 Pod IP？
2. 如何修复？
3. 为什么是 127.0.0.6，而不是其他地址？

```shell
$ kubectl logs -n demo-to -l web=server
X-Forwarded-For=, RemoteAddr=127.0.0.6:44213
X-Forwarded-For=, RemoteAddr=127.0.0.6:44213
X-Forwarded-For=, RemoteAddr=127.0.0.6:44213
X-Forwarded-For=, RemoteAddr=127.0.0.6:44213
X-Forwarded-For=, RemoteAddr=127.0.0.6:44213
X-Forwarded-For=, RemoteAddr=127.0.0.6:44213
```

而客户端的实际IP地址是：

```shell
$ kubectl get po -n demo-from -owide
NAME                             READY   STATUS    RESTARTS   AGE     IP           NODE
simple-client-54f4f5cccb-mzjkw   2/2     Running   0          7m32s   10.20.0.78   cn-xxx.192.168.0.127
```

服务端 Pod IP 和 Service IP 分别是：

```shell
$ kubectl get po -n demo-to -owide
NAME                             READY   STATUS    RESTARTS   AGE     IP           NODE
simple-server-86b499d4dd-zd4pl   2/2     Running   0          4h37m   10.20.0.77   cn-xxx.192.168.0.127

$ kubectl get svc -n demo-to
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
simple-server-svc   ClusterIP   172.16.234.188   <none>        18080/TCP   5h44m
```

## 排查步骤

我们分别做以下排查步骤：

1. 查看服务端 Pod 的 istio-proxy 容器日志，确定容器看到 127.0.0.6 是来自哪个节点；
2. 查看服务端 Pod 的 iptables 规则，确定 Pod 流量转发规则；
3. 查看服务端 Pod 的 Envoy 配置规则，确定客户端 IP 被篡改的原因和实现规则。

### 1. 查看 istio-proxy 日志

istio-proxy 日志实际是 Envoy 访问日志[^2]，打印 Envoy 访问日志可以详细看到 Envoy 如何做代理转发的。如下打印服务端的 Envoy 日志：

```shell
$ kubectl logs -n demo-to -l web=server -c istio-proxy
[2022-12-17T09:30:39.883Z] "GET /common/sauth HTTP/1.1" 200 - via_upstream - "-" 0 60 0 0 "-" "Go-http-client/1.1" "57dd8171-f8e0-9ffc-bfc4-cdf4d9f5ee0f" "simple-server-svc.demo-to:18080" "10.20.0.77:18080" inbound|18080|| 127.0.0.6:44213 10.20.0.77:18080 10.20.0.78:33204 outbound_.18080_._.simple-server-svc.demo-to.svc.cluster.local default
[2022-12-17T09:30:40.885Z] "GET /common/sauth HTTP/1.1" 200 - via_upstream - "-" 0 60 0 0 "-" "Go-http-client/1.1" "84c216fd-6fe1-9986-b7ce-e259cdf61325" "simple-server-svc.demo-to:18080" "10.20.0.77:18080" inbound|18080|| 127.0.0.6:44213 10.20.0.77:18080 10.20.0.78:33204 outbound_.18080_._.simple-server-svc.demo-to.svc.cluster.local default
[2022-12-17T09:30:41.888Z] "GET /common/sauth HTTP/1.1" 200 - via_upstream - "-" 0 60 0 0 "-" "Go-http-client/1.1" "a67101af-e469-91ab-99b0-61bb5abfa108" "simple-server-svc.demo-to:18080" "10.20.0.77:18080" inbound|18080|| 127.0.0.6:44213 10.20.0.77:18080 10.20.0.78:33204 outbound_.18080_._.simple-server-svc.demo-to.svc.cluster.local default
```

各字段解析可参考 Envoy 文档[^3]，以及其他博客[^4]。在 Envoy accesslog 日志中，如果字段实际值是空，则用 "-" 字符串表示。

特定日志字段格式的说明：

`%RESPONSE_CODE_DETAILS%`：响应体状态码详细说明[^5]。

> - via_upstream：The response code was set by the upstream.
> - upstream_response_timeout: The upstream response timed out.
> - duration_timeout: The max connection duration was exceeded.

`%REQ(X?Y):Z%`：如 `%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%`，拿响应报文首部 X-ENVOY-UPSTREAM-SERVICE-TIME 的值。

> HTTP:
> - 从 HTTP Header 中拿到 X 首部的值，如果没有则拿首部 Y。Z 代表截取的字符长度。如果首部都不存在，则打印"-"。
>
> TCP/UDP:
> - 未实现默认"-"。

`%UPSTREAM_CLUSTER%`：上游服务的地址，格式如下[^6]：

> 入流量：inbound|portNumber|portName|Hostname
> 出流量：inbound|portNumber|portName|Hostname
> 
> 示例：
> 1. inbound|9080|http|productpage.istio-bookinfo-1-68819.svc.cluster.local
> 2. outbound|8000||httpbin.foo.svc.cluster.local

格式化处理之后的日志示例如下，以服务端Pod的代理容器日志为例：

```shell
{
  "start_time": "2022-12-17T09:30:41.888Z", // 请求开始时间
  "method": "GET",
  "path": "/common/sauth",
  "protocol": "HTTP/1.1", // 上游服务的协议，如果是 HTTP 则可选值分HTTP版本；如果是 TCP/UDP，则为空。
  "response_code": 200, // HTTP响应状态码
  "response_flags": "-",
  "response_code_details": "via_upstream", // HTTP响应状态码详情，比如：谁设置的此值
  "connection_termination_details": "-", // Envoy 终止请求的L4层原因
  "upstream_transport_failure_reason": "-",
  "bytes_received": 0,
  "bytes_send": 60,
  "duration": 0, // 从请求开始时间到最后一个字节发出的时间跨度
  "x-envoy-upstream-service-time": 0, // 响应报文中 X-ENVOY-UPSTREAM-SERVICE-TIME 首部的值
  "x-forwarded-for": "-", // 请求报文中 X-FORWARDED-FOR 首部的值
  "user-agent": "Go-http-client/1.1",
  "x-request-id": "a67101af-e469-91ab-99b0-61bb5abfa108",
  "authority": "simple-server-svc.demo-to:18080",
  "upstream_host": "10.20.0.77:18080", // Envoy Sidecar 要访问的服务地址。这里就是Pod里的istio-proxy要访问的同Pod里业务容器
  "upstream_cluster": "inbound|18080||", // 上游服务地址
  "upstream_local_address": "127.0.0.6:44213", // 服务端 Envoy 连接服务端时，Envoy 的本地地址（此时服务端获得的 IP 地址是这个）
  "downstream_local_address": "10.20.0.77:18080", // 下游连接中，当前 Envoy 的本地地址。客户端要连的地址
  "downstream_remote_address": "10.20.0.78:33204", // 下游连接中远端地址。对于 Inbound 流量，此值是「下游 pod-ip : 随机端口」
  "requested_server_name": "outbound_.18080_._.simple-server-svc.demo-to.svc.cluster.local",
  "route_name":"default"
}
```

**流量五元组**

流量五元组信息是 Envoy 日志里最重要的部分[^6][^7]，包含：

- **UPSTREAM_CLUSTER**：Envoy 计算出符合条件的转发目的主机集合，这个集合叫做 UPSTREAM_CLUSTER, 并根据负载均衡规则，从这个集合中选择一个 host 作为流量转发的接收端点；
- **UPSTREAM_HOST**：上面选择的这个端点就叫 UPSTREAM_HOST；
- **UPSTREAM_LOCAL_ADDRESS**：上游连接中，当前 Envoy 的本地地址，此值是「当前 pod-ip : 随机端口」；
- **DOWNSTREAM_LOCAL_ADDRESS**：下游连接中，当前 Envoy 的本地地址。即，下游请求方要连接的地址；
- **DOWNSTREAM_REMOTE_ADDRESS**：下游连接中远端地址。即，下游请求方的地址。

![](/images/envoy-model-1.png)

![](/images/envoy-model-2.png)

![](/images/envoy-model-3.png)

一些其他 Istio 运维文章[^8]

根据上面介绍的日志字段和流量五元组，可以知道，服务端Pod里的 server 程序，看到的请求过来的客户端源 IP 地址是 UPSTREAM_LOCAL_ADDRESS：127.0.0.6。

### 2. 查看 iptables 规则

查看 iptables 配置，列出 NAT（网络地址转换）表的所有规则，因为在 Init 容器启动的时候选择给 istio-iptables.sh[^9] 传递的参数中指定将入站流量重定向到 Envoy 的模式为 “REDIRECT”，因此在 iptables 中将只有 NAT 表的规格配置，如果选择 TPROXY 还会有 mangle 表配置[^10]。

查看容器 iptables，需要先进入容器 network namespace：

```shell
# 查容器ID
$ kubectl get po -n demo-to simple-server-56b4d7b7d-9zkbm -ojson | jq '.status.containerStatuses[] | {name,containerID}'
{
  "name": "demo",
  "containerID": "docker://cbd30c99c9f65430e32d2d0d46d4f0a56d261a4585d2a11385dc7ba994c9c948"
}
{
  "name": "istio-proxy",
  "containerID": "docker://a9cb6d70894c71102174cb8250a3de87d230e1586723938bf0a9ff1db7675969"
}

# 在容器所在节点，执行docker inspect
$ docker inspect -f {{.State.Pid}} cbd30c99c9
188902

# 进入容器 network namespace 后，就可以执行 ipstables 了
$ nsenter --target 188902 -n
```

iptables 命令使用说明[^11]，iptables 内置了5张表，分别是 raw、 nat、 mangle、 filter、 security。此处只需要关注 OUTPUT 链[^12]。

```shell
# 查看 NAT 表中规则配置的详细信息
$ iptables -t nat -L -v

# PREROUTING 链：用于目标地址转换（DNAT），将所有入站 TCP 流量跳转到 ISTIO_INBOUND 链上
Chain PREROUTING (policy ACCEPT 11627 packets, 698K bytes)
# pkts 多少个报文数量匹配该规则，bytes 报文大小，target 动作，port 协议，opt 选项，in/out: 入、出口网卡，source/destination: 源、目标ip/ip段
 pkts bytes target     prot opt in     out     source               destination         
11627  698K ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere            

# INPUT 链：处理输入数据包，目前没有配置规则
Chain INPUT (policy ACCEPT 11627 packets, 698K bytes)
 pkts bytes target     prot opt in     out     source               destination         

# OUTPUT 链：将所有出站数据包跳转到 ISTIO_OUTPUT 链上
Chain OUTPUT (policy ACCEPT 1059 packets, 96140 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   44  2640 ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere            

# POSTROUTING 链：所有数据包流出网卡时都要先进入POSTROUTING 链，内核根据数据包目的地判断是否需要转发出去，我们看到此处未做任何处理
Chain POSTROUTING (policy ACCEPT 1060 packets, 96200 bytes)
 pkts bytes target     prot opt in     out     source               destination         

# ISTIO_INBOUND 链：将所有入站流量重定向到 ISTIO_IN_REDIRECT 链上
# 但除了 15008、22、15090、15021、15020 端口
Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15008
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15090
11627  698K RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15021
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15020
    0     0 ISTIO_IN_REDIRECT  tcp  --  any    any     anywhere             anywhere            

# ISTIO_IN_REDIRECT 链：将所有的入站流量跳转到本地的 15006 端口，至此成功的拦截了流量到 Envoy 
Chain ISTIO_IN_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15006

# ISTIO_OUTPUT 链：选择需要重定向到 Envoy（即本地） 的出站流量，所有非 localhost 的流量全部转发到 ISTIO_REDIRECT。为了避免流量在该 Pod 中无限循环，所有到 istio-proxy 用户空间的流量都返回到它的调用点中的下一条规则，本例中即 OUTPUT 链，因为跳出 ISTIO_OUTPUT 规则之后就进入下一条链 POSTROUTING。如果目的地非 localhost 就跳转到 ISTIO_REDIRECT；如果流量是来自 istio-proxy 用户空间的，那么就跳出该链，返回它的调用链继续执行下一条规则（OUTPUT 的下一条规则，无需对流量进行处理）；所有的非 istio-proxy 用户空间的目的地是 localhost 的流量就跳转到 ISTIO_REDIRECT
Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  any    lo      127.0.0.6            anywhere       
# 这里 1337 是 istio-proxy 的 Group ID（runAsGroup: 1337）
# 要识别出哪些流量是由 envoy 发出的，这里使用了 UID 来标记，我们可以通过配置让 envoy 总是由特定的 UID 启动，这样就能够在 iptables 中通过 owner UID match 来过滤出由 envoy 发出的流量[^12]。     
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner UID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner UID match 1337
   43  2580 RETURN     all  --  any    any     anywhere             anywhere             owner UID match 1337
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner GID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             anywhere             owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             localhost           
    1    60 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere            

# ISTIO_REDIRECT 链：将所有流量重定向到 Envoy（即本地） 的 15001 端口
Chain ISTIO_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    1    60 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001
```

通过 iptables 配置，可以知道，发往服务端程序的请求，最终先被 Envoy 的 15006 端口接收；服务端返回的响应，先被 Envoy 的 15001 端口处理。这也会 istio-init 的如何初始化 iptables 一致[^10]：

```shell
$ kubectl get po -n demo-to simple-server-86b499d4dd-bjrjz  -ojson | jq .spec.initContainers[]

{
  "name": "istio-init",
  "image": "docker.io/istio/proxyv2:1.11.3",
  "args": [
    "istio-iptables",
    "-p",
    "15001",
    "-z",
    "15006",
    "-u",
    "1337",
    "-m",
    "REDIRECT",
    "-i",
    "*",
    "-x",
    "",
    "-b",
    "*",
    "-d",
    "15090,15021,15020"
  ],
  "securityContext": {
    "allowPrivilegeEscalation": false,
    "privileged": false,
    "readOnlyRootFilesystem": false,
    "runAsGroup": 0,
    "runAsNonRoot": false,
    "runAsUser": 0
  }
}
```

因此，要搞清楚为什么是服务端看到的是 127.0.0.6，就要看 Envoy 的配置了。

### 3. istio-config 查看 Envoy 配置

接着上面的分析，由于所有的 18080 端口的请求，都被 iptables 发往 15006 端口，现在来看 15006 端口的 Envoy 配置。

```shell
$ istioctl proxy-config listener -n demo-to simple-server-86b499d4dd-bjrjz --port=15006

ADDRESS PORT  MATCH                                                                    DESTINATION
0.0.0.0 15006 Addr: *:15006                                                            Non-HTTP/Non-TCP
0.0.0.0 15006 Trans: tls; App: istio-http/1.0,istio-http/1.1,istio-h2; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: raw_buffer; App: HTTP; Addr: 0.0.0.0/0                            InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; App: TCP TLS; Addr: 0.0.0.0/0                                InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: raw_buffer; Addr: 0.0.0.0/0                                       InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; Addr: 0.0.0.0/0                                              InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; App: istio-http/1.0,istio-http/1.1,istio-h2; Addr: *:18080   Cluster: inbound|18080||
0.0.0.0 15006 Trans: raw_buffer; App: HTTP; Addr: *:18080                              Cluster: inbound|18080||
0.0.0.0 15006 Trans: tls; App: TCP TLS; Addr: *:18080                                  Cluster: inbound|18080||
0.0.0.0 15006 Trans: raw_buffer; Addr: *:18080                                         Cluster: inbound|18080||
0.0.0.0 15006 Trans: tls; Addr: *:18080                                                Cluster: inbound|18080||
```

流量路由分为 Inbound 和 Outbound 两个过程，下面将根据上文中的示例及 sidecar 的配置为读者详细分析此过程[^13]。

**理解 Inbound Handler**

从该 Pod 的 Listener 列表的最后两行中可以看到，0.0.0.0:15006/TCP 的 Listener（其实际名字是 virtualInbound）监听所有的 Inbound 流量，其中包含了匹配规则，来自任意 IP 的对 18080 端口的访问流量，将会路由到 inbound|18080|| Cluster，如果你想以 Json 格式查看该 Listener 的详细配置，可以执行 istioctl proxy-config listeners -n demo-to simple-server-86b499d4dd-bjrjz --port 15006 -o json 命令，你将获得类似下面的输出。

```shell
[
  {
    "name": "virtualInbound",
    "address": {
      "socketAddress": {
        "address": "0.0.0.0",
        "portValue": 15006
      }
    },
    "filterChains": [
      {
        "filterChainMatch": {
          "destinationPort": 15006
        },
        "filters": [{
          # ... 省略
        }],
        "name": "virtualInbound-blackhole"
      },
      # ... 省略
      {
        "filterChainMatch": {
          "destinationPort": 18080,
          "transportProtocol": "raw_buffer"
        },
        "filters": [
          {
            "name": "envoy.filters.network.tcp_proxy",
            "typedConfig": {
              "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
              "statPrefix": "inbound|18080||",
              "cluster": "inbound|18080||"
            }
          }
        ],
        "name": "0.0.0.0_18080"
      },
    ]
  }
]
```

通过 envoy.filters.network.tcp_proxy，知道请求 15006 收到的请求，会发往 0.0.0.0_18080 Cluser。继续查看 0.0.0.0_18080 Cluster 的配置规则：

```shell
$ istioctl proxy-config cluster -n demo-to simple-server-86b499d4dd-bjrjz --fqdn "inbound|18080||" 
SERVICE FQDN     PORT      SUBSET     DIRECTION     TYPE             DESTINATION RULE
                 18080     -          inbound       ORIGINAL_DST     
```

TYPE 为 ORIGINAL_DST，表示将流量发送到原始目标地址（Pod IP），因为原始目标地址即当前 Pod。应该注意到 upstreamBindConfig.sourceAddress.address 的值被改写为了 127.0.0.6，而且对于 Pod 内流量是通过 lo 网卡发送的，这刚好呼应了上文中的 iptables ISTIO_OUTPUT 链中的第一条规则，根据该规则，流量将被透传到 Pod 内的应用容器。

```shell
$ istioctl proxy-config cluster -n demo-to simple-server-86b499d4dd-bjrjz --fqdn "inbound|18080||"  -ojson

[
    {
        "name": "inbound|18080||",
        "type": "ORIGINAL_DST",
        "connectTimeout": "10s",
        "lbPolicy": "CLUSTER_PROVIDED",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295,
                    "trackRemaining": true
                }
            ]
        },
        "typedExtensionProtocolOptions": {
            "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {
                "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
                "useDownstreamProtocolConfig": {
                    "httpProtocolOptions": {},
                    "http2ProtocolOptions": {
                        "maxConcurrentStreams": 1073741824
                    }
                }
            }
        },
        "cleanupInterval": "60s",
        "upstreamBindConfig": {
            "sourceAddress": {
                "address": "127.0.0.6",
                "portValue": 0
            }
        },
        "metadata": {
            "filterMetadata": {
                "istio": {
                    "services": [
                        {
                            "host": "simple-server-svc.demo-to.svc.cluster.local",
                            "name": "simple-server-svc",
                            "namespace": "demo-to"
                        }
                    ]
                }
            }
        }
    }
]
```

现在我们知道，127.0.0.6 是 Envoy 绑定的地址。那如何解决了？

## 解决办法

解决办法很简单，只需要 Istio 的 interceptionMode 模式从默认 REDIRECT 切换为 TPROXY，具体可以参考阿里云的文档[^14]。

```shell
kubectl patch deployment -n demo-to simple-server  -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/interceptionMode":"TPROXY"}}}}}'
```

查看修改后的 iptables（仍要先进入容器的 namespace，再执行 iptables 命令）：

```shell
$ iptables -t mangle -L -v

# 可以看到，拦截的逻辑比较简单，仅仅改了 PREROUTING （关注进入的封包）链
Chain PREROUTING (policy ACCEPT 31760 packets, 7988K bytes)
 pkts bytes target     prot opt in     out     source               destination         
40639   14M ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere            
12304 3649K CONNMARK   tcp  --  any    any     anywhere             anywhere             mark match 0x539 CONNMARK and 0x0

Chain INPUT (policy ACCEPT 41248 packets, 14M bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 32921 packets, 13M bytes)
 pkts bytes target     prot opt in     out     source               destination         
 8202 2747K RETURN     tcp  --  any    lo      anywhere             anywhere             mark match 0x539
    0     0 MARK       tcp  --  any    lo      anywhere            !localhost            owner UID match 1337 MARK set 0x53a
    0     0 MARK       tcp  --  any    lo      anywhere            !localhost            owner GID match 1337 MARK set 0x53a
 4102  902K CONNMARK   tcp  --  any    any     anywhere             anywhere             connmark match  0x539 CONNMARK and 0x0

Chain POSTROUTING (policy ACCEPT 32921 packets, 13M bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain ISTIO_DIVERT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 9487 6308K MARK       all  --  any    any     anywhere             anywhere             MARK set 0x539
 9487 6308K ACCEPT     all  --  any    any     anywhere             anywhere            

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination         
12304 3649K RETURN     tcp  --  any    any     anywhere             anywhere             mark match 0x539
    0     0 RETURN     tcp  --  lo     any     127.0.0.6            anywhere            
 6516 3339K RETURN     tcp  --  lo     any     anywhere             anywhere             mark match ! 0x53a
 # 不拦截特殊端口 
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15090
12331  898K RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15021
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15020
 # 如果SRC_IP:SRC_PORT:DST_IP:DST_PORT已经建立拦截，则打标记，接受封包
 9487 6308K ISTIO_DIVERT  tcp  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
 # 否则，如果目的地不是127.0.0.1，则重定向给Envoy
    1    60 ISTIO_TPROXY  tcp  --  any    any     anywhere             anywhere            

# 对于目的地址不是127.0.0.1的封包，进行透明代理，发送给Envoy的15006监听器，给封包打标记1337（十六进制是0x539）
Chain ISTIO_TPROXY (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    1    60 TPROXY     tcp  --  any    any     anywhere            !localhost            TPROXY redirect 0.0.0.0:15006 mark 0x539/0xffffffff
```


看完 TPROXY1 修改的 mangle，再看下 nat：

```shell
$ iptables -t nat  -L -v

Chain PREROUTING (policy ACCEPT 2842 packets, 171K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 2842 packets, 171K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 294 packets, 25636 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   25  1500 ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere            

Chain POSTROUTING (policy ACCEPT 294 packets, 25636 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain ISTIO_INBOUND (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15008

Chain ISTIO_IN_REDIRECT (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15006

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  any    lo      127.0.0.6            anywhere            
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner UID match 1337
   13   780 RETURN     all  --  any    lo      anywhere             anywhere             ! owner UID match 1337
    0     0 RETURN     all  --  any    any     anywhere             anywhere             owner UID match 1337
# 根据用户不同决定行为，如果GID为1337，意味着是Envoy进程发起的封包，否则是其它进程发起的
# 对于将从lo发出的封包，如果用户是Envoy，目的地址非127.0.0.1的，则重定向到入站虚拟监听器15006
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner GID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner GID match 1337
   12   720 RETURN     all  --  any    any     anywhere             anywhere             owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             localhost           
    0     0 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere            

# 看样子15006是需要将所有入站流量重定向到的端口，而在TPROXY中将入站流量都重定向到15001
Chain ISTIO_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001
```

继续看 15001 端口的 listener：

```shell
$ istioctl proxy-config listeners -n demo-to simple-server-56b4d7b7d-9zkbm --port 15001

ADDRESS PORT  MATCH         DESTINATION
0.0.0.0 15001 ALL           PassthroughCluster
0.0.0.0 15001 Addr: *:15001 Non-HTTP/Non-TCP
```

```shell
istioctl proxy-config listeners -n demo-to simple-server-56b4d7b7d-9zkbm --port 15001 -ojson

[
    {
        "name": "virtualOutbound",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 15001
            }
        },
        "filterChains": [
            {
              // ...省略
            },
            {
                "filters": [
                    {
                        "name": "envoy.filters.network.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
                            "statPrefix": "PassthroughCluster",
                            "cluster": "PassthroughCluster",
                        }
                    }
                ],
                "name": "virtualOutbound-catchall-tcp"
            }
        ],
        "useOriginalDst": true,
        "transparent": true,
        "trafficDirection": "OUTBOUND",
    }
]
```

继续看 envoy.filters.network.tcp_proxy 对应的 PassthroughCluster Cluster：

```shell
$ istioctl proxy-config cluster -n demo-to simple-server-56b4d7b7d-9zkbm  --fqdn "PassthroughCluster"  -ojson

[
  {
    "name": "PassthroughCluster",
    "type": "ORIGINAL_DST",
    "connectTimeout": "10s",
    "lbPolicy": "CLUSTER_PROVIDED",
    "circuitBreakers": {
      "thresholds": [
        {
          "maxConnections": 4294967295,
          "maxPendingRequests": 4294967295,
          "maxRequests": 4294967295,
          "maxRetries": 4294967295,
          "trackRemaining": true
        }
      ]
    },
    "typedExtensionProtocolOptions": {
      "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {
        "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
        "useDownstreamProtocolConfig": {
          "httpProtocolOptions": {},
          "http2ProtocolOptions": {
            "maxConcurrentStreams": 1073741824
          }
        }
      }
    },
    "filters": [
      {
        "name": "istio.metadata_exchange",
        "typedConfig": {
          "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
          "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
          "value": {
            "protocol": "istio-peer-exchange"
          }
        }
      }
    ]
  }
]
```

这里可以看到，没有修改源 IP。

## 参考资料

[^1]: [在服务网格环境下如何保持服务访问时的客户端源IP](https://help.aliyun.com/document_detail/464794.html)
[^2]: [Isito Doc: 获取 Envoy 访问日志](https://istio.io/latest/zh/docs/tasks/observability/logs/access-log/)
[^3]: [Envoy Doc: Access logging](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)
[^4]: [Understanding Istio Access Logs](https://dev.to/aws-builders/understanding-istio-access-logs-2k5o)
[^5]: [Envoy Doc: Response Code Details](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/response_code_details#response-code-details)
[^6]: [Observe Service Mesh with SkyWalking and Envoy Access Log Service](https://tetrate.io/blog/observe-service-mesh-with-skywalking-and-envoy-access-log-service)
[^7]: [Istio 运维实战：Envoy 日志调试指南](https://istio-operation-bible.aeraki.net/docs/debug-istio/envoy-log/)
[^8]: [istio 常见的 10 个异常](https://www.bbsmax.com/A/A2dmD9l45e/)
[^9]: [istio-iptables 源码](https://github.com/istio/istio/blob/1.11.3/tools/istio-iptables/pkg/cmd/root.go)
[^10]: [理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持](https://jimmysong.io/blog/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)
[^11]: [iptables详解-查询命令](https://kiyonlin.github.io/post/work/iptables/iptables-2-%E6%9F%A5%E8%AF%A2%E5%91%BD%E4%BB%A4/)
[^12]: [Linux - iptables](https://dong-dada.github.io/linux/2021/05/06/iptables.html)
[^13]: [Istio 中的 Sidecar 注入、透明流量劫持及流量路由过程详解](https://jimmysong.io/blog/sidecar-injection-iptables-and-traffic-routing/)
[^14]: [在服务网格环境下如何保持服务访问时的客户端源IP](https://help.aliyun.com/document_detail/464794.html)
[^15]: [Istio Sidecar 流量拦截机制分析](https://blog.yingchi.io/posts/2020/6/istio-sidecar-proxy.html)
[^16]: [Istio中的透明代理问题](https://blog.gmem.cc/istio-tproxy)