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

Ginkgo 使用 Gomega 做测试用例断言。断言放在 Subject 节点中，一个 Subject 节点可包含多个断言。如果一个 Gomega 断言执行失败，则 spec 失败。示例代码如下。其中 `Expect()` 函数接收一个参数（actual value）并返回一个 Assertion 接口实例。`Equal()` 返回一个 GomegaMatcher 接口实例。代码含义是断言 actual value 等于 foo。

```go
Expect("foo").To(Equal("foo"))
```

除了 Expect，还有 Eventually、Consistently 函数可以发起断言。区别是，Eventually 断言如果失败会反复重试直到成功或超时；Consistently 断言则是重复执行直到第一次发生失败才停止。

常见 matcher 及其作用如下：

| Matcher | 描述 |
|---|---|
|Equal|匹配相等|
|HaveOccurred| 是 non-nil error|
|Succeed| 是 nil-error |
|MatchError| 匹配 err.Error() 信息相等，或 error 实例相等 |
|BeTrue|匹配 true|
|BeZero|匹配零值|
|ConsistOf| 两边数组完全匹配 |
|ContainElement| 包含元素 |
|HaveLen|数组、字符串、字典等长度匹配|
|Receive|可以成功从 channel 接收数据，`Eventually(c).Should(Receive(&result))`|
|BeClosed|channel 关闭状态|

## 实践

### 随机化

Ginkgo 默认把 suite 下，不同顶层 Container 随机打散运行；顶层 Container 内的 spec 还是顺序执行，已方便调试。如果希望完全打算所有 spec，使用 `ginkgo --randomize-all` 命令。

### 并行执行

Ginkgo 允许通过命令 `ginkgo -p` 开启使用多核 CPU 并行执行 spec。`GinkgoParallelProcess()` 函数可以返回当前 spec 运行所在 CPU 核编号（1到N）。为了避免多个执行程序间数据竞争（data race），我们需要对访问的依赖数据分片、划分命名空间，使各个并行任务在各自命名空间下操作数据，且还需要统一初始化依赖资源，以避免并行任务重复初始化（比如启动数据库等）。`SynchronizedBeforeSuite` 和 `SynchronizedAfterSuite` 用于在 Setup 节点为各个并行任务执行一次且仅一次初始化或资源回收动作。其底层原理是首先在 process #1 上运行，运行完成后在把结果发给所有 process：

```go
var dbClient *db.Client

var _ = SynchronizedBeforeSuite(func() []byte {
  //runs *only* on process #1
  dbRunner := db.NewRunner()
  Expect(dbRunner.Start()).To(Succeed())
  DeferCleanup(dbRunner.Stop)
  return []byte(dbRunner.Address())
}), func(address []byte) {
  //runs on *all* processes
  dbClient = db.NewClient()
  Expect(dbClient.Connect(string(address))).To(Succeed())
  dbClient.SetNamespace(fmt.Sprintf("namespace-%d", GinkgoParallelProcess()))
  DeferCleanup(dbClient.Cleanup)
})
```

### 使用 DescribeTable

DescribeTable 是 Ginkgo 提供的语法糖，用于表格模板的形式创建 Container 节点。示例代码如下：

```go
Describe("book", func() {
  var book *books.Book

  BeforeEach(func() {
    book = &books.Book{
      Title: "Les Miserables",
      Author: "Victor Hugo",
      Pages: 2783,
    }
    Expect(book.IsValid()).To(BeTrue())
  })

  DescribeTable("Extracting the author's first and last name",
    func(author string, isValid bool, firstName string, lastName string) {
      book.Author = author
      Expect(book.IsValid()).To(Equal(isValid))
      Expect(book.AuthorFirstName()).To(Equal(firstName))
      Expect(book.AuthorLastName()).To(Equal(lastName))
    },
    Entry("When author has both names", "Victor Hugo", true, "Victor", "Hugo"),
    Entry("When author has one name", "Hugo", true, "", "Hugo"),
    Entry("When author has a middle name", "Victor Marie Hugo", true, "Victor", "Hugo"),
    Entry("When author has no name", "", false, "", ""),
  )

})
```

### 使用 DeferCleanup

