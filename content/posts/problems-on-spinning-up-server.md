---
title: "Problems on Spinning Up Server"
date: 2022-03-02T11:05:36+08:00
draft: true
---

动服务端的时候得特别注意一下。和之前 istioctl dashboard jaeger --address=192.168.0.70，不加 --address 浏览器访问不到是一个道理

grpc 的问题

```bash
$ k logs client-5994698fff-mdw56 -c client
2022/03/02 02:57:40 could not greet: rpc error: code = Unavailable desc = upstream connect error or disconnect/reset before headers. reset reason: connection failure, transport failure reason: delayed connect error: 111
```