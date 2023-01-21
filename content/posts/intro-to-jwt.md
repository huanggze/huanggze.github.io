---
title: "JWT介绍"
date: 2023-01-21T11:18:26+08:00
toc: true
categories: ["security"]
---

## Cookie、Session 和 Token 辨析

HTTP 是无状态的协议（对于事务处理没有记忆能力，每次客户端和服务端会话完成时，服务端不会保存任何会话信息）：每个请求都是完全独立的，服务端无法确认当前访问者的身份信息，无法分辨上一次的请求发送者和这一次的发送者是不是同一个人。所以服务器与浏览器为了进行会话跟踪（知道是谁在访问我），就必须主动的去维护一个状态，这个状态用于告知服务端前后两个请求是否来自同一浏览器。而这个状态需要通过 cookie 或者 session 去实现[^1]。

> TCP、UDP 都是无状态协议。有状态协议有 TCP，TCP下一次传输的报文段和上一次传输的报文段是有顺序关系的，最终要按照报文段里的序列号对所有报文段进行重排序。。

### Cookie

Cookie 存储在客户端，它是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上；

![](https://pic1.zhimg.com/80/v2-ca7a19608f525e212a3d5f9ffa6ead60_1440w.webp)
[^2]

### Session

Session 是另一种记录服务器和客户端会话状态的机制。基于 cookie 实现，session 存储在服务器端，sessionId 会被存储到客户端的cookie 中；

![](https://pic3.zhimg.com/80/v2-387c8c3f18f99c06d9c6c6d8ae915656_1440w.webp)

session 的痛点：

看起来通过 cookie + session 的方式是解决了问题， 但是我们忽略了一个问题，上述情况能正常工作是因为我们假设 server 是单机工作的，但实际在生产上，为了保障高可用，一般服务器至少需要两台机器，通过负载均衡的方式来决定到底请求该打到哪台机器上。

![](/images/intro-to-jwt.png)

解决办法有三种：

1. session 复制：

A 生成 session 后复制到 B, C，这样每台机器都有一份 session，无论添加购物车的请求打到哪台机器，由于 session 都能找到，故不会有问题

![](/images/intro-to-jwt-0.png)

这种方式虽然可行，但缺点也很明显：

- 同一样的一份 session 保存了多份，数据冗余；
- 如果节点少还好，但如果节点多的话，特别是像阿里，微信这种由于 DAU 上亿，可能需要部署成千上万台机器，这样节点增多复制造成的性能消耗也会很大。

2. session 粘连

这种方式是让每个客户端请求只打到固定的一台机器上，比如浏览器登录请求打到 A 机器后，后续所有的添加购物车请求也都打到 A 机器上，Nginx 的 sticky 模块可以支持这种方式，支持按 ip 或 cookie 粘连等等，如按 ip 粘连方式如下

![](/images/intro-to-jwt-1.png)

这样的话每个 client 请求到达 Nginx 后，只要它的 ip 不变，根据 ip hash 算出来的值会打到固定的机器上，也就不存在 session 找不到的问题了，当然不难看出这种方式缺点也是很明显，对应的机器挂了怎么办？

3. session 共享

这种方式也是目前各大公司普遍采用的方案，将 session 保存在 redis，memcached 等中间件中，请求到来时，各个机器去这些中间件取一下 session 即可。

![](/images/intro-to-jwt-2.png)

缺点其实也不难发现，就是每个请求都要去 redis 取一下 session，多了一次内部连接，消耗了一点性能，另外为了保证 redis 的高可用，必须做集群，当然了对于大公司来说, redis 集群基本都会部署，所以这方案可以说是大公司的首选了。

### Token（无需存储 Session！）

当 server 收到浏览器传过来的 token 时，它会首先取出 token 中的 header + payload，根据密钥生成签名，然后再与 token 中的签名比对，如果成功则说明签名是合法的，即 token 是合法的。而且你会发现 payload 中存有我们的 userId，所以拿到 token 后直接在 payload 中就可获取 userid，避免了像 session 那样要从 redis 去取的开销。

![](https://miro.medium.com/max/1400/1*RvUzEHQi5JEifWCBY4Rkng.webp)
[^3]

## JWT

JSON Web Token（JWT）用于授权而非身份验证。通过身份验证，我们验证用户名和密码是否有效，并将用户登录到系统中。通过授权，我们可以验证发送到服务器的请求是否属于通过身份验证登录的用户，从而可以授予该用户访问系统的权限，继而批准该用户使用获得的 token 访问路由、服务和资源。

### JWT 的结构

JSON Web Token 由三部分组成，以点（.）分隔，分别是[^4]：
- Header（标头）：指定了签名算法
- Payload（有效负载）：可以指定用户 id，过期时间等数据
- Signature（签名）：server 根据 header 知道它该用哪种签名算法，再用密钥根据此签名算法对 head + payload 生成签名用于校验

因此，JWT 通常如下所示：

```
xxxxxx.yyyyyyy.zzzzzzzz
```

![](/images/intro-to-jwt-4.png)

![](https://quant67.com/post/wauth/jwt.io-exp.png)

只要 server 保证密钥不泄露，那么生成的 token 就是安全的，因为如果伪造 token 的话在签名验证环节是无法通过的，就此即可判定 token 非法。

token的缺点是：一旦生成无法让其失效，必须等到其过期才行，这样的话如果服务端检测到了一个安全威胁，也无法使相关的 token 失效。所以 token 更适合一次性的命令认证，设置一个比较短的有效期！

## JWKS

JSON Web Key Set，是多个JWK组合在一起的一种格式。JWKS 提供 JWKS URI 来提供证书查询。验证客户端 token 时，检查 header 中的 kid 来找到正确的公钥证书，验证签名。

![](https://curity.io/images/resources/openidconnect/assertions/assertion-verification.svg)
[^5]

## 参考资料

[^1]: [Cookie、Session、Token与JWT解析](https://www.jianshu.com/p/cab856c32222)
[^2]: [session、cookie、token工作原理及区别](https://zhuanlan.zhihu.com/p/445149223)
[^3]: [Part 6. Authentication with JWT, JSON Web Token](https://losikov.medium.com/part-6-authentication-with-jwt-json-web-token-ec78459b9c88)
[^4]: [JWT 介绍 - Step by Step](https://www.cnblogs.com/ittranslator/p/14595165.html)
[^5]: [Client Assertions and the JWKS URI](https://curity.io/resources/learn/client-assertions-jwks-uri/)