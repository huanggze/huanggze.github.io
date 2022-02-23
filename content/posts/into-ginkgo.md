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
- Context
- When
- DescribeTable

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
- OncePerOrdered

subject node:
- It
- Entry (配合 DescribeTable)

non-node:
- DeferCleanup (setup / subject node)
- Fail/GinkgoRecover (setup / subject node)
- GinkgoWriter.Println(): Fail 的时候打印，正常隐藏
- By
- GinkgoRandomSeed

decorator:
- Serial
- Ordered

gomega:
actual:
- Expect
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

ginkgo cli
--fail-fast
--randomize-all
-p