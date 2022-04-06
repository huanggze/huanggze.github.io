---
title: "Linux 命令：nohup"
date: 2022-04-06T11:42:49+08:00
toc: true
categories: ["Linux"]
series: ["Linux 常用命令"]
---

nohup，no hang up（不挂起），用于在系统后台不挂断地运行命令，即使退出终端不会影响程序的运行[^1]。

nohup 命令，在默认情况下（非重定向时），会输出一个名叫 nohup.out 的文件到当前目录下，如果当前目录的 nohup.out 文件不可写，输出重定向到 $HOME/nohup.out 文件中。

如果要停止运行，需要使 ps 命令找到 nohup 运行脚本的 PID，然后使用 kill 命令来删除。

[^1]: [Linux nohup 命令](https://www.runoob.com/linux/linux-comm-nohup.html)