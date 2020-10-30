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