DeferCleanup 底层会自动生成 AfterEach 节点[^2]，因此可以简化代码，配合 BeforeEach 使用。DeferCleanup() 接收一个函数用于注册清理逻辑，在执行完 spec 后，Ginkgo 会回调注册的清理函数。清理函数可以接收零个或多个参数，示例代码如下：

```go
Describe("Reporting book weight", func() {
  var book *books.Book

  BeforeEach(func() {
    ...
    DeferCleanup(os.Setenv, "WEIGHT_UNITS", os.Getenv("WEIGHT_UNITS"))
  })
  ...
})
```

### 基准测试

Ginkgo 支持 benchmarking。首先创建 Experiment 对象，调用 Sample 方法重复多次执行测试用例并将结果记录到 Experiment 对象中。测试结果也可以保存到文件，以便以后作为基准测试，详见 ExperimentCache [^3]。

```go
// this is our performance spec.  we mark it as Serial to ensure it does not run in
// parallel with other specs (which could affect performance measurements)
// we also label it with "measurement" - this is optional but would allow us to filter out
// measurement-related specs more easily
It("repaginates books efficiently", Serial, Label("measurement"), func() {
	//we create a new experiment
	experiment := gmeasure.NewExperiment("Repaginating Books")

	//Register the experiment as a ReportEntry - this will cause Ginkgo's reporter infrastructure
	//to print out the experiment's report and to include the experiment in any generated reports
	AddReportEntry(experiment.Name, experiment)

	//we sample a function repeatedly to get a statistically significant set of measurements
	experiment.Sample(func(idx int) {
		book = books.LoadFixture("les-miserables.json") //always start with a fresh copy
		book.SetFontSize(10)

		//measure how long it takes to RecomputePages() and store the duration in a "repagination" measurement
		experiment.MeasureDuration("repagination", func() {
			book.RecomputePages()
		})
	}, gmeasure.SamplingConfig{N:20, Duration: time.Minute}) //we'll sample the function up to 20 times or up to a minute, whichever comes first.
})
```

结果输出如下。

```bash
Will run 1 of 1 specs
------------------------------
• [2.029 seconds]
Repaginating Books repaginates books efficiently [measurement]
/path/to/books_test.go:19

  Begin Report Entries >>
  Repaginating Books - /path/to/books_test.go:21 @ 11/04/21 13:42:57.936
    Repaginating Books
    Name          | N  | Min   | Median | Mean  | StdDev | Max
    ==========================================================================
    repagination [duration] | 20 | 5.1ms | 104ms  | 101.4ms | 52.1ms | 196.4ms
  << End Report Entries
```

### Skip 函数

## CLI 工具

ginkgo -r：遍历执行文件下所有 spec

ginkgo --label-filter=\<string\>：执行有指定标签的 spec

ginkgo -v：打印冗余信息

ginkgo -vv：打印更多荣誉信息（very verbose）

ginkgo \<GINKGO-FLAGS\> \<PACKAGES\> -- \<PASS-THROUGHS\>：给 suite 传入命令行参数

```bash
ginkgo -- --server-addr="127.0.0.1:3000" --environment="STAGING"
```

```go
var serverAddr, smokeEnv string

// Register your flags in an init function.  This ensures they are registered _before_ `go test` calls flag.Parse().
func init() {
  flag.StringVar(&serverAddr, "server-addr", "", "Address of the server to smoke-check")
  flag.StringVar(&smokeEnv, "environment", "", "Environment to smoke-check")
}

var client *client.Client
var _ = BeforeSuite(func() {
  // Some basic validations - at this point the flags have been parsed so we can access them
  Expect(serverAddr).NotTo(BeZero(), "Please make sure --server-addr is set correctly.")
  Expect(smokeEnv).To(Or(Equal("PRODUCTION"), Equal("STAGING")), "--environment must be set to PRODUCTION or STAGING.")

  //set up a client 
  client = client.NewClient(serverAddr)
})
```

ginkgo --timeout：设置运行超时

[^1]: [一文讲清楚什么是行为驱动开发](https://www.51cto.com/article/573796.html)
[^2]: [Ginkgo Doc](https://onsi.github.io/ginkgo/#cleaning-up-our-cleanup-code-defercleanup)
[^3]: [Gomega Doc](https://onsi.github.io/gomega/#caching-experiments)