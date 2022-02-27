---
title: "Ginkgo 测试框架介绍"
date: 2022-02-25T21:32:07+08:00
toc: true
categories: ["go"]
---

除了 Go testing 包提供的测试框架，还可以使用 Ginkgo 测试框架。Ginkgo 是一个行为驱动开发（Behavior Driven Development，BDD）测试框架。BDD 是一种敏捷开发的技术，建立在测试驱动开发（Test Driven Development，TDD）基础之上，强调使用 DSL（Domain Specific Language，领域特定语言）描述用户行为、定义业务需求，是需求分析人员、开发人员与测试人员进行沟通的有效方法[^1]。行为驱动开发的核心在于"行为"。当业务需求被划分为不同的业务场景，并以 "Given-When-Then" 的形式描述出来时，就形成了一种范式化的领域建模规约。

如下是使用 Ginkgo 测试框架搭建的测试用例，描述的业务场景是根据书本页数（Book.Pages）对书进行分类，小于 300 页应为短篇，大于 300 页应为小说：

```go
var _ = Describe("Books", func() {
  var foxInSocks, lesMis *books.Book

  BeforeEach(func() {
    lesMis = &books.Book{
      Title:  "Les Miserables",
      Author: "Victor Hugo",
      Pages:  2783,
    }

    foxInSocks = &books.Book{
      Title:  "Fox In Socks",
      Author: "Dr. Seuss",
      Pages:  24,
    }
  })

  Describe("Categorizing books", func() {
    Context("with more than 300 pages", func() {
      It("should be a novel", func() {
        Expect(lesMis.Category()).To(Equal(books.CategoryNovel))
      })
    })

    Context("with fewer than 300 pages", func() {
      It("should be a short story", func() {
        Expect(foxInSocks.Category()).To(Equal(books.CategoryShortStory))
      })
    })
  })
})
```

> ginkgo 大量使用了闭包函数。

## 概念

### Spec，Suite

在 Ginkgo 的语境中，spec 是一个个独立的 Ginkgo 测试用例（前面示例代码中 It 函数中的内容），一组 spec 包含在一个 Ginkgo suite 中。Ginkgo suite 是包层级，包名为 *_test，与代码包并列。使用 `ginkgo bootstrap` 命令自动创建 suite_test.go 代码：

```bash
cd path/to/books
ginkgo bootstrap
Generating ginkgo test suite bootstrap for books in:
  books_suite_test.go
```

```go
package books_test

import (
  . "github.com/onsi/ginkgo/v2"
  . "github.com/onsi/gomega"
  "testing"
)

func TestBooks(t *testing.T) {
  RegisterFailHandler(Fail)
  RunSpecs(t, "Books Suite")
}
```

我们可以把测试用例 spec 加到 suit_test.go 中，也可以放到单独的文件，通过 `ginkgo generate` 命令创建测试代码文件 _test.go：

```bash
ginkgo generate book
Generating ginkgo test for Book in:
  book_test.go
```

```go
package books_test

import (
  . "github.com/onsi/ginkgo/v2"
  . "github.com/onsi/gomega"

  "path/to/books"
)

var _ = Describe("Books", func() {

})
```

### Container，Setup，Subject Node

如何编写 Ginkgo 测试用例呢？Ginkgo 通过树形结构构造测试用例树。这个树包含三种节点：Container 节点、Setup 节点、Subject 节点。Container 节点。

Container 节点用于组织多个 spec，Ginkgo 提供 `Describe`、`When`、`Conext` 方法创建 Container 节点。 Subject 节点放置在 Container 节点中，Subject 节点内实现具体测试用例逻辑。Subject 节点与 spec 概念一一对应。一个 Subject 节点即是一个测试用例。Ginkgo 提供 `It` 方法创建 Subject 节点。而 Setup 节点用于为测试用例准备环境和依赖、回收资源等操作。Ginkgo 提供 `BeforeEach`、`AfterEach`、`BeforeSuite`、`AfterSuite` 等函数创建 Setup 节点。

Ginkgo 推荐的最佳实践是 "Declare in container nodes, initialize in setup nodes"，在 Container 节点声明，在 Setup 节点初始化，这样每个 spec 都能拿到最新初始化的对象，否则可能出现不同测试同时读写数据，产生脏数据，导致测试结果不确定。

### Tree Construction Phase，Run Phase

Suite 运行有两个阶段：Tree Construction Phase 和 Run Phase。Tree Construction 阶段，构建测试用例树，如果使用了 Ginkgo 提供的 `DescribeTable` 和 `Entry` 语法糖，那么会在这个阶段被解析到树结构中。Run 阶段执行 Setup 节点和 Subject 节点，Ginkgo 会在运行每个 spec 前，执行 `BeforeEach` 闭包函数；结束后，执行 `AfterEach` 函数。

### Decorator

Spec decorator 用于给测试用例 spec 添加元信息，以调整 spec 在运行阶段的行为。Decorator 作用在 Container 节点和 Subject 节点。以下示例展示了在 Container 节点中使用 `Serial` decorator 来规定 Container 节点下所有 spec 都应串行执行，而不与其他 spec 并行。

```go
Describe("Something expensive", Serial, func() {
  It("is a resource hog that can't run in parallel", func() {
    ...
  })

  It("is another resource hog that can't run in parallel", func() {
    ...
  })
})
```

常用 Decorator 及其功能如下：

| Decorator      | 描述                                                                                     |
|----------------|----------------------------------------------------------------------------------------|
| Serial         | 串行执行 spec。底层实现是 Serial spec 在 suite 中最后执行，且运行在 #1 进程。#1 进程执行 Serial spec 前等待所有其他进程先退出。 |
| Ordered        | 作用于 Container 节点，顺序执行节点下的所有 spec                                                       |                    
| OncePerOrdered | 作用于 Setup 节点，使 `BeforeEach` 等会把 Ordered spec 作为整体处理                                    |
| Pending        | 不执行的测试用例                                                                               |
| Focus          | 被执行的测试用例                                                                               |
| Label          | 给 spec 打标签，`It("is labelled", Label("first label", "second label"), func() { ... })`   |
| FlakeAttempts               | 如果 spec 失败，不会立刻判断失败，而是尝试 N 次（FlakeAttempts(N)）。还是失败，则判定测试用例失败                          |

### Assertion

Ginkgo 使用 gomega 

## 实践

### 

[^1]: [一文讲清楚什么是行为驱动开发](https://www.51cto.com/article/573796.html)