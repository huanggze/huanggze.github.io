---
title: "Into Ginkgo"
date: 2022-02-22T10:48:54+08:00
draft: true
---

phase:
- Tree Construction Phase 
- Run Phase (执行 setup / subject node)

container node
setup node
subject node

suite
spec

container node:
- Describe
- PDescribe/XDescribe
- FDescribe
- Context
- When
- DescribeTable
- PDescribeTable/XDescribeTable
- FDescribeTable

setup node:
- BeforeEach
- JustBeforeEach
- AfterEach
- JustAfterEach
- BeforeSuite
- AfterSuite
- SynchronizedBeforeSuite
- SynchronizedAfterSuite
- BeforeAll
- AfterAll
- ReportAfterEach
- ReportAfterSuite

subject node:
- It
- PIt/XIt
- FIt
- Entry (配合 DescribeTable)
- PEntry / XEntry
- FEntry

non-node/function:
- DeferCleanup (setup / subject node)
- Fail/GinkgoRecover (setup / subject node)
- GinkgoWriter.Println(): Fail 的时候打印，正常隐藏
- By
- GinkgoRandomSeed
- Skip

decorator:
- Serial
- Ordered
- OncePerOrdered (setup node)
- Pending
- Focus
- Label
- FlakeAttempts

gomega:
actual:
- Expect
- Eventually (asynchronous assertions)
assertion:
- To
- NotTo
matcher:
- HaveOccurred
- Equal
- BeFalse
- BeTrue
- BeNil
- BeZero
- BeEmpty
- MatchError
- Succeed
- ConsistOf
- BeNumerically

ginkgo cli
--fail-fast
--randomize-all
-p
-r 递归遍历
--label-filter=<string>
--until-it-fails
--timeout
-v
-vv(very verbose)
ginkgo unfocus 自动取消focus
SMOKETEST_SERVER_ADDR="127.0.0.1:3000" SMOKETEST_ENV="STAGING" ginkgo
ginkgo <GINKGO-FLAGS> <PACKAGES> -- <PASS-THROUGHS>