---
title: "Makefile 语法学习"
date: 2022-05-15T21:10:16+08:00
toc: true
---

## 变量赋值

### 简单赋值（:=）

一旦 make 读入该变量的定义语句，赋值运算符右边部分会立刻扩展，而扩展后的文本会被存储成该变量的值[^1]：

```makefile
x := foo
y := $(x) bar
x := later
```

等同于：

```makefile
x := later
y := foo bar
```

### 递归变量（=）

扩展的动作会被延迟到该变量被使用的时候进行。

```makefile
x = foo
y = $(x) bar
x = later
```

### 条件变量（?=）

只会在变量的值尚不存在是进行赋值操作。

## 条件语句

ifeq：是否相等

```makefile
ifeq (arg1, arg2)
	...
endif
```

ifneq：是否不等

```makefile
ifneq (arg1, arg2)
	...
else
	...
endif
```

ifdef：是否有值

```makefile
ifdef variable-name
	...
endif
```

ifndef：是否未赋值

```makefile
ifndef branch
    br = master
else
    br = ${branch}
endif

demo:
	@echo $(br)
```

```shell
$ make demo
master

$ make demo branch=dev
dev
```

## 在 makefile 中执行 shell

```makefile
CUR_DIR=$(shell pwd)
```

注意，在使用 $ 时，需要写为 $$。因为 $ 在 makefile 中有特殊含义，使用前需要转义。
```makefile
tag=2.0

all:
	@echo $(shell echo ${tag} | awk -F '.' '{print "v"$$1}')
```

## @ 关闭回显 

make 默认会在执行 recipe 打印 recipe 内容（Recipe Echoing）。要关闭回显，使用@ [^2]：

```makefile
demo_a:
	@sh -c "echo hello,"
	@echo "world!"

demo_b:
	sh -c "echo hello,"
	echo "world!"
```

```shell
$ make demo_a
hello,
world!

$ make demo_b
sh -c "echo hello,"
hello,
echo "world!"
world!
```

## 缩进

recipe 前使用 tag，条件语句前使用空格，否则会报错。

```makefile
demo:
	@echo hello, world!
    ifeq (arg1, arg2)
		@echo do something
    endif
```

这里 `ifeq` 和 `endif` 缩进用空格。

[^1]: [makefile 变量](https://segmentfault.com/a/1190000014562776)
[^2]: [Recipe Echoing](https://www.gnu.org/software/make/manual/html_node/Echoing.html)