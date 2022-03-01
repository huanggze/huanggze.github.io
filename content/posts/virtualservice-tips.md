---
title: "Virtualservice Tips"
date: 2022-03-01T17:05:29+08:00
draft: true
---

https://learnwebanalytics.com/what-kind-of-regular-expression-engine-does-google-analytics-use-and-can-i-use-negative-regex-in-google-analytics/

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  # 虚拟服务的名称
  name: noah-virtualservice
  # 资源所属的namesapce
  namespace: noahsdk
spec:
  hosts:
    # 别改
    - '*'
  gateways:
    # 和gateway里的name一致
    - noah-gateway
  http:
    - match:
        - uri:
            # 限定响应的url前缀
            # 不限定时写/
            # 把限定前缀的写在前面，因为是按顺序匹配的
            prefix: /96618781
      route:
        - destination:
            # 和service里的name一致
            host: noahuni-noahsdk-rc-v3-71-0-17642
          # 在不同service间分配权重
          weight: 100      
    - match:
        - uri:
            # 限定响应的url前缀
            # 不限定时写 ^/\d{8}/.*
            # 把限定前缀的写在前面，因为是按顺序匹配的
            regex: ^/\d{8}/.*
      route:       
        - destination:
            # 和service里的name一致
            host: noah-public-noahsdk-rc-v3-71-0-17642
          # 在不同service间分配权重
          weight: 100
```