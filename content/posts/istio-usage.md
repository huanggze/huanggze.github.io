---
title: "Istio Usafe"
date: 2022-01-26T17:23:41+08:00
draft: true
---

When you use peer authentication policies and mutual TLS, Istio extracts the identity from the peer authentication into the `source.principal`. Similarly, when you use request authentication policies, Istio assigns the identity from the JWT to the `request.auth.principal`. Use these principals to set authorization policies and as telemetry output.

---> https://istio.io/latest/docs/reference/config/security/conditions/



TLS 升级
https://istio.io/latest/docs/concepts/security/#updating-authentication-policies

istio 与 jwt
https://istio.io/latest/docs/concepts/security/#authorization-policies
https://datatracker.ietf.org/doc/html/rfc7519#section-4
```text
eyJhbGciOiJSUzI1NiIsImtpZCI6InhQUUE1SFliT2hPN3R6d3J4UlRSQk9iZ1hBSWlQbXZUUE9wMks4X01rY0kifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tNjVnOWQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNjMTQ4NjE4LTU5MWQtNGIzZC1iMzc2LTBkMzNiNDRjYmIwMSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.Z642jt7a_0xahz-MxigGMBJVjMgjC085UvmNBbKLDVgA68fiNGnbLrTMRkrjRgaleocO6sKxpNRHfBRQtiD6cC-fS5ToLNylyL_VT28CzINFxnrWP0IKN7JRDEfHBuVM1IzKBmVZvvmTCKBtz6kbOc3uJElmpHGxqLcfYT6uvmwLwp_i5rwCJvk6E7FxYf1_XWHDja1v0vnwiHwhaQ8Hj9WGC62vEXj3gM_KAF9GPLfFtigOWNkjbrmUcSCaXfONHEo7lkyIUGKYmmh-Ni1ya6_1354yKvfDk1e6mwjX0VDiqp7qzPxygul1hUm9bc5bKtqCJXIXQIrqiSD64bBu4A
```

什么是 istio 的 root namespace？——》 the default is istio-system.


authorizationpolicy 里 source.principals 和 source.requestprincipals 有什么区别？


性能压测

多集群

Starting with Istio 1.8, the Istio agent on the sidecar will ship with a caching DNS proxy, programmed dynamically by Istiod. Istiod pushes the hostname-to-IP-address mappings for all the services that the application may access based on the Kubernetes services and service entries in the cluster.


envoy 是 kube-proxy 的替代？
https://istio.io/latest/blog/2020/dns-proxy/
https://thenewstack.io/why-do-you-need-istio-when-you-already-have-kubernetes/


----

DNS spoofing
https://istio.io/latest/blog/2020/dns-proxy/#vm-access-to-kubernetes-services
DNS Spoofing on Kubernetes Clusters
https://blog.aquasec.com/dns-spoofing-kubernetes-clusters

DNS in k8s
https://mrkaran.dev/posts/ndots-kubernetes/