---
title: "Kubebuilder Source Code Part 2"
date: 2022-01-15T21:20:28+08:00
draft: true
---

Operator 控制器
K8s 定义了很多抽象内部资源来描述不同工作负载类型，比如 Deployment 用于无状态应用1部署、StatefulSet 用于有状态应用、CronJob 适用于运行定时任务。

Deployment 和 StatefulSet 的区别2：

Deployment 创建的 Pod 之间没有顺序，服务通过 Service 的 Service IP 暴露。Deployment 也可以使用持久化存储卷实现有状态应用部署，但有使用限制。Deployment 只支持通过 .spec.template.spec.volumes.persistentVolumeClaim 引用一个 PVC（提前创建）。如果该 PVC 访问模式支持且设置为 RWO，Deployment 副本数量必须为 1（单 Pod）；否则，使用 RWX 模式，多个 Pod 共享存储。
StatefulSet：每个 Pod 有自己的存储，通过 .spec.volumeClaimTemplates 为每个 Pod 创建一个独立的 PV 保存其数据和状态。即使删除 StatefulSet 或 Pod 宕机，创建的 PVC 仍保留其数据并可以在 Pod 恢复后重新恢复绑定。StatefulSet 和无头服务配合使用（.spec.clusterIP=None），无头服务不做负载均衡，返回所有关联 Pod 的 IP 地址列表。
这些 K8s 内部资源的状态由对应资源的控制器来维护，比如 Deployment 对应 Deployment Controller。K8s 控制组件 kube-controller-manager 包含了所有内部资源控制器。控制器本质上是一个控制回路（control loop）死循环进程，Watch 资源状态，并做出相应调整，调协当前状态（status）至期望状态（spec），如：滚动更新，恢复宕机的 Pod。对于运行在 Pod 中的程序，如下图3中的 DB 和 Web 程序，他们本身并无感知自身运行在 K8s 环境中。应用运维由 K8s 控制器来完成。

images/img.png

虽然 K8s 在容器编排和基础设施层做了统一的抽象，实现了自动化运维，但对特定应用运维以及业务运维上，仍需要人力介入。比如，在 K8s 上安装、升级 Redis 集群，我们需要手动组装和更新多个 K8s 资源，包括：ConfigMap、StatefulSet、Service 等（或通过 Helm）。更复杂的场景是实现业务运维自动化，比如根据数据库某个表字段的更新，重启另一个业务应用。这种场景对控制器提出了更高的要求，要求控制器能拥有业务运维知识，根据特定业务场景自动化管理相关组件。Kubernetes 自身的基础模型元素已经无法支撑不同业务领域下复杂的自动化场景。于是，K8s 社区提出了 Custom Resource（CR）和 Operator 的概念。基于自定义资源和自定义资源控制器，来扩展原生 Kubernetes 的能力。

Operator 的开发者可以将业务相关的运维逻辑编写到 Operator 和 CR 中，并将 CR 应用到 K8s 集群。Operator 控制器通过访问 K8s API，像操作 Deployment 等内部资源一样，监听对应 CR 状态，并创建修改其他 K8s 内部资源或外部依赖。因此，不同于之前图片中橙色框中 DB 和 Web 应用，Operator 需要与 kube-apiserver 通信（C/S 架构）。一个 Operator 应用示例如下，我们可以定义一个 Database 类型（kind）的 CR 实现 Database Operator 声明式配置的方式一键部署数据库服务。

1
2
3
4
5
6
7
8
9
kind: Database
metadata:
name: mydb
spec:
type: mysql
user: root
password: root
status:
phase: Running
Kubebuilder
从前文我们知道，开发一个 Operator 需要首先定义 CR 并完成 Operator 代码。K8s 社区提供了 Kubebuilder 开发框架。本节介绍如何实现前面提到的业务运维需求，监听数据库表字段变化，重启业务应用。我们将该 Operator 命名为 Autorebooter，并定义 RebootPolicy 自定义资源。在 RebootPolicy 定义目标业务应用和数据库监听逻辑。总体上，我们需要定义一个 CR，一个 Operator，并部署到 K8s 集群上。 Kubebuilder 详细使用手册可参考官方开发文档4。

初始化项目
安装 Kubebuilder，当前最新版本是 v2.3.0。

1
2
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
通过 Kubebuilder 创建 Autorebooter Operator：

1
2
cd autorebooter
kubebuilder init --domain my.domain --repo my.domain/autorebooter
为 CR 和 Operator 控制器创建模板代码：这里通过 kubebuilder create api 命令创建。group、version、kind 分别表示 CR 资源的 API 组、版本和资源类型名称。基本上所有 K8s 资源都有这三个部分，如 Job 资源是 batch API Group 下，最新的稳定 API 版本是 v1。该 API Group 下的其他资源还包括 CronJob。GVK 是 K8s 源码中的重要概念，一个 GVK 唯一对应一种资源类型。我们会在本系列后面部分继续介绍。

1
kubebuilder create api --group autoreboot --version v1alpha1 --kind RebootPolicy
输入以上命令，当提示是否 Create Resource [y/n] 以及 Create Controller [y/n] 时，按 y 会创建以下项目结构：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
.
├── api # CR
│   └── v1alpha1
│       ├── groupversion_info.go
│       └── rebootpolicy_types.go # 自定义资源 Go 结构体声明
├── bin
├── config # YAMl 文件
│   ├── crd # 自定义资源定义（Custom Resource Definition）
│   │       # 用于自定义资源注册和服务发现
│   ├── manager # Operator Deployment 部署 YAML 文件
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── rbac # RBAC 权限相关
│   └── samples # CR 实例
├── controllers
│   └─ rebootpolicy_controller.go # 控制器代码，实现了 k8s go-client
│                                 # 访问 kube-apiserver
├── go.mod
├── Dockerfile
├── Makefile # Operator 部署安装命令
└── main.go # Operator 入口程序
定义自定义资源
编写控制器
部署

