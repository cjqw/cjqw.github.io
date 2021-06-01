---
layout: post
title: Presto 源码阅读：ReorderJoins
date: 2018-12-08 23:39 +0800
tags:
- Presto
- RDBMS
- Distributed System
- SQL query optimizer
---


2021 Update:
这篇文章是从我的[知乎专栏](https://www.zhihu.com/column/c_1051437363691147264)搬过来的，作者是同一人。
这几篇文章都是 18 年到 19 年初写的，Presto 社区这几年已经发生了巨大的变化，以下内容仅作参考。

先看一下 Join 重排。Presto 的 Join 重排逻辑是 2018 年中旬加上去的，在此之前开发人员只能手动调整Join顺序，或者使用某公司开发的商业版，所以很多老版本的 Presto 调优的文章都会告诉你一定要注意Join的顺序，要把小表放在哪边这样的trick ：）


回到源代码上来，Join 重排的逻辑主要是在 com.facebook.presto.sql.planner.iterative.rule# ReorderJoins 里面。优化器的入口在 ReorderJoins$apply 里边，函数把整个 Join 树丢到 JoinEnumerator 中，然后递归调用 chooseJoinOrder，直到得到最优解。代码主要逻辑大概长这样：


``` Java
private JoinEnumerationResult chooseJoinOrder(LinkedHashSet<PlanNode> sources, List<Symbol> outputSymbols)
        {
            // 检查是否超时，超时了会退掉，context 的实现在 com.facebook.presto.sql.planner.iterative#IterativeOptimizer
            context.checkTimeoutNotExhausted();

            // 记忆化搜索
            Set<PlanNode> multiJoinKey = ImmutableSet.copyOf(sources);
            JoinEnumerationResult bestResult = memo.get(multiJoinKey);
            if (bestResult == null) {
                checkState(sources.size() > 1, "sources size is less than or equal to one");
                ImmutableList.Builder<JoinEnumerationResult> resultBuilder = ImmutableList.builder();
                // 获得所有划分成两个集合的方式
                Set<Set<Integer>> partitions = generatePartitions(sources.size());
                for (Set<Integer> partition : partitions) {
                    // 获得该 partition 方式的最优解，这里会递归调 chooseJoinOrder
                    JoinEnumerationResult result = createJoinAccordingToPartitioning(sources, outputSymbols, partition);
                    // ...
                    // 处理 corner case
                }
                // 把代价最低的解丢到 memo 里面去
                List<JoinEnumerationResult> results = resultBuilder.build();
                if (results.isEmpty()) {
                    memo.put(multiJoinKey, INFINITE_COST_RESULT);
                    return INFINITE_COST_RESULT;
                }

                bestResult = resultComparator.min(results);
                memo.put(multiJoinKey, bestResult);
            }

            bestResult.planNode.ifPresent((planNode) -> log.debug("Least cost join was: %s", planNode));
            return bestResult;
        }
```

可以看出来，Presto 目前用的是基于 CBO 的动态规划来求最优顺序，搜索的过程中如果超时了就直接退出。求解的过程中还加了一个小优化：


``` Java
        // 这个函数用来划分 partition
        static Set<Set<Integer>> generatePartitions(int totalNodes)
        {
            checkArgument(totalNodes > 1, "totalNodes must be greater than 1");
            Set<Integer> numbers = IntStream.range(0, totalNodes)
                    .boxed()
                    .collect(toImmutableSet());
            return powerSet(numbers).stream()
                    // partition 的时候强制让第一个元素在左边
                    .filter(subSet -> subSet.contains(0))
                    .filter(subSet -> subSet.size() < numbers.size())
                    .collect(toImmutableSet());
        }
        // 这个函数用来计算左节点和右节点 join 的代价
        private JoinEnumerationResult setJoinNodeProperties(JoinNode joinNode)
        {
            // ...
            // 处理 corner case

            List<JoinEnumerationResult> possibleJoinNodes = new ArrayList<>();
            JoinDistributionType joinDistributionType = getJoinDistributionType(session);
            if (joinDistributionType.canPartition() && !joinNode.isCrossJoin()) {
                possibleJoinNodes.add(createJoinEnumerationResult(joinNode.withDistributionType(PARTITIONED)));
                // 这里将左右节点交换再求一次代价，保证上面的优化不会漏掉解
                possibleJoinNodes.add(createJoinEnumerationResult(joinNode.flipChildren().withDistributionType(PARTITIONED)));
            }

            // ...

            return resultComparator.min(possibleJoinNodes);
        }
```

当然，这么小的优化是改变不了动态规划算法巨大的复杂度的：）



## 总结：

1. Presto 的 join 重排是基于动态规划实现的，复杂度为卡特兰数级别
2. join 数量过多时，会超时退出，这时相当于白跑了
3. 接上条，这种机制让之前用户对少量 join 的优化都白做了，用到大量 join 的时候还是要自己调顺序

一点吐槽： Java 玩家写代码真奔放，分个 partition 还要 powerset + filter。

