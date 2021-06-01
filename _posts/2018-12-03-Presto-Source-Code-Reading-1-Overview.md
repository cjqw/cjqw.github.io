---
layout: post
title: Presto 源码阅读： Overview
date: 2018-12-03 23:39 +0800
tags:
- Presto
- RDBMS
- Distributed System
---


2021 Update:
这篇文章是从我的[知乎专栏](https://www.zhihu.com/column/c_1051437363691147264)搬过来的，作者是同一人。
这几篇文章都是 18 年到 19 年初写的，Presto 社区这几年已经发生了巨大的变化，以下内容仅作参考。

Presto 是 Facebook 搞的一个分布式 SQL 执行引擎。它自身不存数据，通过 Connector 从 Hive，Cassandra 等数据源拉数据到 worker 上进行 join，aggregate 等操作。一个 Presto 集群一般长这样：

![img1](https://pic4.zhimg.com/80/v2-308ae963b78bca8b5aada3562b6f8023_1440w.jpg)



其中Presto CLI 是客户端的工具，Coordinator 就是一般说的 master，Coordinator 通过 Hive 等数据源的接口拿到 metadata，对 query 进行处理之后分发到若干 worker 上干事。

一个 query 变成执行计划需要经过这些步骤：


![img2](https://pic2.zhimg.com/80/v2-5a65233d72dc45101f3929264c5f93ad_1440w.jpg)


一条query会先被parse成AST，LogicalPlanner 通过访问数据源接口获得 meta 信息并对 AST 进行优化生成 LogicalPlan，DistributedPlanner 则会根据 optimize 过程中插入的 exchange 结点将 LogicalPlan 分成若干个 SubPlan（怎么变成 distributedplan）。最后由ExecutionPlanner根据DistributedQueryPlan和Discovery Server 给的 worker 信息生成最终的执行计划。

执行计划生成之后分批下发到worker上。Presto 的图执行引擎我还没看，目前知道的是它里面的算子间的数据交互是以拉的方式进行，最小单位是一个最大1MB的page。Worker内部可以对没有数据依赖的算子进行并行计算。另外 Presto 没有 fail over 的机制，查询失败了只能重试。

Presto的优点有：

- 支持 SQL 标准

- 完全基于内存的并行计算，中间结果不需要落盘，速度很快

- 支持多数据源，你还可以手动实现 connector

- 没有 mapreduce里面stage 的概念，并行度更高

- 自身不带数据，扩缩容方便。美团甚至能白天把服务拉起来，晚上把服务关掉。

- 进行了一些工程上的优化，包括动态编译，使用Slice，GC控制等

缺点：

- 吃内存，特别是大表join可能会oom

- 没有fail over机制，没法处理时间较长的 query

- 需要从数据源拉取数据，一方面会带来额外的开销，一方面rt受数据源长尾worker的影响很严重

- 迭代速度很慢，发布五年后才加上join重排优化

目前Presto在国内最大的用户应该是京东和美团，京东还搞了本叫 Presto 技术内幕的书，不过版本很老了。国外有家叫TeraData的公司还搞了一个商业版，不知道好不好使。这个专栏之后看的是4d6999c54211490d3fd0cc5e15afd79c39e8f356 这个commit的版本。



2019 Update：

刚看了下 Presto 的执行引擎，实现的比较粗糙。 page 大小也没有严格的限制，在 MergeOperator, FilterAndProjectOperator 等地用了不同的控制方式，像 RowNumberOperator 这种地方就直接插一列进去了，也没检查页大小。
