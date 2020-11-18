---
title: "Iodine: Fast Dynamic Taint Tracking Using Rollback-free Optimistic Hybrid Analysis"
subtitle: "S&P2019 | 污点追踪"
layout: post
author: "Veronica"
header-style: text
tags:
  - paper
  - 污点追踪
  - 程序分析
---

optimistic hybrid analysis

reduce DIFT overhead

It consists of a predicated whole-program static taint analysis 全程序的静态污点分析, which assumes likely invariants不变量 gathered from profiles配置文件 to dramatically improve precision

The optimized DIFT is sound for executions in which those invariants hold true, and recovers to a conservative DIFT for executions in which those invariants are false.优化的DIFT对于那些不变量为真的执行是合理的，对于那些不变量为假的执行恢复到保守的DIFT。

using OHA to optimize live executions —— unbounded rollback无限回滚

我们的工具Iodine可将执行安全策略的DIFT开销降低到9％，这比传统混合分析的开销低4.4倍，同时仍然可以在实时系统上运行。

#### DIFT

它将源数据（例如，敏感的用户输入）标记为污点，通过数据和/或控制流传播污染，并检查污染的数据是否到达接收器（例如，网络输出）。

DIFT性能开销大，在大多数情况下，减速幅度可能高达一到两个数量级。原因是，在纯动态污点跟踪中[12]，必须监控每条指令，以根据源操作数的污点将污点传播到目标操作数。已有的方法：减少污点源，使粒度变粗，但是这些方法会降低准确性



<img src="/Users/veronica/Library/Application Support/typora-user-images/image-20201103211550155.png" alt="image-20201103211550155" style="zoom:50%;" />

**OHA的思想就是静态分析需要把那些实际动态执行时不会执行到的也优化掉。**例如图1(c)中，假设所有执行的结果变量p都是非负数，那么R代码块就肯定不会被执行（这就是一种Invariant）。这种情况下，第2、4、5行的monitor都可以直接删掉，只需要在R里面加上一个Invariant check就可以了。

但是如果本来被认为是不会执行到的代码块R被执行了，就需要把前面的第2行代码认为是不稳定的，因为y来源于s。老式的OHA方法在这种情况下，就会完全重新开始动态分析，由于Invariant不会经常被违背，这种回滚在离线分析情景下是可以接受的，例如调试和取证分析。但是对于在线安全分析来说这是不可以接受的。

Iodine通过前向恢复 (Forward Recovery) 来解决这个问题，完全消除了回滚的需要

之所以需要回滚的原因是，被删掉的monitor与潜在的可能会被违背的Invariant之间的依赖关系。作者的想法是找出Safe elision，这些是不存在依赖关系的。Iodine就是只将证明是Safe elision的monitor找出来删掉。

预测前向分析得到的Elision都是Safe Elision，这个可以从图1-c中的例子里直观的感受到。通过预测前向分析，会扫描每条指令，如果源操作不会被污染，那就认为是Elision。所以例子中预测前向分析只会认为1，5，6为Elision

预测后向分析得到的Elision不一定是Safe Elision。从图1-c的例子中可以看出来，预测后向分析会把2也认为是Elision。

###### 我不懂那个为什么预测前向分析得到的Elision是1，5，6， 预测后向分析得到的Elision是2

