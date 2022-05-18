---
title: "Learn Git"
date: 2022-05-16T21:46:44+08:00
draft: true
toc: true
---

## git rev-list

按时间倒序列出 commit（即 reverse-list）。

假设 commit 提交历史从近到远有 5 条：

```shell
$ git log
E - (HEAD -> master) addE (10/19/2019 13:31:19)
D - addD (10/18/2019 13:31:19)
C - addC (10/17/2019 13:31:19)
B - addB (10/16/2019 13:31:19)
A - addA (10/15/2019 13:31:19)
```

例子[^1]：

`git rev-list D`：打印 D 及更早的 commit 节点

`git rev-list D...B`：打印 D 到 B 之间的节点，且不包括 B

`git rev-list D ^B`：打印 D 及更早的节点，并剔除 B 及更早的节点（同上）

`git rev-list B..D`：打印 D 到 B 之间的节点，且不包括 B（同上）

参数：

\--count：统计结果集中 commit 节点数量

`git rev-list --count HEAD` 统计所有 commit 数量

## git remote

打印远端仓库的信息。

参数：

-v：打印远端仓库详细信息

```shell
$ git remote -v
origin  https://github.com/huanggze/huanggze.github.io.git (fetch)
origin  https://github.com/huanggze/huanggze.github.io.git (push)
```

## git rev-parse

解析 commit，获取 commit id、分支名等信息（rev 即 revision）。

```shell
$ git rev-parse main  
978670a662eb3cb1fe9f89b1e4d166150cea51b7

$ git rev-parse HEAD~2
dfd6dd4c24043a59243fc1e08056afa1cc81929c
```

参数：

\--abbrev-ref <ref>：获取分支名。如下示例获取当前分支名：
\--short：短 commit id

```shell
$ git rev-parse --abbrev-ref HEAD
main

$ git rev-parse --short HEAD
978670a
```

## git log

打印 commit 日志。

参数：

-\<num\>，-n \<num\>：打印指定数量的 commit 日志：

```shell
$ git log -2

Author: xxx <xxx@gmail.com>
Date:   Tue May 10 23:40:54 2022 +0800

    update
    
    Signed-off-by: xxx <xxx@gmail.com>

commit 502bab4fb850f965dc9d4f5d6b53fd57cf587ac2
Merge: dfd6dd4 1166a1b
Author: xxx <xxx@gmail.com>
Date:   Mon May 9 22:57:46 2022 +0800

    Merge branch 'main' of https://github.com/xxx/xxx.github.io
```

\--pretty：输出格式。oneline 用一行现实；format:\<format-string\> 有特殊的输出语法[^2]，如 %B 打印原始 body（包括主题以及主题内容），%s 只打印主题：

```shell
$ git log --pretty=format:%B -1
update

Signed-off-by: xxx <xxx@gmail.com>

$ git log --pretty=format:%s -1
update
```

## git config

git config user.name：打印 git 用户名

[^1]: [Git rev-list 详解](https://blog.csdn.net/Gdeer/article/details/102667263)
[^2]: [git log: pretty formats](https://git-scm.com/docs/git-log#_pretty_formats)