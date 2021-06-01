---
layout: post
title: Presto 源码阅读：Coordinator
date: 2018-12-03 23:39 +0800
tags:
- Presto
- RDBMS
- Distributed System
---


2021 Update:
这篇文章是从我的[知乎专栏](https://www.zhihu.com/column/c_1051437363691147264)搬过来的，作者是同一人。
这几篇文章都是 18 年到 19 年初写的，Presto 社区这几年已经发生了巨大的变化，以下内容仅作参考。

在 Presto 的架构中， Coordinator 负责接收 SQL 查询，将 SQL 语句转换为执行计划并调度 Worker 进行计算。


![SQL 处理流程（这图有点老，某些方法过时了](https://pic4.zhimg.com/80/v2-9f75bf4f8f59ebf023147893db88d66b_1440w.jpg)

SQL 处理的入口在 com.facebook.presto.execution#SqlQueryExecution 里面，不过analyse 的处理逻辑被丢到构造函数里面了。

``` Java
    private SqlQueryExecution(
            String query,
            Session session,
            // ...
            CostCalculator costCalculator,
            WarningCollector warningCollector)
    {
        try (SetThreadName ignored = new SetThreadName("Query-%s", session.getQueryId())) {
            // ...

            // 生成 Analysis
            // analyze query
            requireNonNull(preparedQuery, "preparedQuery is null");
            Analyzer analyzer = new Analyzer(
                    stateMachine.getSession(),
                    metadata,
                    sqlParser,
                    accessControl,
                    Optional.of(queryExplainer),
                    preparedQuery.getParameters(),
                    warningCollector);
            this.analysis = analyzer.analyze(preparedQuery.getStatement());

            stateMachine.setUpdateType(analysis.getUpdateType());

            // ...
        }
    }
```
剩下的逻辑 是从 start 开始的：
``` Java
    @Override
    public void start()
    {
        if (stateMachine.transitionToWaitingForResources()) {
            // 异步处理 query
            waitForMinimumWorkers();
        }
    }

    private void waitForMinimumWorkers()
    {
        ListenableFuture<?> minimumWorkerFuture = clusterSizeMonitor.waitForMinimumWorkers();
        addSuccessCallback(minimumWorkerFuture, this::startExecution);
        addExceptionCallback(minimumWorkerFuture, stateMachine::transitionToFailed);
    }

    private void startExecution()
    {
        try (SetThreadName ignored = new SetThreadName("Query-%s", stateMachine.getQueryId())) {
            try {
                // ...

                // 生成 Logical Plan
                // analyze query
                PlanRoot plan = analyzeQuery();

                metadata.beginQuery(getSession(), plan.getConnectors());

                // 生成 Distribution Plan
                // plan distribution of query
                planDistribution(plan);

                // transition to starting
                if (!stateMachine.transitionToStarting()) {
                    // query already started or finished
                    return;
                }

                // 调度 Worker 进行计算
                // if query is not finished, start the scheduler, otherwise cancel it
                SqlQueryScheduler scheduler = queryScheduler.get();

                // ...
            }
            // ...
        }
    }
```
## Analyse
这部分的逻辑要进 com.facebook.presto.sql.analyzer#Analyzer 看
``` Java
    public Analysis analyze(Statement statement, boolean isDescribe)
    {
        // 这一步是为了处理 EXPLAIN,DESCRIBE INPUT/OUTPUT,SHOW QUERIES/STATS
        Statement rewrittenStatement = StatementRewrite.rewrite(session, metadata, sqlParser, queryExplainer, statement, parameters, accessControl, warningCollector);
        // 这一步会调用 visitor 分析各个字句
        Analysis analysis = new Analysis(rewrittenStatement, parameters, isDescribe);
        StatementAnalyzer analyzer = new StatementAnalyzer(analysis, metadata, sqlParser, accessControl, session, warningCollector);
        analyzer.analyze(rewrittenStatement, Optional.empty());

        // 检查是否有每列的权限
        // check column access permissions for each table
        analysis.getTableColumnReferences().forEach((accessControlInfo, tableColumnReferences) ->
                tableColumnReferences.forEach((tableName, columns) ->
                        accessControlInfo.getAccessControl().checkCanSelectFromColumns(
                                session.getRequiredTransactionId(),
                                accessControlInfo.getIdentity(),
                                tableName,
                                columns)));
        return analysis;
    }
```
在 Analyse 阶段，Coordinator 会根据数据库的 metadata 对 query 进行 validate 并确定每个变量的列信息和函数的输入输出。analyzer.analyze 会调 visitor 对 statement 进行分析，以后再展开。这一步过后会获得一个 Analysis 对象，里面除了 statement 之外还保存了列信息，各字句信息等等。

## Logical Plan
获得 Analysis 之后，Presto 会 LogicalPlanner 生成 Locial Plan。LogicalPlanner 的主要工作就是调用各种优化器对 PlanTree 进行优化，并对生成的 Plan 进行 validate。在优化过程中，优化器会在 Plan 中插入 Exchange 结点。之后 planFragmenter 会根据这些 Exchange 结点将 Plan 切分成 SubPlan。这一步不会把 SubPlan 绑定到具体的 Worker 上。
``` Java
    private PlanRoot doAnalyzeQuery()
    {
        // time analysis phase
        long analysisStart = System.nanoTime();

        // 生成 Logical Plan
        // plan query
        PlanNodeIdAllocator idAllocator = new PlanNodeIdAllocator();
        LogicalPlanner logicalPlanner = new LogicalPlanner(stateMachine.getSession(), planOptimizers, idAllocator, metadata, sqlParser, statsCalculator, costCalculator, stateMachine.getWarningCollector());
        Plan plan = logicalPlanner.plan(analysis);
        queryPlan.set(plan);

        // extract inputs
        List<Input> inputs = new InputExtractor(metadata, stateMachine.getSession()).extractInputs(plan.getRoot());
        stateMachine.setInputs(inputs);

        // extract output
        Optional<Output> output = new OutputExtractor().extractOutput(plan.getRoot());
        stateMachine.setOutput(output);

        // 生成 SubPlan
        // fragment the plan
        SubPlan fragmentedPlan = planFragmenter.createSubPlans(stateMachine.getSession(), metadata, nodePartitioningManager, plan, false);

        // record analysis time
        stateMachine.recordAnalysisTime(analysisStart);

        boolean explainAnalyze = analysis.getStatement() instanceof Explain && ((Explain) analysis.getStatement()).isAnalyze();
        return new PlanRoot(fragmentedPlan, !explainAnalyze, extractConnectors(analysis));
    }
```
LogicalPlanner 的代码：
``` Java
    public Plan plan(Analysis analysis, Stage stage)
    {
        // 根据 Statement 生成原始的 Plan
        PlanNode root = planStatement(analysis, analysis.getStatement());

        planSanityChecker.validateIntermediatePlan(root, session, metadata, sqlParser, symbolAllocator.getTypes(), warningCollector);

        // 调用 PlanOptimize 进行优化
        if (stage.ordinal() >= Stage.OPTIMIZED.ordinal()) {
            for (PlanOptimizer optimizer : planOptimizers) {
                root = optimizer.optimize(root, session, symbolAllocator.getTypes(), symbolAllocator, idAllocator, warningCollector);
                requireNonNull(root, format("%s returned a null plan", optimizer.getClass().getName()));
            }
        }

        // 对生成的 Plan 进行校验
        if (stage.ordinal() >= Stage.OPTIMIZED_AND_VALIDATED.ordinal()) {
            // make sure we produce a valid plan after optimizations run. This is mainly to catch programming errors
            planSanityChecker.validateFinalPlan(root, session, metadata, sqlParser, symbolAllocator.getTypes(), warningCollector);
        }
```
## ExecutionPlan
这一步比较简单，就是分 stage 分 partition。作者还往 planDistribution 里强行塞进来生成 Scheduler 的代码，个人感觉丢到另一个函数比较好。
``` Java
    private void planDistribution(PlanRoot plan)
    {
        // ...

        // plan the execution on the active nodes
        DistributedExecutionPlanner distributedPlanner = new DistributedExecutionPlanner(splitManager);
        StageExecutionPlan outputStageExecutionPlan = distributedPlanner.plan(plan.getRoot(), stateMachine.getSession());
        stateMachine.recordDistributedPlanningTime(distributedPlanningStart);

        // ensure split sources are closed
        stateMachine.addStateChangeListener(state -> {
            if (state.isDone()) {
                closeSplitSources(outputStageExecutionPlan);
            }
        });

        // if query was canceled, skip creating scheduler
        if (stateMachine.isDone()) {
            return;
        }

        // record output field
        stateMachine.setColumns(outputStageExecutionPlan.getFieldNames(), outputStageExecutionPlan.getFragment().getTypes());

        PartitioningHandle partitioningHandle = plan.getRoot().getFragment().getPartitioningScheme().getPartitioning().getHandle();
        OutputBuffers rootOutputBuffers = createInitialEmptyOutputBuffers(partitioningHandle)
                .withBuffer(OUTPUT_BUFFER_ID, BROADCAST_PARTITION_ID)
                .withNoMoreBufferIds();

        // 无法理解为什么要把构造 scheduler 的代码丢这里
        // build the stage execution objects (this doesn't schedule execution)
        SqlQueryScheduler scheduler = new SqlQueryScheduler(
                // ...
                executionPolicy,
                schedulerStats);

        queryScheduler.set(scheduler);

        // if query was canceled during scheduler creation, abort the scheduler
        // directly since the callback may have already fired
        if (stateMachine.isDone()) {
            scheduler.abort();
            queryScheduler.set(null);
        }
    }
```
## Schedule
最后一步就是调度各个 Worker 干活了，任务调度主要代码 com.facebook.presto.execution.scheduler#SqlQueryScheduler 里面。因为 Presto 各 Worker 间是通过拉的方式传输数据的，所以调度的时候要考虑被 block 的情况。这段代码比较长，简化之后是这个样子：

``` Java
    private void schedule()
    {
        try (SetThreadName ignored = new SetThreadName("Query-%s", queryStateMachine.getQueryId())) {
            // ...
            while (!executionSchedule.isFinished()) {
                // ...
                for (SqlStageExecution stage : executionSchedule.getStagesToSchedule()) {
                    // 修改 stage 在状态机里的状态
                    stage.beginScheduling();

                    // 调 Worker 干活
                    // perform some scheduling work
                    ScheduleResult result = stageSchedulers.get(stage.getStageId())
                            .schedule();

                    // modify parent and children based on the results of the scheduling
                    // 如果 result 没吐完就丢到 blockStages 里面
                    if (result.isFinished()) {
                        stage.schedulingComplete();
                    }
                    else if (!result.getBlocked().isDone()) {
                        blockedStages.add(result.getBlocked());
                    }
                    stageLinkages.get(stage.getStageId())
                            .processScheduleResults(stage.getState(), result.getNewTasks());
                    schedulerStats.getSplitsScheduledPerIteration().add(result.getSplitsScheduled());
                    if (result.getBlockedReason().isPresent()) {
                        switch (result.getBlockedReason().get()) {
                            // ...
                        }
                    }
                }

                // make sure to update stage linkage at least once per loop to catch async state changes (e.g., partial cancel)
                // ...

                // wait for a state change and then schedule again
                if (!blockedStages.isEmpty()) {
                    try (TimeStat.BlockTimer timer = schedulerStats.getSleepTime().time()) {
                        tryGetFutureValue(whenAnyComplete(blockedStages), 1, SECONDS);
                    }
                    for (ListenableFuture<?> blockedStage : blockedStages) {
                        blockedStage.cancel(true);
                    }
                }
            }

            // ...
        }
    }
```

## 总结
Presto 中，query 在 Coordinator 中要经过 Analysis, LogicalPlan,ExecutionPlan 这几个阶段，最后被调度到不同的 Worker 上执行。这部分代码挺乱的，甚至还把 Analyse 的逻辑丢到构造函数里了。出于考古的目的我找到了 749de301e33baa2220ac4ac0402d6722725e2503 这个 commit，原来开发者是为了让 validate 不用在线程池排队才将 analyse 提到外面来的。然而这还是没法解释为什么要把 analyse 丢到构造函数，为什么要把 scheduler 丢到 planDistribution。其他类有时候也会出现构造函数有代码逻辑的情况，看代码的时候应该注意一下。

