---
title: "Linux 命令：awk"
date: 2022-05-15T21:57:51+08:00
draft: true
---

## 关系运算符[^1]

```shell
echo 1 1 | awk '$1==$2 {print "相等"}'
```

## 分割

-F 或 --field-separator=fs 指定输入文件的分割符，**默认是空格**。

```shell
echo cat:dog:pig | awk -F ':' '{print $1, $2, $3}'
```

## 终止

exit 执行退出。以下命令只要不输入"1"使等式相等，就会一直等待输入：

```shell
awk '$1==1 {print "end..."; exit}'
```

## 输入源

输入源可以来自echo、文本或用户输入。其中文本文件是一行行读取的。

```shell
# echo
echo hello | awk '{print $1}'

# 文本
awk '{print $1}' input.txt

# 用户输入
awk '{print $1}'
```

[^1]: [awk: Comparison Operators](https://www.gnu.org/software/gawk/manual/html_node/Comparison-Operators.html)

