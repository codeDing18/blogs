# Worker原理

---

## 接收coordinator请求

Coordinator端发送过来的task都是由TaskResource的/v1/task/{taskId}处理

```java
//TaskResource
@ResourceSecurity(INTERNAL_ONLY)
    @POST
    @Path("{taskId}")
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public void createOrUpdateTask(
            @PathParam("taskId") TaskId taskId,
            TaskUpdateRequest taskUpdateRequest,
            @Context UriInfo uriInfo,
            @Suspended AsyncResponse asyncResponse)
    {
        //获取session
        Session session = taskUpdateRequest.getSession().toSession(sessionPropertyManager, taskUpdateRequest.getExtraCredentials(), taskUpdateRequest.getExchangeEncryptionKey());

        if (injectFailure(session.getTraceToken(), taskId, RequestType.CREATE_OR_UPDATE_TASK, asyncResponse)) {
            return;
        }
				//真正的处理在updateTask中
        TaskInfo taskInfo = taskManager.updateTask(session,
                taskId,
                taskUpdateRequest.getFragment(),
                taskUpdateRequest.getSplitAssignments(),
                taskUpdateRequest.getOutputIds(),
                taskUpdateRequest.getDynamicFilterDomains());

        if (shouldSummarize(uriInfo)) {
            taskInfo = taskInfo.summarize();
        }

        asyncResponse.resume(Response.ok().entity(taskInfo).build());
    }


//SqlTaskManager
public TaskInfo updateTask(
            Session session,
            TaskId taskId,
            Optional<PlanFragment> fragment,
            List<SplitAssignment> splitAssignments,
            OutputBuffers outputBuffers,
            Map<DynamicFilterId, Domain> dynamicFilterDomains)
    {
        try {
            return versionEmbedder.embedVersion(() -> doUpdateTask(session, taskId, fragment, splitAssignments, outputBuffers, dynamicFilterDomains)).call();
        }
        catch (Exception e) {
        }
    }

//SqlTaskManager
private TaskInfo doUpdateTask(
            Session session,
            TaskId taskId,
            Optional<PlanFragment> fragment,
            List<SplitAssignment> splitAssignments,
            OutputBuffers outputBuffers,
            Map<DynamicFilterId, Domain> dynamicFilterDomains)
    {
  			//根据taskId从缓存tasks中获取，如果没有就生成一个sqlTask
        //生成sqlTask最终调用的是SqlTask的createSqlTask方法
        SqlTask sqlTask = tasks.getUnchecked(taskId);
        QueryContext queryContext = sqlTask.getQueryContext();
        if (!queryContext.isMemoryLimitsInitialized()) {
            RetryPolicy retryPolicy = getRetryPolicy(session);
            if (retryPolicy == TASK) {
                // Memory limit for fault tolerant queries should only be enforced by the MemoryPool.
                // LowMemoryKiller is responsible for freeing up the MemoryPool if necessary.
                queryContext.initializeMemoryLimits(false, /* unlimited */ Long.MAX_VALUE);
            }
            else {
                long sessionQueryMaxMemoryPerNode = getQueryMaxMemoryPerNode(session).toBytes();

         // Session properties are only allowed to decrease memory limits, not increase them
                queryContext.initializeMemoryLimits(
                        resourceOvercommit(session),
                        min(sessionQueryMaxMemoryPerNode, queryMaxMemoryPerNode));
            }
        }
  
  			//确保catalog已经加载了
        fragment.ifPresent(planFragment -> connectorServicesProvider.ensureCatalogsLoaded(session, planFragment.getActiveCatalogs()));
				
        sqlTask.recordHeartbeat();
        //在updateTask真正执行
        return sqlTask.updateTask(session, fragment, splitAssignments, outputBuffers, dynamicFilterDomains);
    }

//SqlTask
public TaskInfo updateTask(
            Session session,
            Optional<PlanFragment> fragment,
            List<SplitAssignment> splitAssignments,
            OutputBuffers outputBuffers,
            Map<DynamicFilterId, Domain> dynamicFilterDomains)
    {
        try {
            // trace token must be set first to make sure failure injection for getTaskResults requests works as expected
            session.getTraceToken().ifPresent(traceToken::set);

            // The LazyOutput buffer does not support write methods, so the actual
            // output buffer must be established before drivers are created (e.g.
            // a VALUES query).
            outputBuffer.setOutputBuffers(outputBuffers);

            // assure the task execution is only created once
            SqlTaskExecution taskExecution;
            synchronized (this) {
                // is task already complete?
                TaskHolder taskHolder = taskHolderReference.get();
                if (taskHolder.isFinished()) {
                    return taskHolder.getFinalTaskInfo();
                }
                taskExecution = taskHolder.getTaskExecution();
                if (taskExecution == null) {
                    checkState(fragment.isPresent(), "fragment must be present");
                    //创建taskExecution
                    taskExecution = sqlTaskExecutionFactory.create(
                            session,
                            queryContext,
                            taskStateMachine,
                            outputBuffer,
                            fragment.get(),
                            this::notifyStatusChanged);
                    taskHolderReference.compareAndSet(taskHolder, new TaskHolder(taskExecution));
                    needsPlan.set(false);
                    //taskExecution的执行
                    taskExecution.start();
                }
            }

            taskExecution.addSplitAssignments(splitAssignments);
            taskExecution.getTaskContext().addDynamicFilter(dynamicFilterDomains);
        }
        catch (Error e) {
        }
        catch (RuntimeException e) {
        }

        return getTaskInfo();
    }
```

