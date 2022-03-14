---
title: "Linux 命令：ss"
date: 2022-03-02T14:09:57+08:00
toc: true
categories: ["Linux"]
series: ["Linux 常用命令"]
---

ss（Socket Statistics）[^1]命令功能类似于 netstat，用于获取 socket 统计信息，比 netstat 能展示更多信息。

## 命令格式

ss [options] [ FILTER ]

### 选项

| 参数               | 描述                                   |
|------------------|--------------------------------------|
| -H, -\-no-header | 不打印头部标题行                             |
| -n, -\-numeric   | 不解析域名                                |
| -r, -\-resolve   | 解析域名                                 |
| -a, -\-all       | 显示正在监听和非监听（对于 TCP 来说是已建立的连接）的 socket |
| -l, -\-listening | 显示正在监听的 socket（默认不显示）                |
| -p, -\-processes | socket 对应的进程                         |
| -4, -\-ipv4      | 只显示 ipv4 socket                      |
| -t, -\-tcp       | 只显示 TCP socket                       |
| -u, -\-udp       | 只显示 UDP socket                       |
| -x, -\-unix      | 只显示 Unix 域套接字（Unix domain sockets）   |

### 过滤器

1. 按 socket 状态过滤

```bash
ss state [ STATE-FILTER ] [ EXPRESSION ]
ss exclude [ STATE-FILTER ] [ EXPRESSION ]
```

TCP 状态有 LISTEN、SYN-SENT、SYN-RECEIVED、SYN-RECEIVED、ESTABLISHED、FIN-WAIT-1、FIN-WAIT-2、CLOSE-WAIT、CLOSING、LAST-ACK、
TIME-WAIT、CLOSED[^2]。STATE-FILTER 对应可选项有 all、connected、synchronized、bucket、big。

过滤表达式的写法可参考 ss 手册[^3]。

```bash
# Display all established ssh connections.
ss -o state established '( dport = :ssh or sport = :ssh )'
```

2. 按地址/端口过滤

```bash
ss dst [ADDRESS_PATTERN]
ss src [ADDRESS_PATTERN]
ss dport [OP] [PORT]
ss sport [OP] [PORT] 
```

```bash
# sockets connected to port 22 on network 10.0.0.0...255.
ss dst 10.0.0.0/24:22
```

前面两种，状态过滤和地址过滤，可形成组合过滤：

```bash
# List all the tcp sockets in state FIN-WAIT-1 for our apache to network 193.233.7/24 and look at their timers.
ss -o state fin-wait-1 '( sport = :http or sport = :https )' dst 193.233.7/24
```

### 输出

ss 的输出格式包含六列[^1]：

| 列                  | 描述                                              |
|--------------------|-------------------------------------------------|
| Netid              | socket 类型和传输层协议，tcp，udp，raw，u_str（unix_stream）等 |
| State              | socket 状态                                       |
| Recv-Q             | 网络接受队列[^4]                                      |
| Send-Q             | 网络发送队列                                          |
| Local Address:Port | 本地 socket                                       |
| Peer Address:Port  | 对端 socket                                       |
| 额外信息               | -o，-e，-p 开启                                     |

[^1]: [SS Utility: Quick Intro](https://www.cyberciti.biz/files/ss.html)
[^2]: [RFC: 793 [Page 20]](https://www.rfc-editor.org/rfc/rfc793.txt)
[^3]: [ss(8) — Linux manual page](https://man7.org/linux/man-pages/man8/ss.8.html)
[^4]: [What's the meaning of Recv-Q and Send-Q in "netstat -an"?](https://www.ibm.com/support/pages/node/6537582)