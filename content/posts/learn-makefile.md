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

## 在 recipe 中使用变量

```makefile
recipe: 
	$(eval var=value)
	@echo ${var}
```

如果没有 eval，则不能得到期望结果[^3]：

1. 写成 var=value

`make recipe` 无输出。

```makefile
recipe: 
	var=value
	@echo ${var}
```

该写法等同于[^4]：

```shell
$ sh -c 'var=value'
$ sh -c 'echo ${var}'

```

> The recipe of a rule consists of one or more shell command lines to be executed, one at a time, in the order they appear.

2. 写成 var=value

`make recipe` 会报错：make: var: No such file or directory。

```makefile
recipe: 
	var = value
	@echo ${var}
```

该写法等同于：

```shell
$ sh -c 'var = value'
sh: var: command not found
$ sh -c 'echo ${var}'
```

## .PHONY 意义及用法

我们经常在 makefile 文件中看到类似下面的代码：

```makefile
.PHONY: clean
clean:
	rm *.o temp
```

要讲清楚 .PHONY 的意义及用法，先要理解 makefile 中 rule 的概念。一个 rule 由 target 和 recipe组成：

```makefile
target: prerequisites ...
	recipe
	recipe
	...
```

target 可以是文件名，可执行程序，以及更常见操作名（the name of an action）。而当 target 的含义是操作名的时候，这种 target 需要用 `.PHONY` 来指明是 phony target。

如果不用 .PHONY，当当前目录存在与 target 同名的文件时，make target 不会执行，并提示 make: `xxx' is up to date.

[^1]: [makefile 变量](https://segmentfault.com/a/1190000014562776)
[^2]: [Recipe Echoing](https://www.gnu.org/software/make/manual/html_node/Echoing.html)
[^3]: [Can't assign variable inside recipe](https://stackoverflow.com/questions/6519234/cant-assign-variable-inside-recipe)
[^4]: [5 Writing Recipes in Rules](https://www.gnu.org/software/make/manual/html_node/Recipes.html#Recipes)