这里先看下taskExecution是怎么创建的

```java
//SqlTaskExecutionFactory
public SqlTaskExecution create(
            Session session,
            QueryContext queryContext,
            TaskStateMachine taskStateMachine,
            OutputBuffer outputBuffer,
            PlanFragment fragment,
            Runnable notifyStatusChanged)
    {
        TaskContext taskContext = queryContext.addTaskContext(
                taskStateMachine,
                session,
                notifyStatusChanged,
                perOperatorCpuTimerEnabled,
                cpuTimerEnabled);

        LocalExecutionPlan localExecutionPlan;
        try (SetThreadName ignored = new SetThreadName("Task-%s", taskStateMachine.getTaskId())) {
            try {
                //根据传过来的plan fragment生成物理执行计划
                localExecutionPlan = planner.plan(
                        taskContext,
                        fragment.getRoot(),
                        TypeProvider.copyOf(fragment.getSymbols()),
                        fragment.getPartitioningScheme(),
                        fragment.getPartitionedSources(),
                        outputBuffer);
            }
            catch (Throwable e) {       
            }
        }
        return new SqlTaskExecution(
                taskStateMachine,
                taskContext,
                outputBuffer,
                localExecutionPlan,
                taskExecutor,
                splitMonitor,
                taskNotificationExecutor);
    }

//LocalExecutionPlanner
public LocalExecutionPlan plan(
            TaskContext taskContext,
            PlanNode plan,
            TypeProvider types,
            PartitioningScheme partitioningScheme,
            List<PlanNodeId> partitionedSourceOrder,
            OutputBuffer outputBuffer)
    {
        List<Symbol> outputLayout = partitioningScheme.getOutputLayout();

        if (partitioningScheme.getPartitioning().getHandle().equals(FIXED_BROADCAST_DISTRIBUTION) ||
  partitioningScheme.getPartitioning().getHandle().equals(FIXED_ARBITRARY_DISTRIBUTION) ||    partitioningScheme.getPartitioning().getHandle().equals(SCALED_WRITER_ROUND_ROBIN_DISTRIBUTION) || partitioningScheme.getPartitioning().getHandle().equals(SINGLE_DISTRIBUTION) || partitioningScheme.getPartitioning().getHandle().equals(COORDINATOR_DISTRIBUTION)) {
          //
          return plan(taskContext, plan, outputLayout, types, partitionedSourceOrder, new TaskOutputFactory(outputBuffer));
        }

        // We can convert the symbols directly into channels, because the root must be a sink and therefore the layout is fixed
        List<Integer> partitionChannels;
        List<Optional<NullableValue>> partitionConstants;
        List<Type> partitionChannelTypes;
        if (partitioningScheme.getHashColumn().isPresent()) {
            partitionChannels = ImmutableList.of(outputLayout.indexOf(partitioningScheme.getHashColumn().get()));
            partitionConstants = ImmutableList.of(Optional.empty());
            partitionChannelTypes = ImmutableList.of(BIGINT);
        }
        else {
            partitionChannels = partitioningScheme.getPartitioning().getArguments().stream()
                    .map(argument -> {
                        if (argument.isConstant()) {
                            return -1;
                        }
                        return outputLayout.indexOf(argument.getColumn());
                    })
                    .collect(toImmutableList());
            partitionConstants = partitioningScheme.getPartitioning().getArguments().stream()
                    .map(argument -> {
                        if (argument.isConstant()) {
                            return Optional.of(argument.getConstant());
                        }
                        return Optional.<NullableValue>empty();
                    })
                    .collect(toImmutableList());
            partitionChannelTypes = partitioningScheme.getPartitioning().getArguments().stream()
                    .map(argument -> {
                        if (argument.isConstant()) {
                            return argument.getConstant().getType();
                        }
                        return types.get(argument.getColumn());
                    })
                    .collect(toImmutableList());
        }

        PartitionFunction partitionFunction = nodePartitioningManager.getPartitionFunction(taskContext.getSession(), partitioningScheme, partitionChannelTypes);
        OptionalInt nullChannel = OptionalInt.empty();
        Set<Symbol> partitioningColumns = partitioningScheme.getPartitioning().getColumns();

        // partitioningColumns expected to have one column in the normal case, and zero columns when partitioning on a constant
        checkArgument(!partitioningScheme.isReplicateNullsAndAny() || partitioningColumns.size() <= 1);
        if (partitioningScheme.isReplicateNullsAndAny() && partitioningColumns.size() == 1) {
            nullChannel = OptionalInt.of(outputLayout.indexOf(getOnlyElement(partitioningColumns)));
        }

        return plan(
                taskContext,
                plan,
                outputLayout,
                types,
                partitionedSourceOrder,
                new PartitionedOutputFactory(
                        partitionFunction,
                        partitionChannels,
                        partitionConstants,
                        partitioningScheme.isReplicateNullsAndAny(),
                        nullChannel,
                        outputBuffer,
                        maxPagePartitioningBufferSize,
                        positionsAppenderFactory,
                        taskContext.getSession().getExchangeEncryptionKey(),
                        taskContext.newAggregateMemoryContext(),
                        getPagePartitioningBufferPoolSize(taskContext.getSession())));
    }

    public LocalExecutionPlan plan(
            TaskContext taskContext,
            PlanNode plan,
            List<Symbol> outputLayout,
            TypeProvider types,
            List<PlanNodeId> partitionedSourceOrder,
            OutputFactory outputOperatorFactory)
    {
        Session session = taskContext.getSession();
        LocalExecutionPlanContext context = new LocalExecutionPlanContext(taskContext, types);

        PhysicalOperation physicalOperation = plan.accept(new Visitor(session), context);

        Function<Page, Page> pagePreprocessor = enforceLoadedLayoutProcessor(outputLayout, physicalOperation.getLayout());

        List<Type> outputTypes = outputLayout.stream()
                .map(types::get)
                .collect(toImmutableList());

        context.addDriverFactory(
                context.isInputDriver(),
                true,
                new PhysicalOperation(
                        outputOperatorFactory.createOutputOperator(
                                context.getNextOperatorId(),
                                plan.getId(),
                                outputTypes,
                                pagePreprocessor,
                                new PagesSerdeFactory(plannerContext.getBlockEncodingSerde(), isExchangeCompressionEnabled(session))),
                        physicalOperation),
                context.getDriverInstanceCount());

        // notify operator factories that planning has completed
        context.getDriverFactories().stream()
                .map(DriverFactory::getOperatorFactories)
                .flatMap(List::stream)
                .filter(LocalPlannerAware.class::isInstance)
                .map(LocalPlannerAware.class::cast)
                .forEach(LocalPlannerAware::localPlannerComplete);

        return new LocalExecutionPlan(context.getDriverFactories(), partitionedSourceOrder);
    }
```









## 算子执行