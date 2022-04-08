---
title: "Linux 命令：ulimit"
date: 2022-04-08T21:41:46+08:00
categories: ["Linux"]
series: ["Linux 常用命令"]
---

ulimit 可以用来控制 shell 执行程序资源使用[^1]。

## 查看全部资源限制

```bash
$ ulimit -a
-t: cpu time (seconds)              unlimited
-f: file size (blocks)              unlimited
-d: data seg size (kbytes)          unlimited
-s: stack size (kbytes)             8192
-c: core file size (blocks)         0
-v: address space (kbytes)          unlimited
-l: locked-in-memory size (kbytes)  unlimited
-u: processes                       2784
-n: file descriptors                256
```

## 查看最多占用 CPU 的时间

```bash
$ ulimit -t
unlimited
```

## 设置 CPU 使用时间上限

```bash
$ ulimit -t 1
```

使用 CPU 密集型计算任务测试：

```bash
// 计算 pi 小数点后 3000 位
$ echo "scale=3000; a(1)*4" | bc -l
zsh: done                echo "scale=3000; a(1)*4" |
zsh: cpu limit exceeded  bc -l
```

[^1]: [Linux ulimit 命令](https://www.runoob.com/linux/linux-comm-ulimit.html)