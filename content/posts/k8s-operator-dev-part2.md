---
title: "K8s Operator 开发（二）：API 注册"
date: 2022-01-17T21:33:05+08:00
draft: true
toc: true
categories: ["Kubernetes"]
series: ["Kubernetes-operator-开发"]
---

## K8s API

Operator 通过与 kube-apiserver 通信访问 K8s 资源和自定义资源。kube-apiserver 是 Kubernetes 集群的核心组件和入口，暴露 RESTful HTTP API 接口，支持标准的 POST，GET，UPDATE，DELETE，PATCH 方法以及额外支持 WATCH 和 LIST 操作。


