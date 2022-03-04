---
title: "K8s Request Insufficient"
date: 2022-03-04T15:08:38+08:00
draft: true
---

是request不足了，默认调度是看request的，您那边可以执行kubectl describe nodes nodename 看具体谁申请的多，目前有两个方案


方案1:您那边优化下request使用情况，可以修改已有deployment应用的，不过这个会重建pod您谨慎操作


方案2:您那边可以考虑在扩容一个节点用来抗压力。


100%就代表没有了。


节点上请求量和使用量算法：https://help.aliyun.com/document_detail/92802.html?spm=a2c4g.11174283.6.669.43b92cee1268Vx
可以参考这个文档理解下这两个参数：https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/


看起来/目录使用了71% 可以执行du -sh /*来分析都是谁占用的
