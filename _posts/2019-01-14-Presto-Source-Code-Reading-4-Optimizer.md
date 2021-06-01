---
layout: post
title: Presto 源码阅读：Optimizer
date: 2019-01-14 23:39 +0800
tags:
- Presto
- RDBMS
- Distributed System
- SQL query optimizer
---


2021 Update:
这篇文章是从我的[知乎专栏](https://www.zhihu.com/column/c_1051437363691147264)搬过来的，作者是同一人。
这几篇文章都是 18 年到 19 年初写的，Presto 社区这几年已经发生了巨大的变化，以下内容仅作参考。

之前说到，Presto 的 LogicalPlanner 会调用一堆 Optimizer 对 PlanTree 进行优化。实际上 Presto 将所有对 PlanTree 的改写逻辑都加到了 planOptimizers 里面。跟 Calcite 相比，Presto 的优化器做的还是挺业余的，参考价值不大。如果不是要往里面加优化器的话，值得看的几条优化规则有 ReorderJoins (影响 Join 的执行顺序)， AddExchanges (影响 SubPlan 的划分) 和 DetermineJoinDistributionType (影响 Join 结点的类型) 。Predict 下沉到数据源的逻辑似乎不是在 Optimize 的过程做的。



Presto 里的优化规则可以在 com.facebook.presto.sql.planner#PlanOptimizers 找到。规则分两类，第一类是在 com.facebook.presto.sql.planner.optimizations 里，都是通过 visitor 模式来对树进行改写；第二类在 com.facebook.presto.sql.planner.iterative.rule 里，这里的规则都是通过 pattern match 来触发。网上看到一个说法，Presto 一开始都是手写 visitor，后面规则一多就写吐了，于是加了个 pattern match 的模块来触发规则，后期的规则就都用 pattern 来触发了。由于 pattern 用得太 high，你可以在 Optimizers 里发现数十个形如 PruneXXXColumns， PushLimitThroughXXX 的优化规则，实际上列裁剪和 limit 下推这种规则还是写 visitor 好一些。

## ReorderJoins
SQL 的优化规则可以分成两种： Join 重排和其他。Join 重排的计算量过于巨大，与其它规则根本不是一个量级，有的数据库甚至使用了遗传算法来计算最优顺序。Presto 目前的 Join 重排算法是基于动态规划的，还设置了超时机制。据我所知，大部分公司对单条 SQL 中 join 的数量都是有限制的，动态规划倒也能解决大部分场景的需求。Join 重排的细节之前的文章已经有过介绍，想了解的可以翻一下之前的文章。

## AddExchange
Presto 中，AddExchangesSubPlan 是根据 Exchange 划分的。因此 AddExchanges 其实是划分 SubPlan 的过程。阅读这部分代码可以帮助我们了解 Presto 的分布式执行逻辑。代码很长，不过基本都是看一下当前结点的种类和meta，根据一定的规则插结点。 AddExchanges 将 optimize 的过程分成了两个阶段，第一阶段大部分都可以在单机上做，是比较传统的规则，包括谓词下推，列消除，Join 重排以及一些简单的结点变换。第二阶段由于 Exchange 结点将执行计划分成了多个阶段，因此又跑了一些通用优化，除此之外还多了一些针对 SubPlan 的优化，比如 PushPartialAggregationThroughJoin 等等。不同优化规则的执行顺序是个很有意思的问题，感兴趣的读者可以看一下 com.facebook.presto.sql.planner$PlanOptimizers，注释还是挺多的，神奇的是作者居然想通过注释将 optimize 的过程分成几个阶段，对此我已经见怪不怪了。

## DetermineJoinDistributionType
这个优化规则决定了 Join 的类型，还有一个 DetermineSemiJoinDistributionType 的规则，前者是典型的 CBO 规则，后者是 RBO 规则，代码都很简洁，也挺重要的。想弄清楚 DetermineJoinDistributionType 的逻辑还是要看 CostProvider 的具体实现。Presto 的 cost 计算机制还是有点复杂的，用到了 cache，计算了子树的 cost， 最终的计算逻辑在 com.facebook.presto.cost$CostCalculatorWithEstimatedExchanges#calculateJoinCost 里。

## 总结
Presto 的 optimizer 有着浓浓的半成品的气息，另外作者是将大部分对 Plan 的改写逻辑丢到 optimizer 里边了。我孤陋寡闻，不知道把 AddExchange 这种划分子图的逻辑当成优化规则是不是通用做法。考虑到针对异构计算节点的优化和一些编译优化，划分子图之后最好单独做一些优化工作，甚至可以将划分后的子图丢到 worker 上做优化。总而言之 Optimizer 是 Presto 相当重要的一个部分，直接决定了各个子计划的形态。

