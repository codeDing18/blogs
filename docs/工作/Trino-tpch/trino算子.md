# Output

---

## TaskOutputOperator

```java
public class TaskOutputOperator
        implements Operator
{
    public static class TaskOutputFactory
            implements OutputFactory
    {
        private final OutputBuffer outputBuffer;

        public TaskOutputFactory(OutputBuffer outputBuffer)
        {
            this.outputBuffer = requireNonNull(outputBuffer, "outputBuffer is null");
        }

        @Override
        public OperatorFactory createOutputOperator(int operatorId, PlanNodeId planNodeId, List<Type> types, Function<Page, Page> pagePreprocessor, PagesSerdeFactory serdeFactory)
        {
            return new TaskOutputOperatorFactory(operatorId, planNodeId, outputBuffer, pagePreprocessor, serdeFactory);
        }
    }

    public static class TaskOutputOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final OutputBuffer outputBuffer;
        private final Function<Page, Page> pagePreprocessor;
        private final PagesSerdeFactory serdeFactory;

        public TaskOutputOperatorFactory(int operatorId, PlanNodeId planNodeId, OutputBuffer outputBuffer, Function<Page, Page> pagePreprocessor, PagesSerdeFactory serdeFactory)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.outputBuffer = requireNonNull(outputBuffer, "outputBuffer is null");
            this.pagePreprocessor = requireNonNull(pagePreprocessor, "pagePreprocessor is null");
            this.serdeFactory = requireNonNull(serdeFactory, "serdeFactory is null");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, TaskOutputOperator.class.getSimpleName());
            return new TaskOutputOperator(operatorContext, outputBuffer, pagePreprocessor, serdeFactory);
        }

        @Override
        public void noMoreOperators()
        {
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new TaskOutputOperatorFactory(operatorId, planNodeId, outputBuffer, pagePreprocessor, serdeFactory);
        }
    }
    
    //关于算子执行的所有信息
    private final OperatorContext operatorContext;
    //该算子的输出缓存
    private final OutputBuffer outputBuffer;
    //lambda表达式，确保收到的page已经被加载到内存中了
    private final Function<Page, Page> pagePreprocessor;
    //将得到的page序列化成Slice
    private final PageSerializer serializer;
    private ListenableFuture<Void> isBlocked = NOT_BLOCKED;
    private boolean finished;

    public TaskOutputOperator(OperatorContext operatorContext, OutputBuffer outputBuffer, Function<Page, Page> pagePreprocessor, PagesSerdeFactory serdeFactory)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.outputBuffer = requireNonNull(outputBuffer, "outputBuffer is null");
        this.pagePreprocessor = requireNonNull(pagePreprocessor, "pagePreprocessor is null");
        this.serializer = serdeFactory.createSerializer(operatorContext.getSession().getExchangeEncryptionKey().map(Ciphers::deserializeAesEncryptionKey));

        LocalMemoryContext memoryContext = operatorContext.localUserMemoryContext();
        // memory footprint of serializer does not change over time
        memoryContext.setBytes(serializer.getRetainedSizeInBytes());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        finished = true;
    }

    @Override
    public boolean isFinished()
    {
        return finished && isBlocked().isDone();
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        // Avoid re-synchronizing on the output buffer when operator is already blocked
        if (isBlocked.isDone()) {
            isBlocked = outputBuffer.isFull();
            if (isBlocked.isDone()) {
                isBlocked = NOT_BLOCKED;
            }
        }
        return isBlocked;
    }

    @Override
    public boolean needsInput()
    {
        return !finished && isBlocked().isDone();
    }

    //添加其它算子处理好的数据，以page为处理单位
    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");
        if (page.getPositionCount() == 0) {
            return;
        }

        page = pagePreprocessor.apply(page);

        outputBuffer.enqueue(splitAndSerializePage(page));
        operatorContext.recordOutput(page.getSizeInBytes(), page.getPositionCount());
    }

    private List<Slice> splitAndSerializePage(Page page)
    {
        List<Page> split = splitPage(page, DEFAULT_MAX_PAGE_SIZE_IN_BYTES);
        ImmutableList.Builder<Slice> builder = ImmutableList.builderWithExpectedSize(split.size());
        for (Page p : split) {
            builder.add(serializer.serialize(p));
        }
        return builder.build();
    }

    @Override
    public Page getOutput()
    {
        return null;
    }
}
```



## PartitionedOutputOperator

```java
public class PartitionedOutputOperator
        implements Operator
{
    public static class PartitionedOutputFactory
            implements OutputFactory
    {
        private final PartitionFunction partitionFunction;
        private final List<Integer> partitionChannels;
        private final List<Optional<NullableValue>> partitionConstants;
        private final OutputBuffer outputBuffer;
        private final boolean replicatesAnyRow;
        private final OptionalInt nullChannel;
        private final DataSize maxMemory;
        private final PositionsAppenderFactory positionsAppenderFactory;
        private final Optional<Slice> exchangeEncryptionKey;
        private final AggregatedMemoryContext memoryContext;
        private final int pagePartitionerPoolSize;

        public PartitionedOutputFactory(
                PartitionFunction partitionFunction,
                List<Integer> partitionChannels,
                List<Optional<NullableValue>> partitionConstants,
                boolean replicatesAnyRow,
                OptionalInt nullChannel,
                OutputBuffer outputBuffer,
                DataSize maxMemory,
                PositionsAppenderFactory positionsAppenderFactory,
                Optional<Slice> exchangeEncryptionKey,
                AggregatedMemoryContext memoryContext,
                int pagePartitionerPoolSize)
        {
            this.partitionFunction = requireNonNull(partitionFunction, "partitionFunction is null");
            this.partitionChannels = requireNonNull(partitionChannels, "partitionChannels is null");
            this.partitionConstants = requireNonNull(partitionConstants, "partitionConstants is null");
            this.replicatesAnyRow = replicatesAnyRow;
            this.nullChannel = requireNonNull(nullChannel, "nullChannel is null");
            this.outputBuffer = requireNonNull(outputBuffer, "outputBuffer is null");
            this.maxMemory = requireNonNull(maxMemory, "maxMemory is null");
            this.positionsAppenderFactory = requireNonNull(positionsAppenderFactory, "positionsAppenderFactory is null");
            this.exchangeEncryptionKey = requireNonNull(exchangeEncryptionKey, "exchangeEncryptionKey is null");
            this.memoryContext = requireNonNull(memoryContext, "memoryContext is null");
            this.pagePartitionerPoolSize = pagePartitionerPoolSize;
        }

        @Override
        public OperatorFactory createOutputOperator(
                int operatorId,
                PlanNodeId planNodeId,
                List<Type> types,
                Function<Page, Page> pagePreprocessor,
                PagesSerdeFactory serdeFactory)
        {
            return new PartitionedOutputOperatorFactory(
                    operatorId,
                    planNodeId,
                    types,
                    pagePreprocessor,
                    partitionFunction,
                    partitionChannels,
                    partitionConstants,
                    replicatesAnyRow,
                    nullChannel,
                    outputBuffer,
                    serdeFactory,
                    maxMemory,
                    positionsAppenderFactory,
                    exchangeEncryptionKey,
                    memoryContext,
                    pagePartitionerPoolSize);
        }
    }

    public static class PartitionedOutputOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Type> sourceTypes;
        private final Function<Page, Page> pagePreprocessor;
        private final PartitionFunction partitionFunction;
        private final List<Integer> partitionChannels;
        private final List<Optional<NullableValue>> partitionConstants;
        private final boolean replicatesAnyRow;
        private final OptionalInt nullChannel;
        private final OutputBuffer outputBuffer;
        private final PagesSerdeFactory serdeFactory;
        private final DataSize maxMemory;
        private final PositionsAppenderFactory positionsAppenderFactory;
        private final Optional<Slice> exchangeEncryptionKey;
        private final AggregatedMemoryContext memoryContext;
        private final int pagePartitionerPoolSize;
        private final PagePartitionerPool pagePartitionerPool;

        public PartitionedOutputOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                List<Type> sourceTypes,
                Function<Page, Page> pagePreprocessor,
                PartitionFunction partitionFunction,
                List<Integer> partitionChannels,
                List<Optional<NullableValue>> partitionConstants,
                boolean replicatesAnyRow,
                OptionalInt nullChannel,
                OutputBuffer outputBuffer,
                PagesSerdeFactory serdeFactory,
                DataSize maxMemory,
                PositionsAppenderFactory positionsAppenderFactory,
                Optional<Slice> exchangeEncryptionKey,
                AggregatedMemoryContext memoryContext,
                int pagePartitionerPoolSize)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.sourceTypes = requireNonNull(sourceTypes, "sourceTypes is null");
            this.pagePreprocessor = requireNonNull(pagePreprocessor, "pagePreprocessor is null");
            this.partitionFunction = requireNonNull(partitionFunction, "partitionFunction is null");
            this.partitionChannels = requireNonNull(partitionChannels, "partitionChannels is null");
            this.partitionConstants = requireNonNull(partitionConstants, "partitionConstants is null");
            this.replicatesAnyRow = replicatesAnyRow;
            this.nullChannel = requireNonNull(nullChannel, "nullChannel is null");
            this.outputBuffer = requireNonNull(outputBuffer, "outputBuffer is null");
            this.serdeFactory = requireNonNull(serdeFactory, "serdeFactory is null");
            this.maxMemory = requireNonNull(maxMemory, "maxMemory is null");
            this.positionsAppenderFactory = requireNonNull(positionsAppenderFactory, "positionsAppenderFactory is null");
            this.exchangeEncryptionKey = requireNonNull(exchangeEncryptionKey, "exchangeEncryptionKey is null");
            this.memoryContext = requireNonNull(memoryContext, "memoryContext is null");
            this.pagePartitionerPoolSize = pagePartitionerPoolSize;
            this.pagePartitionerPool = new PagePartitionerPool(
                    pagePartitionerPoolSize,
                    () -> new PagePartitioner(
                            partitionFunction,
                            partitionChannels,
                            partitionConstants,
                            replicatesAnyRow,
                            nullChannel,
                            outputBuffer,
                            serdeFactory,
                            sourceTypes,
                            maxMemory,
                            positionsAppenderFactory,
                            exchangeEncryptionKey,
                            memoryContext));
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, PartitionedOutputOperator.class.getSimpleName());
            return new PartitionedOutputOperator(
                    operatorContext,
                    pagePreprocessor,
                    outputBuffer,
                    pagePartitionerPool);
        }

        @Override
        public void noMoreOperators()
        {
            pagePartitionerPool.close();
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new PartitionedOutputOperatorFactory(
                    operatorId,
                    planNodeId,
                    sourceTypes,
                    pagePreprocessor,
                    partitionFunction,
                    partitionChannels,
                    partitionConstants,
                    replicatesAnyRow,
                    nullChannel,
                    outputBuffer,
                    serdeFactory,
                    maxMemory,
                    positionsAppenderFactory,
                    exchangeEncryptionKey,
                    memoryContext,
                    pagePartitionerPoolSize);
        }
    }

    private final OperatorContext operatorContext;
    private final Function<Page, Page> pagePreprocessor;
    private final PagePartitionerPool pagePartitionerPool;
    private final PagePartitioner partitionFunction;
    // outputBuffer is used only to block the operator from finishing if the outputBuffer is full
    private final OutputBuffer outputBuffer;
    private ListenableFuture<Void> isBlocked = NOT_BLOCKED;
    private boolean finished;

    public PartitionedOutputOperator(
            OperatorContext operatorContext,
            Function<Page, Page> pagePreprocessor,
            OutputBuffer outputBuffer,
            PagePartitionerPool pagePartitionerPool)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.pagePreprocessor = requireNonNull(pagePreprocessor, "pagePreprocessor is null");
        this.pagePartitionerPool = requireNonNull(pagePartitionerPool, "pagePartitionerPool is null");
        this.outputBuffer = requireNonNull(outputBuffer, "outputBuffer is null");
        this.partitionFunction = requireNonNull(pagePartitionerPool.poll(), "partitionFunction is null");
        this.partitionFunction.setupOperator(operatorContext);
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        if (!finished) {
            pagePartitionerPool.release(partitionFunction);
            finished = true;
        }
    }

    @Override
    public boolean isFinished()
    {
        return finished && isBlocked().isDone();
    }

    @Override
    public void close()
            throws Exception
    {
        // make sure the operator is finished and partitionFunction released
        finish();
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        // Avoid re-synchronizing on the output buffer when operator is already blocked
        if (isBlocked.isDone()) {
            isBlocked = outputBuffer.isFull();
            if (isBlocked.isDone()) {
                isBlocked = NOT_BLOCKED;
            }
        }
        return isBlocked;
    }

    @Override
    public boolean needsInput()
    {
        return !finished && isBlocked().isDone();
    }

    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");
        checkState(!finished);

        if (page.getPositionCount() == 0) {
            return;
        }

        page = pagePreprocessor.apply(page);
        partitionFunction.partitionPage(page);
    }

    @Override
    public Page getOutput()
    {
        return null;
    }

    public static class PartitionedOutputInfo
            implements Mergeable<PartitionedOutputInfo>, OperatorInfo
    {
        private final long rowsAdded;
        private final long pagesAdded;
        private final long outputBufferPeakMemoryUsage;

        @JsonCreator
        public PartitionedOutputInfo(
                @JsonProperty("rowsAdded") long rowsAdded,
                @JsonProperty("pagesAdded") long pagesAdded,
                @JsonProperty("outputBufferPeakMemoryUsage") long outputBufferPeakMemoryUsage)
        {
            this.rowsAdded = rowsAdded;
            this.pagesAdded = pagesAdded;
            this.outputBufferPeakMemoryUsage = outputBufferPeakMemoryUsage;
        }

        @JsonProperty
        public long getRowsAdded()
        {
            return rowsAdded;
        }

        @JsonProperty
        public long getPagesAdded()
        {
            return pagesAdded;
        }

        @JsonProperty
        public long getOutputBufferPeakMemoryUsage()
        {
            return outputBufferPeakMemoryUsage;
        }

        @Override
        public PartitionedOutputInfo mergeWith(PartitionedOutputInfo other)
        {
            return new PartitionedOutputInfo(
                    rowsAdded + other.rowsAdded,
                    pagesAdded + other.pagesAdded,
                    Math.max(outputBufferPeakMemoryUsage, other.outputBufferPeakMemoryUsage));
        }

        @Override
        public boolean isFinal()
        {
            return true;
        }

        @Override
        public String toString()
        {
            return toStringHelper(this)
                    .add("rowsAdded", rowsAdded)
                    .add("pagesAdded", pagesAdded)
                    .add("outputBufferPeakMemoryUsage", outputBufferPeakMemoryUsage)
                    .toString();
        }
    }
}

```

# exchange

---

## LocalExchangeSourceOperator

```java
public class LocalExchangeSourceOperator
        implements Operator
{
    public static class LocalExchangeSourceOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final LocalExchange localExchange;
        private boolean closed;

        public LocalExchangeSourceOperatorFactory(int operatorId, PlanNodeId planNodeId, LocalExchange localExchange)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.localExchange = requireNonNull(localExchange, "localExchange is null");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");

            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, LocalExchangeSourceOperator.class.getSimpleName());
            return new LocalExchangeSourceOperator(operatorContext, localExchange.getNextSource(driverContext::getPhysicalWrittenDataSize));
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            throw new UnsupportedOperationException("Source operator factories cannot be duplicated");
        }
    }

    private final OperatorContext operatorContext;
    private final LocalExchangeSource source;
    private ListenableFuture<Void> isBlocked = NOT_BLOCKED;

    public LocalExchangeSourceOperator(OperatorContext operatorContext, LocalExchangeSource source)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.source = requireNonNull(source, "source is null");
        operatorContext.setInfoSupplier(source::getBufferInfo);
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        source.finish();
    }

    @Override
    public boolean isFinished()
    {
        return source.isFinished();
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        if (isBlocked.isDone()) {
            isBlocked = source.waitForReading();
            if (isBlocked.isDone()) {
                isBlocked = NOT_BLOCKED;
            }
        }
        return isBlocked;
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException();
    }

    @Override
    public Page getOutput()
    {
        Page page = source.removePage();
        if (page != null) {
            operatorContext.recordProcessedInput(page.getSizeInBytes(), page.getPositionCount());
        }
        return page;
    }

    @Override
    public void close()
    {
        source.close();
    }
}
```



## LocalExchangeSinkOperator

```java
public class LocalExchangeSinkOperator
        implements Operator
{
    public static class LocalExchangeSinkOperatorFactory
            implements OperatorFactory, LocalPlannerAware
    {
        private final int operatorId;
        private final LocalExchangeSinkFactory sinkFactory;
        private final PlanNodeId planNodeId;
        private final Function<Page, Page> pagePreprocessor;
        private boolean closed;

        public LocalExchangeSinkOperatorFactory(LocalExchangeSinkFactory sinkFactory, int operatorId, PlanNodeId planNodeId, Function<Page, Page> pagePreprocessor)
        {
            this.sinkFactory = requireNonNull(sinkFactory, "sinkFactory is null");
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.pagePreprocessor = requireNonNull(pagePreprocessor, "pagePreprocessor is null");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, LocalExchangeSinkOperator.class.getSimpleName());

            return new LocalExchangeSinkOperator(operatorContext, sinkFactory.createSink(), pagePreprocessor);
        }

        @Override
        public void noMoreOperators()
        {
            if (!closed) {
                closed = true;
                sinkFactory.close();
            }
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new LocalExchangeSinkOperatorFactory(sinkFactory.duplicate(), operatorId, planNodeId, pagePreprocessor);
        }

        @Override
        public void localPlannerComplete()
        {
            sinkFactory.noMoreSinkFactories();
        }
    }

    private final OperatorContext operatorContext;
    private final LocalExchangeSink sink;
    private final Function<Page, Page> pagePreprocessor;
    private ListenableFuture<Void> isBlocked = NOT_BLOCKED;

    LocalExchangeSinkOperator(OperatorContext operatorContext, LocalExchangeSink sink, Function<Page, Page> pagePreprocessor)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.sink = requireNonNull(sink, "sink is null");
        this.pagePreprocessor = requireNonNull(pagePreprocessor, "pagePreprocessor is null");
        operatorContext.setFinishedFuture(sink.isFinished());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        sink.finish();
    }

    @Override
    public boolean isFinished()
    {
        return sink.isFinished().isDone();
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        if (isBlocked.isDone()) {
            isBlocked = sink.waitForWriting();
            if (isBlocked.isDone()) {
                isBlocked = NOT_BLOCKED;
            }
        }
        return isBlocked;
    }

    @Override
    public boolean needsInput()
    {
        return !isFinished() && isBlocked().isDone();
    }

    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");
        page = pagePreprocessor.apply(page);
        operatorContext.recordOutput(page.getSizeInBytes(), page.getPositionCount());
        sink.addPage(page);
    }

    @Override
    public Page getOutput()
    {
        return null;
    }

    @Override
    public void close()
    {
        finish();
    }
}
```



## LocalMergeSourceOperator

```java
public class LocalMergeSourceOperator
        implements Operator
{
    public static class LocalMergeSourceOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final LocalExchange localExchange;
        private final List<Type> types;
        private final OrderingCompiler orderingCompiler;
        private final List<Integer> sortChannels;
        private final List<SortOrder> orderings;
        private boolean closed;

        public LocalMergeSourceOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                LocalExchange localExchange,
                List<Type> types,
                OrderingCompiler orderingCompiler,
                List<Integer> sortChannels,
                List<SortOrder> orderings)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.localExchange = requireNonNull(localExchange, "localExchange is null");
            this.types = ImmutableList.copyOf(requireNonNull(types, "types is null"));
            this.orderingCompiler = requireNonNull(orderingCompiler, "orderingCompiler is null");
            this.sortChannels = ImmutableList.copyOf(requireNonNull(sortChannels, "sortChannels is null"));
            this.orderings = ImmutableList.copyOf(requireNonNull(orderings, "orderings is null"));
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");

            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, LocalMergeSourceOperator.class.getSimpleName());
            PageWithPositionComparator comparator = orderingCompiler.compilePageWithPositionComparator(types, sortChannels, orderings);
            List<LocalExchangeSource> sources = IntStream.range(0, localExchange.getBufferCount())
                    .boxed()
                    .map(index -> localExchange.getNextSource(driverContext::getPhysicalWrittenDataSize))
                    .collect(toImmutableList());
            return new LocalMergeSourceOperator(operatorContext, sources, types, comparator);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            throw new UnsupportedOperationException("Source operator factories cannot be duplicated");
        }
    }

    private final OperatorContext operatorContext;
    private final List<LocalExchangeSource> sources;
    private final WorkProcessor<Page> mergedPages;

    public LocalMergeSourceOperator(OperatorContext operatorContext, List<LocalExchangeSource> sources, List<Type> types, PageWithPositionComparator comparator)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.sources = requireNonNull(sources, "sources is null");
        List<WorkProcessor<Page>> pageProducers = sources.stream()
                .map(LocalExchangeSource::pages)
                .collect(toImmutableList());
        mergedPages = mergeSortedPages(
                pageProducers,
                requireNonNull(comparator, "comparator is null"),
                types,
                operatorContext.aggregateUserMemoryContext(),
                operatorContext.getDriverContext().getYieldSignal());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        sources.forEach(LocalExchangeSource::finish);
    }

    @Override
    public boolean isFinished()
    {
        return mergedPages.isFinished();
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        if (mergedPages.isBlocked()) {
            return mergedPages.getBlockedFuture();
        }

        return NOT_BLOCKED;
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException();
    }

    @Override
    public Page getOutput()
    {
        if (!mergedPages.process() || mergedPages.isFinished()) {
            return null;
        }

        Page page = mergedPages.getResult();
        operatorContext.recordProcessedInput(page.getSizeInBytes(), page.getPositionCount());
        return page;
    }

    @Override
    public void close()
    {
        sources.forEach(LocalExchangeSource::close);
    }
}
```

# join

---

## LookupJoinOperator

```java
public class LookupJoinOperator
        implements AdapterWorkProcessorOperator
{
    private final ListenableFuture<LookupSourceProvider> lookupSourceProviderFuture;
    private final boolean waitForBuild;
    private final PageBuffer pageBuffer;
    private final WorkProcessor<Page> pages;
    private final SpillingJoinProcessor joinProcessor;
    private final JoinStatisticsCounter statisticsCounter;

    LookupJoinOperator(
            List<Type> probeTypes,
            List<Type> buildOutputTypes,
            JoinType joinType,
            boolean outputSingleMatch,
            boolean waitForBuild,
            LookupSourceFactory lookupSourceFactory,
            JoinProbeFactory joinProbeFactory,
            Runnable afterClose,
            OptionalInt lookupJoinsCount,
            HashGenerator hashGenerator,
            PartitioningSpillerFactory partitioningSpillerFactory,
            ProcessorContext processorContext,
            Optional<WorkProcessor<Page>> sourcePages)
    {
        this.statisticsCounter = new JoinStatisticsCounter(joinType);
        this.waitForBuild = waitForBuild;
        lookupSourceProviderFuture = lookupSourceFactory.createLookupSourceProvider();
        pageBuffer = new PageBuffer();
        PageJoinerFactory pageJoinerFactory = (lookupSourceProvider, joinerPartitioningSpillerFactory, savedRows) ->
                new DefaultPageJoiner(
                        processorContext,
                        probeTypes,
                        buildOutputTypes,
                        joinType,
                        outputSingleMatch,
                        hashGenerator,
                        joinProbeFactory,
                        lookupSourceFactory,
                        lookupSourceProvider,
                        joinerPartitioningSpillerFactory,
                        statisticsCounter,
                        savedRows);
        joinProcessor = new SpillingJoinProcessor(
                afterClose,
                lookupJoinsCount,
                waitForBuild,
                lookupSourceFactory,
                lookupSourceProviderFuture,
                partitioningSpillerFactory,
                pageJoinerFactory,
                sourcePages.orElse(pageBuffer.pages()));
        pages = flatten(WorkProcessor.create(joinProcessor));
    }

    @Override
    public Optional<OperatorInfo> getOperatorInfo()
    {
        return Optional.of(statisticsCounter.get());
    }

    @Override
    public boolean needsInput()
    {
        return (!waitForBuild || lookupSourceProviderFuture.isDone()) && pageBuffer.isEmpty() && !pageBuffer.isFinished();
    }

    @Override
    public void addInput(Page page)
    {
        pageBuffer.add(page);
    }

    @Override
    public void finish()
    {
        pageBuffer.finish();
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    @Override
    public void close()
    {
        joinProcessor.close();
    }
}

```



## HashBuilderOperator

```java
@ThreadSafe
public class HashBuilderOperator
        implements Operator
{
    private static final Logger log = Logger.get(HashBuilderOperator.class);

    public static class HashBuilderOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final JoinBridgeManager<PartitionedLookupSourceFactory> lookupSourceFactoryManager;
        private final List<Integer> outputChannels;
        private final List<Integer> hashChannels;
        private final OptionalInt preComputedHashChannel;
        private final Optional<JoinFilterFunctionFactory> filterFunctionFactory;
        private final Optional<Integer> sortChannel;
        private final List<JoinFilterFunctionFactory> searchFunctionFactories;
        private final PagesIndex.Factory pagesIndexFactory;

        private final int expectedPositions;
        private final boolean spillEnabled;
        private final SingleStreamSpillerFactory singleStreamSpillerFactory;
        private final HashArraySizeSupplier hashArraySizeSupplier;

        private int partitionIndex;

        private boolean closed;

        public HashBuilderOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                JoinBridgeManager<PartitionedLookupSourceFactory> lookupSourceFactoryManager,
                List<Integer> outputChannels,
                List<Integer> hashChannels,
                OptionalInt preComputedHashChannel,
                Optional<JoinFilterFunctionFactory> filterFunctionFactory,
                Optional<Integer> sortChannel,
                List<JoinFilterFunctionFactory> searchFunctionFactories,
                int expectedPositions,
                PagesIndex.Factory pagesIndexFactory,
                boolean spillEnabled,
                SingleStreamSpillerFactory singleStreamSpillerFactory,
                HashArraySizeSupplier hashArraySizeSupplier)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            requireNonNull(sortChannel, "sortChannel cannot be null");
            requireNonNull(searchFunctionFactories, "searchFunctionFactories is null");
            checkArgument(sortChannel.isPresent() != searchFunctionFactories.isEmpty(), "both or none sortChannel and searchFunctionFactories must be set");
            this.lookupSourceFactoryManager = requireNonNull(lookupSourceFactoryManager, "lookupSourceFactoryManager is null");

            this.outputChannels = ImmutableList.copyOf(requireNonNull(outputChannels, "outputChannels is null"));
            this.hashChannels = ImmutableList.copyOf(requireNonNull(hashChannels, "hashChannels is null"));
            this.preComputedHashChannel = requireNonNull(preComputedHashChannel, "preComputedHashChannel is null");
            this.filterFunctionFactory = requireNonNull(filterFunctionFactory, "filterFunctionFactory is null");
            this.sortChannel = sortChannel;
            this.searchFunctionFactories = ImmutableList.copyOf(searchFunctionFactories);
            this.pagesIndexFactory = requireNonNull(pagesIndexFactory, "pagesIndexFactory is null");
            this.spillEnabled = spillEnabled;
            this.singleStreamSpillerFactory = requireNonNull(singleStreamSpillerFactory, "singleStreamSpillerFactory is null");
            this.hashArraySizeSupplier = requireNonNull(hashArraySizeSupplier, "hashArraySizeSupplier is null");

            this.expectedPositions = expectedPositions;
        }

        @Override
        public HashBuilderOperator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, HashBuilderOperator.class.getSimpleName());

            PartitionedLookupSourceFactory lookupSourceFactory = this.lookupSourceFactoryManager.getJoinBridge();
            verify(partitionIndex < lookupSourceFactory.partitions());
            partitionIndex++;
            return new HashBuilderOperator(
                    operatorContext,
                    lookupSourceFactory,
                    partitionIndex - 1,
                    outputChannels,
                    hashChannels,
                    preComputedHashChannel,
                    filterFunctionFactory,
                    sortChannel,
                    searchFunctionFactories,
                    expectedPositions,
                    pagesIndexFactory,
                    spillEnabled,
                    singleStreamSpillerFactory,
                    hashArraySizeSupplier);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            throw new UnsupportedOperationException("Parallel hash build cannot be duplicated");
        }
    }

    @VisibleForTesting
    public enum State
    {
        /**
         * Operator accepts input
         */
        CONSUMING_INPUT,

        /**
         * Memory revoking occurred during {@link #CONSUMING_INPUT}. Operator accepts input and spills it
         */
        SPILLING_INPUT,

        /**
         * LookupSource has been built and passed on without any spill occurring
         */
        LOOKUP_SOURCE_BUILT,

        /**
         * Input has been finished and spilled
         */
        INPUT_SPILLED,

        /**
         * Spilled input is being unspilled
         */
        INPUT_UNSPILLING,

        /**
         * Spilled input has been unspilled, LookupSource built from it
         */
        INPUT_UNSPILLED_AND_BUILT,

        /**
         * No longer needed
         */
        CLOSED
    }

    private static final double INDEX_COMPACTION_ON_REVOCATION_TARGET = 0.8;

    private final OperatorContext operatorContext;
    private final LocalMemoryContext localUserMemoryContext;
    private final LocalMemoryContext localRevocableMemoryContext;
    private final PartitionedLookupSourceFactory lookupSourceFactory;
    private final ListenableFuture<Void> lookupSourceFactoryDestroyed;
    private final int partitionIndex;

    private final List<Integer> outputChannels;
    private final List<Integer> hashChannels;
    private final OptionalInt preComputedHashChannel;
    private final Optional<JoinFilterFunctionFactory> filterFunctionFactory;
    private final Optional<Integer> sortChannel;
    private final List<JoinFilterFunctionFactory> searchFunctionFactories;

    private final PagesIndex index;
    private final HashArraySizeSupplier hashArraySizeSupplier;

    private final boolean spillEnabled;
    private final SingleStreamSpillerFactory singleStreamSpillerFactory;

    private State state = State.CONSUMING_INPUT;
    private Optional<ListenableFuture<Void>> lookupSourceNotNeeded = Optional.empty();
    private final SpilledLookupSourceHandle spilledLookupSourceHandle = new SpilledLookupSourceHandle();
    private Optional<SingleStreamSpiller> spiller = Optional.empty();
    private ListenableFuture<Void> spillInProgress = NOT_BLOCKED;
    private Optional<ListenableFuture<List<Page>>> unspillInProgress = Optional.empty();
    @Nullable
    private LookupSourceSupplier lookupSourceSupplier;
    private OptionalLong lookupSourceChecksum = OptionalLong.empty();

    private Optional<Runnable> finishMemoryRevoke = Optional.empty();

    public HashBuilderOperator(
            OperatorContext operatorContext,
            PartitionedLookupSourceFactory lookupSourceFactory,
            int partitionIndex,
            List<Integer> outputChannels,
            List<Integer> hashChannels,
            OptionalInt preComputedHashChannel,
            Optional<JoinFilterFunctionFactory> filterFunctionFactory,
            Optional<Integer> sortChannel,
            List<JoinFilterFunctionFactory> searchFunctionFactories,
            int expectedPositions,
            PagesIndex.Factory pagesIndexFactory,
            boolean spillEnabled,
            SingleStreamSpillerFactory singleStreamSpillerFactory,
            HashArraySizeSupplier hashArraySizeSupplier)
    {
        requireNonNull(pagesIndexFactory, "pagesIndexFactory is null");

        this.operatorContext = operatorContext;
        this.partitionIndex = partitionIndex;
        this.filterFunctionFactory = filterFunctionFactory;
        this.sortChannel = sortChannel;
        this.searchFunctionFactories = searchFunctionFactories;
        this.localUserMemoryContext = operatorContext.localUserMemoryContext();
        this.localRevocableMemoryContext = operatorContext.localRevocableMemoryContext();

        this.index = pagesIndexFactory.newPagesIndex(lookupSourceFactory.getTypes(), expectedPositions);
        this.lookupSourceFactory = lookupSourceFactory;
        lookupSourceFactoryDestroyed = lookupSourceFactory.isDestroyed();

        this.outputChannels = outputChannels;
        this.hashChannels = hashChannels;
        this.preComputedHashChannel = preComputedHashChannel;

        this.spillEnabled = spillEnabled;
        this.singleStreamSpillerFactory = requireNonNull(singleStreamSpillerFactory, "singleStreamSpillerFactory is null");
        this.hashArraySizeSupplier = requireNonNull(hashArraySizeSupplier, "hashArraySizeSupplier is null");
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @VisibleForTesting
    public State getState()
    {
        return state;
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        switch (state) {
            case CONSUMING_INPUT:
                return NOT_BLOCKED;

            case SPILLING_INPUT:
                return spillInProgress;

            case LOOKUP_SOURCE_BUILT:
                return lookupSourceNotNeeded.orElseThrow(() -> new IllegalStateException("Lookup source built, but disposal future not set"));

            case INPUT_SPILLED:
                return spilledLookupSourceHandle.getUnspillingOrDisposeRequested();

            case INPUT_UNSPILLING:
                return unspillInProgress.map(HashBuilderOperator::asVoid)
                        .orElseThrow(() -> new IllegalStateException("Unspilling in progress, but unspilling future not set"));

            case INPUT_UNSPILLED_AND_BUILT:
                return spilledLookupSourceHandle.getDisposeRequested();

            case CLOSED:
                return NOT_BLOCKED;
        }
        throw new IllegalStateException("Unhandled state: " + state);
    }

    private static <T> ListenableFuture<Void> asVoid(ListenableFuture<T> future)
    {
        return Futures.transform(future, v -> null, directExecutor());
    }

    @Override
    public boolean needsInput()
    {
        boolean stateNeedsInput = (state == State.CONSUMING_INPUT)
                || (state == State.SPILLING_INPUT && spillInProgress.isDone());

        return stateNeedsInput && !lookupSourceFactoryDestroyed.isDone();
    }

    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");

        if (lookupSourceFactoryDestroyed.isDone()) {
            close();
            return;
        }

        if (state == State.SPILLING_INPUT) {
            spillInput(page);
            return;
        }

        checkState(state == State.CONSUMING_INPUT);
        updateIndex(page);
    }

    private void updateIndex(Page page)
    {
        index.addPage(page);

        if (spillEnabled) {
            localRevocableMemoryContext.setBytes(index.getEstimatedSize().toBytes());
        }
        else {
            if (!localUserMemoryContext.trySetBytes(index.getEstimatedSize().toBytes())) {
                index.compact();
                localUserMemoryContext.setBytes(index.getEstimatedSize().toBytes());
            }
        }
        operatorContext.recordOutput(page.getSizeInBytes(), page.getPositionCount());
    }

    private void spillInput(Page page)
    {
        checkState(spillInProgress.isDone(), "Previous spill still in progress");
        checkSuccess(spillInProgress, "spilling failed");
        spillInProgress = getSpiller().spill(page);
    }

    @Override
    public ListenableFuture<Void> startMemoryRevoke()
    {
        checkState(spillEnabled, "Spill not enabled, no revokable memory should be reserved");

        if (state == State.CONSUMING_INPUT) {
            long indexSizeBeforeCompaction = index.getEstimatedSize().toBytes();
            index.compact();
            long indexSizeAfterCompaction = index.getEstimatedSize().toBytes();
            if (indexSizeAfterCompaction < indexSizeBeforeCompaction * INDEX_COMPACTION_ON_REVOCATION_TARGET) {
                finishMemoryRevoke = Optional.of(() -> {});
                return immediateVoidFuture();
            }

            finishMemoryRevoke = Optional.of(() -> {
                index.clear();
                localUserMemoryContext.setBytes(index.getEstimatedSize().toBytes());
                localRevocableMemoryContext.setBytes(0);
                lookupSourceFactory.setPartitionSpilledLookupSourceHandle(partitionIndex, spilledLookupSourceHandle);
                state = State.SPILLING_INPUT;
            });
            return spillIndex();
        }
        if (state == State.LOOKUP_SOURCE_BUILT) {
            finishMemoryRevoke = Optional.of(() -> {
                lookupSourceFactory.setPartitionSpilledLookupSourceHandle(partitionIndex, spilledLookupSourceHandle);
                lookupSourceNotNeeded = Optional.empty();
                index.clear();
                localUserMemoryContext.setBytes(index.getEstimatedSize().toBytes());
                localRevocableMemoryContext.setBytes(0);
                lookupSourceChecksum = OptionalLong.of(lookupSourceSupplier.checksum());
                lookupSourceSupplier = null;
                state = State.INPUT_SPILLED;
            });
            return spillIndex();
        }
        if (operatorContext.getReservedRevocableBytes() == 0) {
            // Probably stale revoking request
            finishMemoryRevoke = Optional.of(() -> {});
            return immediateVoidFuture();
        }

        throw new IllegalStateException(format("State %s cannot have revocable memory, but has %s revocable bytes", state, operatorContext.getReservedRevocableBytes()));
    }

    private ListenableFuture<Void> spillIndex()
    {
        checkState(spiller.isEmpty(), "Spiller already created");
        spiller = Optional.of(singleStreamSpillerFactory.create(
                index.getTypes(),
                operatorContext.getSpillContext().newLocalSpillContext(),
                operatorContext.newLocalUserMemoryContext(HashBuilderOperator.class.getSimpleName())));
        return getSpiller().spill(index.getPages());
    }

    @Override
    public void finishMemoryRevoke()
    {
        checkState(finishMemoryRevoke.isPresent(), "Cannot finish unknown revoking");
        finishMemoryRevoke.get().run();
        finishMemoryRevoke = Optional.empty();
    }

    @Override
    public Page getOutput()
    {
        return null;
    }

    @Override
    public void finish()
    {
        if (lookupSourceFactoryDestroyed.isDone()) {
            close();
            return;
        }

        if (finishMemoryRevoke.isPresent()) {
            return;
        }

        switch (state) {
            case CONSUMING_INPUT:
                finishInput();
                return;

            case LOOKUP_SOURCE_BUILT:
                disposeLookupSourceIfRequested();
                return;

            case SPILLING_INPUT:
                finishSpilledInput();
                return;

            case INPUT_SPILLED:
                if (spilledLookupSourceHandle.getDisposeRequested().isDone()) {
                    close();
                }
                else {
                    unspillLookupSourceIfRequested();
                }
                return;

            case INPUT_UNSPILLING:
                finishLookupSourceUnspilling();
                return;

            case INPUT_UNSPILLED_AND_BUILT:
                disposeUnspilledLookupSourceIfRequested();
                return;

            case CLOSED:
                // no-op
                return;
        }

        throw new IllegalStateException("Unhandled state: " + state);
    }

    private void finishInput()
    {
        checkState(state == State.CONSUMING_INPUT);
        if (lookupSourceFactoryDestroyed.isDone()) {
            close();
            return;
        }

        LookupSourceSupplier partition = buildLookupSource();
        if (spillEnabled) {
            localRevocableMemoryContext.setBytes(partition.get().getInMemorySizeInBytes());
        }
        else {
            localUserMemoryContext.setBytes(partition.get().getInMemorySizeInBytes());
        }
        lookupSourceNotNeeded = Optional.of(lookupSourceFactory.lendPartitionLookupSource(partitionIndex, partition));

        state = State.LOOKUP_SOURCE_BUILT;
    }

    private void disposeLookupSourceIfRequested()
    {
        checkState(state == State.LOOKUP_SOURCE_BUILT);
        verify(lookupSourceNotNeeded.isPresent());
        if (!lookupSourceNotNeeded.get().isDone()) {
            return;
        }

        index.clear();
        localRevocableMemoryContext.setBytes(0);
        localUserMemoryContext.setBytes(index.getEstimatedSize().toBytes());
        lookupSourceSupplier = null;
        close();
    }

    private void finishSpilledInput()
    {
        checkState(state == State.SPILLING_INPUT);
        if (!spillInProgress.isDone()) {
            // Not ready to handle finish() yet
            return;
        }
        checkSuccess(spillInProgress, "spilling failed");
        state = State.INPUT_SPILLED;
    }

    private void unspillLookupSourceIfRequested()
    {
        checkState(state == State.INPUT_SPILLED);
        if (!spilledLookupSourceHandle.getUnspillingRequested().isDone()) {
            // Nothing to do yet.
            return;
        }

        verify(spiller.isPresent());
        verify(unspillInProgress.isEmpty());

        localUserMemoryContext.setBytes(getSpiller().getSpilledPagesInMemorySize() + index.getEstimatedSize().toBytes());
        unspillInProgress = Optional.of(getSpiller().getAllSpilledPages());

        state = State.INPUT_UNSPILLING;
    }

    private void finishLookupSourceUnspilling()
    {
        checkState(state == State.INPUT_UNSPILLING);
        if (!unspillInProgress.get().isDone()) {
            // Pages have not been unspilled yet.
            return;
        }

        // Use Queue so that Pages already consumed by Index are not retained by us.
        Queue<Page> pages = new ArrayDeque<>(getDone(unspillInProgress.get()));
        unspillInProgress = Optional.empty();
        long sizeOfUnspilledPages = pages.stream()
                .mapToLong(Page::getSizeInBytes)
                .sum();
        long retainedSizeOfUnspilledPages = pages.stream()
                .mapToLong(Page::getRetainedSizeInBytes)
                .sum();
        log.debug(
                "Unspilling for operator %s, unspilled partition %d, sizeOfUnspilledPages %s, retainedSizeOfUnspilledPages %s",
                operatorContext,
                partitionIndex,
                succinctBytes(sizeOfUnspilledPages),
                succinctBytes(retainedSizeOfUnspilledPages));
        localUserMemoryContext.setBytes(retainedSizeOfUnspilledPages + index.getEstimatedSize().toBytes());

        while (!pages.isEmpty()) {
            Page next = pages.remove();
            index.addPage(next);
            // There is no attempt to compact index, since unspilled pages are unlikely to have blocks with retained size > logical size.
            retainedSizeOfUnspilledPages -= next.getRetainedSizeInBytes();
            localUserMemoryContext.setBytes(retainedSizeOfUnspilledPages + index.getEstimatedSize().toBytes());
        }

        LookupSourceSupplier partition = buildLookupSource();
        lookupSourceChecksum.ifPresent(checksum ->
                checkState(partition.checksum() == checksum, "Unspilled lookupSource checksum does not match original one"));
        localUserMemoryContext.setBytes(partition.get().getInMemorySizeInBytes());

        spilledLookupSourceHandle.setLookupSource(partition);

        state = State.INPUT_UNSPILLED_AND_BUILT;
    }

    private void disposeUnspilledLookupSourceIfRequested()
    {
        checkState(state == State.INPUT_UNSPILLED_AND_BUILT);
        if (!spilledLookupSourceHandle.getDisposeRequested().isDone()) {
            return;
        }

        index.clear();
        localUserMemoryContext.setBytes(index.getEstimatedSize().toBytes());

        close();
        spilledLookupSourceHandle.setDisposeCompleted();
    }

    private LookupSourceSupplier buildLookupSource()
    {
        LookupSourceSupplier partition = index.createLookupSourceSupplier(operatorContext.getSession(), hashChannels, preComputedHashChannel, filterFunctionFactory, sortChannel, searchFunctionFactories, Optional.of(outputChannels), hashArraySizeSupplier);
        checkState(lookupSourceSupplier == null, "lookupSourceSupplier is already set");
        this.lookupSourceSupplier = partition;
        return partition;
    }

    @Override
    public boolean isFinished()
    {
        if (lookupSourceFactoryDestroyed.isDone()) {
            // Finish early when the probe side is empty
            close();
            return true;
        }

        return state == State.CLOSED;
    }

    private SingleStreamSpiller getSpiller()
    {
        return spiller.orElseThrow(() -> new IllegalStateException("Spiller not created"));
    }

    @Override
    public void close()
    {
        if (state == State.CLOSED) {
            return;
        }
        // close() can be called in any state, due for example to query failure, and must clean resource up unconditionally

        lookupSourceSupplier = null;
        unspillInProgress = Optional.empty();
        state = State.CLOSED;
        finishMemoryRevoke = finishMemoryRevoke.map(ifPresent -> () -> {});

        try (Closer closer = Closer.create()) {
            closer.register(index::clear);
            spiller.ifPresent(closer::register);
            closer.register(() -> localUserMemoryContext.setBytes(0));
            closer.register(() -> localRevocableMemoryContext.setBytes(0));
        }
        catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```



## NestedLoopJoinOperator

```java
public class NestedLoopJoinOperator
        implements Operator
{
    public static class NestedLoopJoinOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final JoinBridgeManager<NestedLoopJoinBridge> joinBridgeManager;
        private final List<Integer> probeChannels;
        private final List<Integer> buildChannels;
        private boolean closed;

        public NestedLoopJoinOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                JoinBridgeManager<NestedLoopJoinBridge> nestedLoopJoinBridgeManager,
                List<Integer> probeChannels,
                List<Integer> buildChannels)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.joinBridgeManager = nestedLoopJoinBridgeManager;
            joinBridgeManager.incrementProbeFactoryCount();
            this.probeChannels = ImmutableList.copyOf(requireNonNull(probeChannels, "probeChannels is null"));
            this.buildChannels = ImmutableList.copyOf(requireNonNull(buildChannels, "buildChannels is null"));
        }

        private NestedLoopJoinOperatorFactory(NestedLoopJoinOperatorFactory other)
        {
            requireNonNull(other, "other is null");
            this.operatorId = other.operatorId;
            this.planNodeId = other.planNodeId;

            this.joinBridgeManager = other.joinBridgeManager;

            this.probeChannels = ImmutableList.copyOf(other.probeChannels);
            this.buildChannels = ImmutableList.copyOf(other.buildChannels);

            // closed is intentionally not copied
            closed = false;
            joinBridgeManager.incrementProbeFactoryCount();
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            NestedLoopJoinBridge nestedLoopJoinBridge = joinBridgeManager.getJoinBridge();

            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, NestedLoopJoinOperator.class.getSimpleName());

            joinBridgeManager.probeOperatorCreated();
            return new NestedLoopJoinOperator(
                    operatorContext,
                    nestedLoopJoinBridge,
                    probeChannels,
                    buildChannels,
                    () -> joinBridgeManager.probeOperatorClosed());
        }

        @Override
        public void noMoreOperators()
        {
            joinBridgeManager.probeOperatorFactoryClosed();
            if (closed) {
                return;
            }
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new NestedLoopJoinOperatorFactory(this);
        }
    }

    private final ListenableFuture<NestedLoopJoinPages> nestedLoopJoinPagesFuture;
    private final ListenableFuture<Void> blockedFutureView;

    private final OperatorContext operatorContext;
    private final Runnable afterClose;

    private final int[] probeChannels;
    private final int[] buildChannels;
    private List<Page> buildPages;
    private Page probePage;
    private Iterator<Page> buildPageIterator;
    private NestedLoopOutputIterator nestedLoopPageBuilder;
    private boolean finishing;
    private boolean closed;

    private NestedLoopJoinOperator(OperatorContext operatorContext, NestedLoopJoinBridge joinBridge, List<Integer> probeChannels, List<Integer> buildChannels, Runnable afterClose)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.nestedLoopJoinPagesFuture = joinBridge.getPagesFuture();
        blockedFutureView = asVoid(nestedLoopJoinPagesFuture);
        this.probeChannels = Ints.toArray(requireNonNull(probeChannels, "probeChannels is null"));
        this.buildChannels = Ints.toArray(requireNonNull(buildChannels, "buildChannels is null"));
        this.afterClose = requireNonNull(afterClose, "afterClose is null");
    }

    private static <T> ListenableFuture<Void> asVoid(ListenableFuture<T> future)
    {
        return Futures.transform(future, v -> null, directExecutor());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        finishing = true;
    }

    @Override
    public boolean isFinished()
    {
        boolean finished = finishing && probePage == null;

        if (finished) {
            close();
        }
        return finished;
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        return blockedFutureView;
    }

    @Override
    public boolean needsInput()
    {
        if (finishing || probePage != null) {
            return false;
        }

        if (buildPages == null) {
            Optional<NestedLoopJoinPages> nestedLoopJoinPages = tryGetFutureValue(nestedLoopJoinPagesFuture);
            if (nestedLoopJoinPages.isPresent()) {
                buildPages = nestedLoopJoinPages.get().getPages();
            }
        }
        return buildPages != null;
    }

    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");
        checkState(!finishing, "Operator is finishing");
        checkState(buildPages != null, "Page source has not been built yet");
        checkState(probePage == null, "Current page has not been completely processed yet");
        checkState(buildPageIterator == null || !buildPageIterator.hasNext(), "Current buildPageIterator has not been completely processed yet");

        if (page.getPositionCount() > 0) {
            probePage = page;
            buildPageIterator = buildPages.iterator();
        }
    }

    @Override
    public Page getOutput()
    {
        // Either probe side or build side is not ready
        if (probePage == null || buildPages == null) {
            return null;
        }

        if (nestedLoopPageBuilder != null && nestedLoopPageBuilder.hasNext()) {
            return nestedLoopPageBuilder.next();
        }

        if (buildPageIterator.hasNext()) {
            nestedLoopPageBuilder = createNestedLoopOutputIterator(probePage, buildPageIterator.next(), probeChannels, buildChannels);
            return nestedLoopPageBuilder.next();
        }

        probePage = null;
        nestedLoopPageBuilder = null;
        return null;
    }

    @Override
    public void close()
    {
        buildPages = null;
        probePage = null;
        nestedLoopPageBuilder = null;
        buildPageIterator = null;
        // We don't want to release the supplier multiple times, since its reference counted
        if (closed) {
            return;
        }
        closed = true;
        // `afterClose` must be run last.
        afterClose.run();
    }

    @VisibleForTesting
    static NestedLoopOutputIterator createNestedLoopOutputIterator(Page probePage, Page buildPage, int[] probeChannels, int[] buildChannels)
    {
        if (probeChannels.length == 0 && buildChannels.length == 0) {
            int probePositions = probePage.getPositionCount();
            int buildPositions = buildPage.getPositionCount();
            try {
                // positionCount is an int. Make sure the product can still fit in an int.
                int outputPositions = multiplyExact(probePositions, buildPositions);
                if (outputPositions <= PageProcessor.MAX_BATCH_SIZE) {
                    return new PageRepeatingIterator(new Page(outputPositions), 1);
                }
            }
            catch (ArithmeticException overflow) {
            }
            // Repeat larger position count a smaller position count number of times
            Page outputPage = new Page(max(probePositions, buildPositions));
            return new PageRepeatingIterator(outputPage, min(probePositions, buildPositions));
        }
        if (probeChannels.length == 0 && probePage.getPositionCount() <= buildPage.getPositionCount()) {
            return new PageRepeatingIterator(buildPage.getColumns(buildChannels), probePage.getPositionCount());
        }
        if (buildChannels.length == 0 && buildPage.getPositionCount() <= probePage.getPositionCount()) {
            return new PageRepeatingIterator(probePage.getColumns(probeChannels), buildPage.getPositionCount());
        }
        return new NestedLoopPageBuilder(probePage, buildPage, probeChannels, buildChannels);
    }

    // bi-morphic parent class for the two implementations allowed. Adding a third implementation will make getOutput megamorphic and
    // should be avoided
    @VisibleForTesting
    abstract static sealed class NestedLoopOutputIterator
            permits PageRepeatingIterator, NestedLoopPageBuilder
    {
        public abstract boolean hasNext();

        public abstract Page next();
    }

    private static final class PageRepeatingIterator
            extends NestedLoopOutputIterator
    {
        private final Page page;
        private int remainingCount;

        private PageRepeatingIterator(Page page, int repetitions)
        {
            this.page = requireNonNull(page, "page is null");
            this.remainingCount = repetitions;
        }

        @Override
        public boolean hasNext()
        {
            return remainingCount > 0;
        }

        @Override
        public Page next()
        {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            remainingCount--;
            return page;
        }
    }

    /**
     * This class takes one probe page(p rows) and one build page(b rows) and
     * build n pages with m rows in each page, where n = min(p, b) and m = max(p, b)
     */
    private static final class NestedLoopPageBuilder
            extends NestedLoopOutputIterator
    {
        //  Avoids allocation a new block array per iteration
        private final Block[] resultBlockBuffer;
        private final Block[] smallPageOutputBlocks;
        private final int indexForRleBlocks;
        private final int largePagePositionCount;
        private final int maxRowIndex; // number of rows - 1

        private int rowIndex = -1; // Iterator on the rows in the page with less rows.

        NestedLoopPageBuilder(Page probePage, Page buildPage, int[] probeChannels, int[] buildChannels)
        {
            requireNonNull(probePage, "probePage is null");
            checkArgument(probePage.getPositionCount() > 0, "probePage has no rows");
            requireNonNull(buildPage, "buildPage is null");
            checkArgument(buildPage.getPositionCount() > 0, "buildPage has no rows");

            Page largePage;
            Page smallPage;
            int indexForPageBlocks;
            int[] channelsForSmallPage;
            int[] channelsForLargePage;
            if (buildPage.getPositionCount() > probePage.getPositionCount()) {
                largePage = buildPage;
                smallPage = probePage;
                channelsForLargePage = buildChannels;
                channelsForSmallPage = probeChannels;
                indexForPageBlocks = probeChannels.length;
                this.indexForRleBlocks = 0;
            }
            else {
                largePage = probePage;
                smallPage = buildPage;
                channelsForLargePage = probeChannels;
                channelsForSmallPage = buildChannels;
                indexForPageBlocks = 0;
                this.indexForRleBlocks = probeChannels.length;
            }
            this.largePagePositionCount = largePage.getPositionCount();
            this.maxRowIndex = smallPage.getPositionCount() - 1;

            this.resultBlockBuffer = new Block[channelsForLargePage.length + channelsForSmallPage.length];
            // Put the blocks from the page with more rows in the output buffer in order
            for (int i = 0; i < channelsForLargePage.length; i++) {
                resultBlockBuffer[indexForPageBlocks + i] = largePage.getBlock(channelsForLargePage[i]);
            }
            // Extract the small page output blocks in order
            this.smallPageOutputBlocks = new Block[channelsForSmallPage.length];
            for (int i = 0; i < channelsForSmallPage.length; i++) {
                this.smallPageOutputBlocks[i] = smallPage.getBlock(channelsForSmallPage[i]);
            }
        }

        @Override
        public boolean hasNext()
        {
            return rowIndex < maxRowIndex;
        }

        @Override
        public Page next()
        {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }

            rowIndex++;

            // For the page with less rows, create RLE blocks and add them to the blocks array
            for (int i = 0; i < smallPageOutputBlocks.length; i++) {
                Block block = smallPageOutputBlocks[i].getSingleValueBlock(rowIndex);
                resultBlockBuffer[indexForRleBlocks + i] = RunLengthEncodedBlock.create(block, largePagePositionCount);
            }
            // Page constructor will create a copy of the block buffer (and must for correctness)
            return new Page(largePagePositionCount, resultBlockBuffer);
        }
    }
}
```



## NestedLoopBuildOperator

```java
public class NestedLoopBuildOperator
        implements Operator
{
    public static class NestedLoopBuildOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final JoinBridgeManager<NestedLoopJoinBridge> nestedLoopJoinBridgeManager;

        private boolean closed;

        public NestedLoopBuildOperatorFactory(int operatorId, PlanNodeId planNodeId, JoinBridgeManager<NestedLoopJoinBridge> nestedLoopJoinBridgeManager)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.nestedLoopJoinBridgeManager = requireNonNull(nestedLoopJoinBridgeManager, "nestedLoopJoinBridgeManager is null");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, NestedLoopBuildOperator.class.getSimpleName());
            return new NestedLoopBuildOperator(operatorContext, nestedLoopJoinBridgeManager.getJoinBridge());
        }

        @Override
        public void noMoreOperators()
        {
            if (closed) {
                return;
            }
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new NestedLoopBuildOperatorFactory(operatorId, planNodeId, nestedLoopJoinBridgeManager);
        }
    }

    private final OperatorContext operatorContext;
    private final NestedLoopJoinBridge nestedLoopJoinBridge;
    private final NestedLoopJoinPagesBuilder nestedLoopJoinPagesBuilder;
    private final LocalMemoryContext localUserMemoryContext;

    // Initially, probeDoneWithPages is not present.
    // Once finish is called, probeDoneWithPages will be set to a future that completes when the pages are no longer needed by the probe side.
    // When the pages are no longer needed, the isFinished method on this operator will return true.
    private Optional<ListenableFuture<Void>> probeDoneWithPages = Optional.empty();

    public NestedLoopBuildOperator(OperatorContext operatorContext, NestedLoopJoinBridge nestedLoopJoinBridge)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.nestedLoopJoinBridge = requireNonNull(nestedLoopJoinBridge, "nestedLoopJoinBridge is null");
        this.nestedLoopJoinPagesBuilder = new NestedLoopJoinPagesBuilder(operatorContext);
        this.localUserMemoryContext = operatorContext.localUserMemoryContext();
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        if (probeDoneWithPages.isPresent()) {
            return;
        }

        // nestedLoopJoinPagesBuilder and the built NestedLoopJoinPages will mostly share the same objects.
        // Extra allocation is minimal during build call. As a result, memory accounting is not updated here.
        probeDoneWithPages = Optional.of(nestedLoopJoinBridge.setPages(nestedLoopJoinPagesBuilder.build()));
    }

    @Override
    public boolean isFinished()
    {
        return probeDoneWithPages.map(Future::isDone).orElse(false);
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        return probeDoneWithPages.orElse(NOT_BLOCKED);
    }

    @Override
    public boolean needsInput()
    {
        return probeDoneWithPages.isEmpty();
    }

    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");
        checkState(!isFinished(), "Operator is already finished");

        if (page.getPositionCount() == 0) {
            return;
        }

        nestedLoopJoinPagesBuilder.addPage(page);
        if (!localUserMemoryContext.trySetBytes(nestedLoopJoinPagesBuilder.getEstimatedSize().toBytes())) {
            nestedLoopJoinPagesBuilder.compact();
            localUserMemoryContext.setBytes(nestedLoopJoinPagesBuilder.getEstimatedSize().toBytes());
        }
        operatorContext.recordOutput(page.getSizeInBytes(), page.getPositionCount());
    }

    @Override
    public Page getOutput()
    {
        return null;
    }
}
```



## LookupOuterOperator

```java
public class LookupOuterOperator
        implements Operator
{
    public static class LookupOuterOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Type> probeOutputTypes;
        private final List<Type> buildOutputTypes;
        private final JoinBridgeManager<?> joinBridgeManager;

        private boolean closed;

        public LookupOuterOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                List<Type> probeOutputTypes,
                List<Type> buildOutputTypes,
                JoinBridgeManager<?> joinBridgeManager)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.probeOutputTypes = ImmutableList.copyOf(requireNonNull(probeOutputTypes, "probeOutputTypes is null"));
            this.buildOutputTypes = ImmutableList.copyOf(requireNonNull(buildOutputTypes, "buildOutputTypes is null"));
            this.joinBridgeManager = joinBridgeManager;
        }

        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "LookupOuterOperatorFactory is closed");

            ListenableFuture<OuterPositionIterator> outerPositionsFuture = joinBridgeManager.getOuterPositionsFuture();
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, LookupOuterOperator.class.getSimpleName());
            joinBridgeManager.outerOperatorCreated();
            return new LookupOuterOperator(operatorContext, outerPositionsFuture, probeOutputTypes, buildOutputTypes, () -> joinBridgeManager.outerOperatorClosed());
        }

        @Override
        public void noMoreOperators()
        {
            joinBridgeManager.outerOperatorFactoryClosed();
            if (closed) {
                return;
            }
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            throw new UnsupportedOperationException("Source operator factories cannot be duplicated");
        }
    }

    private final OperatorContext operatorContext;
    private final ListenableFuture<OuterPositionIterator> outerPositionsFuture;
    private final ListenableFuture<Void> blockedFutureView;

    private final List<Type> probeOutputTypes;
    private final Runnable onClose;

    private final PageBuilder pageBuilder;

    private OuterPositionIterator outerPositions;
    private boolean closed;

    public LookupOuterOperator(
            OperatorContext operatorContext,
            ListenableFuture<OuterPositionIterator> outerPositionsFuture,
            List<Type> probeOutputTypes,
            List<Type> buildOutputTypes,
            Runnable onClose)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.outerPositionsFuture = requireNonNull(outerPositionsFuture, "outerPositionsFuture is null");
        blockedFutureView = asVoid(outerPositionsFuture);

        List<Type> types = ImmutableList.<Type>builder()
                .addAll(requireNonNull(probeOutputTypes, "probeOutputTypes is null"))
                .addAll(requireNonNull(buildOutputTypes, "buildOutputTypes is null"))
                .build();
        this.probeOutputTypes = ImmutableList.copyOf(probeOutputTypes);
        this.pageBuilder = new PageBuilder(types);
        this.onClose = requireNonNull(onClose, "onClose is null");
    }

    private static <T> ListenableFuture<Void> asVoid(ListenableFuture<T> future)
    {
        return Futures.transform(future, v -> null, directExecutor());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        return blockedFutureView;
    }

    @Override
    public void finish()
    {
        // this is a source operator, so we can just terminate the output now
        close();
    }

    @Override
    public boolean isFinished()
    {
        return closed;
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException();
    }

    @Override
    public Page getOutput()
    {
        if (outerPositions == null) {
            outerPositions = tryGetFutureValue(outerPositionsFuture).orElse(null);
            if (outerPositions == null) {
                return null;
            }
        }

        boolean outputPositionsFinished = false;
        while (!pageBuilder.isFull()) {
            // write build columns
            outputPositionsFinished = !outerPositions.appendToNext(pageBuilder, probeOutputTypes.size());
            if (outputPositionsFinished) {
                break;
            }
            pageBuilder.declarePosition();

            // write nulls into probe columns
            // todo use RLE blocks
            for (int probeChannel = 0; probeChannel < probeOutputTypes.size(); probeChannel++) {
                pageBuilder.getBlockBuilder(probeChannel).appendNull();
            }
        }

        // only flush full pages unless we are done
        Page page = null;
        if (pageBuilder.isFull() || (outputPositionsFinished && !pageBuilder.isEmpty())) {
            page = pageBuilder.build();
            pageBuilder.reset();
        }

        if (outputPositionsFinished) {
            close();
        }
        return page;
    }

    @Override
    public void close()
    {
        if (closed) {
            return;
        }
        closed = true;
        pageBuilder.reset();
        onClose.run();
    }
}
```

# others

---

## FilterAndProjectOperator

```java
public class FilterAndProjectOperator
        implements WorkProcessorOperator
{
    private final WorkProcessor<Page> pages;
    private final PageProcessorMetrics metrics = new PageProcessorMetrics();

    private FilterAndProjectOperator(
            Session session,
            MemoryTrackingContext memoryTrackingContext,
            DriverYieldSignal yieldSignal,
            WorkProcessor<Page> sourcePages,
            PageProcessor pageProcessor,
            List<Type> types,
            DataSize minOutputPageSize,
            int minOutputPageRowCount,
            boolean avoidPageMaterialization)
    {
        AggregatedMemoryContext localAggregatedMemoryContext = newSimpleAggregatedMemoryContext();
        LocalMemoryContext outputMemoryContext = localAggregatedMemoryContext.newLocalMemoryContext(FilterAndProjectOperator.class.getSimpleName());
        ConnectorSession connectorSession = session.toConnectorSession();

        this.pages = sourcePages
                .flatMap(page -> pageProcessor.createWorkProcessor(
                        connectorSession,
                        yieldSignal,
                        outputMemoryContext,
                        metrics,
                        page,
                        avoidPageMaterialization))
                .transformProcessor(processor -> mergePages(types, minOutputPageSize.toBytes(), minOutputPageRowCount, processor, localAggregatedMemoryContext))
                .blocking(() -> memoryTrackingContext.localUserMemoryContext().setBytes(localAggregatedMemoryContext.getBytes()));
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    @Override
    public Metrics getMetrics()
    {
        return metrics.getMetrics();
    }

    public static OperatorFactory createOperatorFactory(
            int operatorId,
            PlanNodeId planNodeId,
            Supplier<PageProcessor> processor,
            List<Type> types,
            DataSize minOutputPageSize,
            int minOutputPageRowCount)
    {
        return createAdapterOperatorFactory(new Factory(
                operatorId,
                planNodeId,
                processor,
                types,
                minOutputPageSize,
                minOutputPageRowCount));
    }

    private static class Factory
            implements BasicAdapterWorkProcessorOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final Supplier<PageProcessor> processor;
        private final List<Type> types;
        private final DataSize minOutputPageSize;
        private final int minOutputPageRowCount;
        private boolean closed;

        private Factory(
                int operatorId,
                PlanNodeId planNodeId,
                Supplier<PageProcessor> processor,
                List<Type> types,
                DataSize minOutputPageSize,
                int minOutputPageRowCount)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.processor = requireNonNull(processor, "processor is null");
            this.types = ImmutableList.copyOf(requireNonNull(types, "types is null"));
            this.minOutputPageSize = requireNonNull(minOutputPageSize, "minOutputPageSize is null");
            this.minOutputPageRowCount = minOutputPageRowCount;
        }

        @Override
        public WorkProcessorOperator create(ProcessorContext processorContext, WorkProcessor<Page> sourcePages)
        {
            checkState(!closed, "Factory is already closed");
            return new FilterAndProjectOperator(
                    processorContext.getSession(),
                    processorContext.getMemoryTrackingContext(),
                    processorContext.getDriverYieldSignal(),
                    sourcePages,
                    processor.get(),
                    types,
                    minOutputPageSize,
                    minOutputPageRowCount,
                    true);
        }

        @Override
        public WorkProcessorOperator createAdapterOperator(ProcessorContext processorContext, WorkProcessor<Page> sourcePages)
        {
            checkState(!closed, "Factory is already closed");
            return new FilterAndProjectOperator(
                    processorContext.getSession(),
                    processorContext.getMemoryTrackingContext(),
                    processorContext.getDriverYieldSignal(),
                    sourcePages,
                    processor.get(),
                    types,
                    minOutputPageSize,
                    minOutputPageRowCount,
                    false);
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return planNodeId;
        }

        @Override
        public String getOperatorType()
        {
            return FilterAndProjectOperator.class.getSimpleName();
        }

        @Override
        public void close()
        {
            closed = true;
        }

        @Override
        public BasicAdapterWorkProcessorOperatorFactory duplicate()
        {
            return new Factory(operatorId, planNodeId, processor, types, minOutputPageSize, minOutputPageRowCount);
        }
    }
}
```



## ScanFilterAndProjectOperator

```java
public class ScanFilterAndProjectOperator
        implements WorkProcessorSourceOperator
{
    private final WorkProcessor<Page> pages;
    private final PageProcessorMetrics pageProcessorMetrics = new PageProcessorMetrics();

    @Nullable
    private RecordCursor cursor;
    @Nullable
    private ConnectorPageSource pageSource;

    private long processedPositions;
    private long processedBytes;
    private long physicalBytes;
    private long physicalPositions;
    private long readTimeNanos;
    private long dynamicFilterSplitsProcessed;
    private Metrics metrics = Metrics.EMPTY;

    private ScanFilterAndProjectOperator(
            Session session,
            MemoryTrackingContext memoryTrackingContext,
            DriverYieldSignal yieldSignal,
            WorkProcessor<Split> splits,
            PageSourceProvider pageSourceProvider,
            CursorProcessor cursorProcessor,
            PageProcessor pageProcessor,
            TableHandle table,
            Iterable<ColumnHandle> columns,
            DynamicFilter dynamicFilter,
            Iterable<Type> types,
            DataSize minOutputPageSize,
            int minOutputPageRowCount,
            boolean avoidPageMaterialization)
    {
        pages = splits.flatTransform(
                new SplitToPages(
                        session,
                        yieldSignal,
                        pageSourceProvider,
                        cursorProcessor,
                        pageProcessor,
                        table,
                        columns,
                        dynamicFilter,
                        types,
                        memoryTrackingContext.aggregateUserMemoryContext(),
                        minOutputPageSize,
                        minOutputPageRowCount,
                        avoidPageMaterialization));
    }

    @Override
    public DataSize getPhysicalInputDataSize()
    {
        return DataSize.ofBytes(physicalBytes);
    }

    @Override
    public long getPhysicalInputPositions()
    {
        return physicalPositions;
    }

    @Override
    public DataSize getInputDataSize()
    {
        return DataSize.ofBytes(processedBytes);
    }

    @Override
    public long getInputPositions()
    {
        return processedPositions;
    }

    @Override
    public Duration getReadTime()
    {
        return new Duration(readTimeNanos, NANOSECONDS);
    }

    @Override
    public long getDynamicFilterSplitsProcessed()
    {
        return dynamicFilterSplitsProcessed;
    }

    @Override
    public Metrics getConnectorMetrics()
    {
        return metrics;
    }

    @Override
    public Metrics getMetrics()
    {
        if (cursor != null) {
            return Metrics.EMPTY;
        }
        return pageProcessorMetrics.getMetrics();
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    @Override
    public void close()
    {
        if (pageSource != null) {
            try {
                pageSource.close();
                metrics = pageSource.getMetrics();
            }
            catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        }
        else if (cursor != null) {
            cursor.close();
        }
    }

    private class SplitToPages
            implements WorkProcessor.Transformation<Split, WorkProcessor<Page>>
    {
        final Session session;
        final DriverYieldSignal yieldSignal;
        final PageSourceProvider pageSourceProvider;
        final CursorProcessor cursorProcessor;
        final PageProcessor pageProcessor;
        final TableHandle table;
        final List<ColumnHandle> columns;
        final DynamicFilter dynamicFilter;
        final List<Type> types;
        final LocalMemoryContext memoryContext;
        final AggregatedMemoryContext localAggregatedMemoryContext;
        final LocalMemoryContext pageSourceMemoryContext;
        final LocalMemoryContext outputMemoryContext;
        final DataSize minOutputPageSize;
        final int minOutputPageRowCount;
        final boolean avoidPageMaterialization;

        SplitToPages(
                Session session,
                DriverYieldSignal yieldSignal,
                PageSourceProvider pageSourceProvider,
                CursorProcessor cursorProcessor,
                PageProcessor pageProcessor,
                TableHandle table,
                Iterable<ColumnHandle> columns,
                DynamicFilter dynamicFilter,
                Iterable<Type> types,
                AggregatedMemoryContext aggregatedMemoryContext,
                DataSize minOutputPageSize,
                int minOutputPageRowCount,
                boolean avoidPageMaterialization)
        {
            this.session = requireNonNull(session, "session is null");
            this.yieldSignal = requireNonNull(yieldSignal, "yieldSignal is null");
            this.pageSourceProvider = requireNonNull(pageSourceProvider, "pageSourceProvider is null");
            this.cursorProcessor = requireNonNull(cursorProcessor, "cursorProcessor is null");
            this.pageProcessor = requireNonNull(pageProcessor, "pageProcessor is null");
            this.table = requireNonNull(table, "table is null");
            this.columns = ImmutableList.copyOf(requireNonNull(columns, "columns is null"));
            this.dynamicFilter = requireNonNull(dynamicFilter, "dynamicFilter is null");
            this.types = ImmutableList.copyOf(requireNonNull(types, "types is null"));
            this.memoryContext = aggregatedMemoryContext.newLocalMemoryContext(ScanFilterAndProjectOperator.class.getSimpleName());
            this.localAggregatedMemoryContext = newSimpleAggregatedMemoryContext();
            this.pageSourceMemoryContext = localAggregatedMemoryContext.newLocalMemoryContext(ScanFilterAndProjectOperator.class.getSimpleName());
            this.outputMemoryContext = localAggregatedMemoryContext.newLocalMemoryContext(ScanFilterAndProjectOperator.class.getSimpleName());
            this.minOutputPageSize = requireNonNull(minOutputPageSize, "minOutputPageSize is null");
            this.minOutputPageRowCount = minOutputPageRowCount;
            this.avoidPageMaterialization = avoidPageMaterialization;
        }

        @Override
        public TransformationState<WorkProcessor<Page>> process(Split split)
        {
            if (split == null) {
                memoryContext.close();
                return finished();
            }

            checkState(cursor == null && pageSource == null, "Table scan split already set");

            if (!dynamicFilter.getCurrentPredicate().isAll()) {
                dynamicFilterSplitsProcessed++;
            }

            ConnectorPageSource source;
            if (split.getConnectorSplit() instanceof EmptySplit) {
                source = new EmptyPageSource();
            }
            else {
                source = pageSourceProvider.createPageSource(session, split, table, columns, dynamicFilter);
            }

            if (source instanceof RecordPageSource) {
                cursor = ((RecordPageSource) source).getCursor();
                return ofResult(processColumnSource());
            }
            pageSource = source;
            return ofResult(processPageSource());
        }

        WorkProcessor<Page> processColumnSource()
        {
            return WorkProcessor
                    .create(new RecordCursorToPages(session, yieldSignal, cursorProcessor, types, pageSourceMemoryContext, outputMemoryContext))
                    .yielding(yieldSignal::isSet)
                    .blocking(() -> memoryContext.setBytes(localAggregatedMemoryContext.getBytes()));
        }

        WorkProcessor<Page> processPageSource()
        {
            ConnectorSession connectorSession = session.toConnectorSession();
            return WorkProcessor
                    .create(new ConnectorPageSourceToPages(pageSourceMemoryContext))
                    .yielding(yieldSignal::isSet)
                    .flatMap(page -> pageProcessor.createWorkProcessor(
                            connectorSession,
                            yieldSignal,
                            outputMemoryContext,
                            pageProcessorMetrics,
                            page,
                            avoidPageMaterialization))
                    .transformProcessor(processor -> mergePages(types, minOutputPageSize.toBytes(), minOutputPageRowCount, processor, localAggregatedMemoryContext))
                    .blocking(() -> memoryContext.setBytes(localAggregatedMemoryContext.getBytes()));
        }
    }

    private class RecordCursorToPages
            implements WorkProcessor.Process<Page>
    {
        final ConnectorSession session;
        final DriverYieldSignal yieldSignal;
        final CursorProcessor cursorProcessor;
        final PageBuilder pageBuilder;
        final LocalMemoryContext pageSourceMemoryContext;
        final LocalMemoryContext outputMemoryContext;

        boolean finished;

        RecordCursorToPages(
                Session session,
                DriverYieldSignal yieldSignal,
                CursorProcessor cursorProcessor,
                List<Type> types,
                LocalMemoryContext pageSourceMemoryContext,
                LocalMemoryContext outputMemoryContext)
        {
            this.session = session.toConnectorSession();
            this.yieldSignal = yieldSignal;
            this.cursorProcessor = cursorProcessor;
            this.pageBuilder = new PageBuilder(types);
            this.pageSourceMemoryContext = pageSourceMemoryContext;
            this.outputMemoryContext = outputMemoryContext;
        }

        @Override
        public ProcessState<Page> process()
        {
            if (!finished) {
                CursorProcessorOutput output = cursorProcessor.process(session, yieldSignal, cursor, pageBuilder);
                pageSourceMemoryContext.setBytes(cursor.getMemoryUsage());

                processedPositions += output.getProcessedRows();
                // TODO: derive better values for cursors
                processedBytes = cursor.getCompletedBytes();
                physicalBytes = cursor.getCompletedBytes();
                physicalPositions = processedPositions;
                readTimeNanos = cursor.getReadTimeNanos();
                if (output.isNoMoreRows()) {
                    finished = true;
                }
            }

            if (pageBuilder.isFull() || (finished && !pageBuilder.isEmpty())) {
                // only return a page if buffer is full or cursor has finished
                Page page = pageBuilder.build();
                pageBuilder.reset();
                outputMemoryContext.setBytes(pageBuilder.getRetainedSizeInBytes());
                return ProcessState.ofResult(page);
            }
            if (finished) {
                checkState(pageBuilder.isEmpty());
                return ProcessState.finished();
            }
            outputMemoryContext.setBytes(pageBuilder.getRetainedSizeInBytes());
            return ProcessState.yielded();
        }
    }

    private class ConnectorPageSourceToPages
            implements WorkProcessor.Process<Page>
    {
        final LocalMemoryContext pageSourceMemoryContext;

        ConnectorPageSourceToPages(LocalMemoryContext pageSourceMemoryContext)
        {
            this.pageSourceMemoryContext = pageSourceMemoryContext;
        }

        @Override
        public ProcessState<Page> process()
        {
            if (pageSource.isFinished()) {
                return ProcessState.finished();
            }

            CompletableFuture<?> isBlocked = pageSource.isBlocked();
            if (!isBlocked.isDone()) {
                return ProcessState.blocked(asVoid(toListenableFuture(isBlocked)));
            }

            Page page = pageSource.getNextPage();
            pageSourceMemoryContext.setBytes(pageSource.getMemoryUsage());

            if (page == null) {
                if (pageSource.isFinished()) {
                    return ProcessState.finished();
                }
                return ProcessState.yielded();
            }

            recordMaterializedBytes(page, sizeInBytes -> processedBytes += sizeInBytes);

            // update operator stats
            processedPositions += page.getPositionCount();
            physicalBytes = pageSource.getCompletedBytes();
            physicalPositions = pageSource.getCompletedPositions().orElse(processedPositions);
            readTimeNanos = pageSource.getReadTimeNanos();
            metrics = pageSource.getMetrics();

            return ProcessState.ofResult(page);
        }
    }

    private static <T> ListenableFuture<Void> asVoid(ListenableFuture<T> future)
    {
        return Futures.transform(future, v -> null, directExecutor());
    }

    public static class ScanFilterAndProjectOperatorFactory
            implements SourceOperatorFactory, AdapterWorkProcessorSourceOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final Supplier<CursorProcessor> cursorProcessor;
        private final Supplier<PageProcessor> pageProcessor;
        private final PlanNodeId sourceId;
        private final PageSourceProvider pageSourceProvider;
        private final TableHandle table;
        private final List<ColumnHandle> columns;
        private final DynamicFilter dynamicFilter;
        private final List<Type> types;
        private final DataSize minOutputPageSize;
        private final int minOutputPageRowCount;
        private boolean closed;

        public ScanFilterAndProjectOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                PlanNodeId sourceId,
                PageSourceProvider pageSourceProvider,
                Supplier<CursorProcessor> cursorProcessor,
                Supplier<PageProcessor> pageProcessor,
                TableHandle table,
                Iterable<ColumnHandle> columns,
                DynamicFilter dynamicFilter,
                List<Type> types,
                DataSize minOutputPageSize,
                int minOutputPageRowCount)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.cursorProcessor = requireNonNull(cursorProcessor, "cursorProcessor is null");
            this.pageProcessor = requireNonNull(pageProcessor, "pageProcessor is null");
            this.sourceId = requireNonNull(sourceId, "sourceId is null");
            this.pageSourceProvider = requireNonNull(pageSourceProvider, "pageSourceProvider is null");
            this.table = requireNonNull(table, "table is null");
            this.columns = ImmutableList.copyOf(requireNonNull(columns, "columns is null"));
            this.dynamicFilter = dynamicFilter;
            this.types = requireNonNull(types, "types is null");
            this.minOutputPageSize = requireNonNull(minOutputPageSize, "minOutputPageSize is null");
            this.minOutputPageRowCount = minOutputPageRowCount;
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getSourceId()
        {
            return sourceId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return planNodeId;
        }

        @Override
        public String getOperatorType()
        {
            return ScanFilterAndProjectOperator.class.getSimpleName();
        }

        @Override
        public SourceOperator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, getOperatorType());
            return new WorkProcessorSourceOperatorAdapter(operatorContext, this);
        }

        @Override
        public WorkProcessorSourceOperator create(
                Session session,
                MemoryTrackingContext memoryTrackingContext,
                DriverYieldSignal yieldSignal,
                WorkProcessor<Split> splits)
        {
            return create(session, memoryTrackingContext, yieldSignal, splits, true);
        }

        @Override
        public WorkProcessorSourceOperator createAdapterOperator(
                Session session,
                MemoryTrackingContext memoryTrackingContext,
                DriverYieldSignal yieldSignal,
                WorkProcessor<Split> splits)
        {
            return create(session, memoryTrackingContext, yieldSignal, splits, false);
        }

        private ScanFilterAndProjectOperator create(
                Session session,
                MemoryTrackingContext memoryTrackingContext,
                DriverYieldSignal yieldSignal,
                WorkProcessor<Split> splits,
                boolean avoidPageMaterialization)
        {
            return new ScanFilterAndProjectOperator(
                    session,
                    memoryTrackingContext,
                    yieldSignal,
                    splits,
                    pageSourceProvider,
                    cursorProcessor.get(),
                    pageProcessor.get(),
                    table,
                    columns,
                    dynamicFilter,
                    types,
                    minOutputPageSize,
                    minOutputPageRowCount,
                    avoidPageMaterialization);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }
    }
}
```



## ExchangeOperator

```java
public class ExchangeOperator
        implements SourceOperator
{
    public static final CatalogHandle REMOTE_CATALOG_HANDLE = createRootCatalogHandle("$remote", new CatalogVersion("remote"));

    public static class ExchangeOperatorFactory
            implements SourceOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId sourceId;
        private final DirectExchangeClientSupplier directExchangeClientSupplier;
        private final PagesSerdeFactory serdeFactory;
        private final RetryPolicy retryPolicy;
        private final ExchangeManagerRegistry exchangeManagerRegistry;
        private ExchangeDataSource exchangeDataSource;
        private boolean closed;

        private final NoMoreSplitsTracker noMoreSplitsTracker = new NoMoreSplitsTracker();
        private int nextOperatorInstanceId;

        public ExchangeOperatorFactory(
                int operatorId,
                PlanNodeId sourceId,
                DirectExchangeClientSupplier directExchangeClientSupplier,
                PagesSerdeFactory serdeFactory,
                RetryPolicy retryPolicy,
                ExchangeManagerRegistry exchangeManagerRegistry)
        {
            this.operatorId = operatorId;
            this.sourceId = sourceId;
            this.directExchangeClientSupplier = directExchangeClientSupplier;
            this.serdeFactory = serdeFactory;
            this.retryPolicy = requireNonNull(retryPolicy, "retryPolicy is null");
            this.exchangeManagerRegistry = requireNonNull(exchangeManagerRegistry, "exchangeManagerRegistry is null");
        }

        @Override
        public PlanNodeId getSourceId()
        {
            return sourceId;
        }

        @Override
        public SourceOperator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            TaskContext taskContext = driverContext.getPipelineContext().getTaskContext();
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, sourceId, ExchangeOperator.class.getSimpleName());
            LocalMemoryContext memoryContext = driverContext.getPipelineContext().localMemoryContext();
            if (exchangeDataSource == null) {
                // The decision of what exchange to use (streaming vs external) is currently made at the scheduling phase. It is more convenient to deliver it as part of a RemoteSplit.
                // Postponing this decision until scheduling allows to dynamically change the exchange type as part of an adaptive query re-planning.
                // LazyExchangeDataSource allows to choose an exchange source implementation based on the information received from a split.
                TaskId taskId = taskContext.getTaskId();
                exchangeDataSource = new LazyExchangeDataSource(
                        taskId.getQueryId(),
                        new ExchangeId(format("direct-exchange-%s-%s", taskId.getStageId().getId(), sourceId)),
                        directExchangeClientSupplier,
                        memoryContext,
                        taskContext::sourceTaskFailed,
                        retryPolicy,
                        exchangeManagerRegistry);
            }
            int operatorInstanceId = nextOperatorInstanceId;
            nextOperatorInstanceId++;
            ExchangeOperator exchangeOperator = new ExchangeOperator(
                    operatorContext,
                    sourceId,
                    exchangeDataSource,
                    serdeFactory.createDeserializer(driverContext.getSession().getExchangeEncryptionKey().map(Ciphers::deserializeAesEncryptionKey)),
                    noMoreSplitsTracker,
                    operatorInstanceId);
            noMoreSplitsTracker.operatorAdded(operatorInstanceId);
            return exchangeOperator;
        }

        @Override
        public void noMoreOperators()
        {
            noMoreSplitsTracker.noMoreOperators();
            if (noMoreSplitsTracker.isNoMoreSplits()) {
                if (exchangeDataSource != null) {
                    exchangeDataSource.noMoreInputs();
                }
            }
            closed = true;
        }
    }

    private final OperatorContext operatorContext;
    private final PlanNodeId sourceId;
    private final ExchangeDataSource exchangeDataSource;
    private final PageDeserializer deserializer;
    private final NoMoreSplitsTracker noMoreSplitsTracker;
    private final int operatorInstanceId;

    private ListenableFuture<Void> isBlocked = NOT_BLOCKED;

    public ExchangeOperator(
            OperatorContext operatorContext,
            PlanNodeId sourceId,
            ExchangeDataSource exchangeDataSource,
            PageDeserializer deserializer,
            NoMoreSplitsTracker noMoreSplitsTracker,
            int operatorInstanceId)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.sourceId = requireNonNull(sourceId, "sourceId is null");
        this.exchangeDataSource = requireNonNull(exchangeDataSource, "exchangeDataSource is null");
        this.deserializer = requireNonNull(deserializer, "serializer is null");
        this.noMoreSplitsTracker = requireNonNull(noMoreSplitsTracker, "noMoreSplitsTracker is null");
        this.operatorInstanceId = operatorInstanceId;

        LocalMemoryContext memoryContext = operatorContext.localUserMemoryContext();
        // memory footprint of deserializer does not change over time
        memoryContext.setBytes(deserializer.getRetainedSizeInBytes());

        operatorContext.setInfoSupplier(exchangeDataSource::getInfo);
    }

    @Override
    public PlanNodeId getSourceId()
    {
        return sourceId;
    }

    @Override
    public void addSplit(Split split)
    {
        requireNonNull(split, "split is null");
        checkArgument(split.getCatalogHandle().equals(REMOTE_CATALOG_HANDLE), "split is not a remote split");

        RemoteSplit remoteSplit = (RemoteSplit) split.getConnectorSplit();
        exchangeDataSource.addInput(remoteSplit.getExchangeInput());
    }

    @Override
    public void noMoreSplits()
    {
        noMoreSplitsTracker.noMoreSplits(operatorInstanceId);
        if (noMoreSplitsTracker.isNoMoreSplits()) {
            exchangeDataSource.noMoreInputs();
        }
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        close();
    }

    @Override
    public boolean isFinished()
    {
        return exchangeDataSource.isFinished();
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        // Avoid registering a new callback in the data source when one is already pending
        if (isBlocked.isDone()) {
            isBlocked = exchangeDataSource.isBlocked();
            if (isBlocked.isDone()) {
                isBlocked = NOT_BLOCKED;
            }
        }
        return isBlocked;
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException(getClass().getName() + " cannot take input");
    }

    @Override
    public Page getOutput()
    {
        Slice page = exchangeDataSource.pollPage();
        if (page == null) {
            return null;
        }

        Page deserializedPage = deserializer.deserialize(page);
        operatorContext.recordNetworkInput(page.length(), deserializedPage.getPositionCount());
        operatorContext.recordProcessedInput(deserializedPage.getSizeInBytes(), deserializedPage.getPositionCount());

        return deserializedPage;
    }

    @Override
    public void close()
    {
        exchangeDataSource.close();
    }

    @ThreadSafe
    private static class NoMoreSplitsTracker
    {
        private final IntSet allOperators = new IntOpenHashSet();
        private final IntSet noMoreSplitsOperators = new IntOpenHashSet();
        private boolean noMoreOperators;

        public synchronized void operatorAdded(int operatorInstanceId)
        {
            checkState(!noMoreOperators, "noMoreOperators is set");
            allOperators.add(operatorInstanceId);
        }

        public synchronized void noMoreOperators()
        {
            noMoreOperators = true;
        }

        public synchronized void noMoreSplits(int operatorInstanceId)
        {
            checkState(allOperators.contains(operatorInstanceId), "operatorInstanceId not found: %s", operatorInstanceId);
            noMoreSplitsOperators.add(operatorInstanceId);
        }

        public synchronized boolean isNoMoreSplits()
        {
            return noMoreOperators && noMoreSplitsOperators.containsAll(allOperators);
        }
    }
}
```



## DynamicFilterSourceOperator

```java
/**
 * This operator acts as a simple "pass-through" pipe, while saving its input pages.
 * The collected pages' value are used for creating a run-time filtering constraint (for probe-side table scan in an inner join).
 * We record all values for the run-time filter only for small build-side pages (which should be the case when using "broadcast" join).
 * For large inputs on build side, we can optionally record the min and max values per channel for orderable types (except Double and Real).
 */
public class DynamicFilterSourceOperator
        implements Operator
{
    private static final int EXPECTED_BLOCK_BUILDER_SIZE = 8;

    public static class Channel
    {
        private final DynamicFilterId filterId;
        private final Type type;
        private final int index;

        public Channel(DynamicFilterId filterId, Type type, int index)
        {
            this.filterId = filterId;
            this.type = type;
            this.index = index;
        }
    }

    public static class DynamicFilterSourceOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final DynamicFilterSourceConsumer dynamicPredicateConsumer;
        private final List<Channel> channels;
        private final int maxDistinctValues;
        private final DataSize maxFilterSize;
        private final int minMaxCollectionLimit;
        private final BlockTypeOperators blockTypeOperators;

        private boolean closed;
        private int createdOperatorsCount;

        public DynamicFilterSourceOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                DynamicFilterSourceConsumer dynamicPredicateConsumer,
                List<Channel> channels,
                int maxDistinctValues,
                DataSize maxFilterSize,
                int minMaxCollectionLimit,
                BlockTypeOperators blockTypeOperators)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.dynamicPredicateConsumer = requireNonNull(dynamicPredicateConsumer, "dynamicPredicateConsumer is null");
            this.channels = requireNonNull(channels, "channels is null");
            verify(channels.stream().map(channel -> channel.filterId).collect(toSet()).size() == channels.size(),
                    "duplicate dynamic filters are not allowed");
            verify(channels.stream().map(channel -> channel.index).collect(toSet()).size() == channels.size(),
                    "duplicate channel indices are not allowed");
            this.maxDistinctValues = maxDistinctValues;
            this.maxFilterSize = maxFilterSize;
            this.minMaxCollectionLimit = minMaxCollectionLimit;
            this.blockTypeOperators = requireNonNull(blockTypeOperators, "blockTypeOperators is null");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            createdOperatorsCount++;
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, DynamicFilterSourceOperator.class.getSimpleName());
            if (!dynamicPredicateConsumer.isDomainCollectionComplete()) {
                return new DynamicFilterSourceOperator(
                        operatorContext,
                        dynamicPredicateConsumer,
                        channels,
                        planNodeId,
                        maxDistinctValues,
                        maxFilterSize,
                        minMaxCollectionLimit,
                        blockTypeOperators);
            }
            // Return a pass-through operator which adds little overhead
            return new PassthroughDynamicFilterSourceOperator(operatorContext);
        }

        @Override
        public void noMoreOperators()
        {
            checkState(!closed, "Factory is already closed");
            closed = true;
            dynamicPredicateConsumer.setPartitionCount(createdOperatorsCount);
        }

        @Override
        public OperatorFactory duplicate()
        {
            // A duplicate factory may be required for DynamicFilterSourceOperatorFactory in fault tolerant execution mode
            // by LocalExecutionPlanner#addLookupOuterDrivers to add a new driver to output the unmatched rows in an outer join.
            // Since the logic for tracking partitions count for dynamicPredicateConsumer requires there to be only one DynamicFilterSourceOperatorFactory,
            // we turn off dynamic filtering and provide a duplicate factory which will act as pass through to allow the query to succeed.
            dynamicPredicateConsumer.addPartition(TupleDomain.all());
            return new DynamicFilterSourceOperatorFactory(
                    operatorId,
                    planNodeId,
                    new DynamicFilterSourceConsumer() {
                        @Override
                        public void addPartition(TupleDomain<DynamicFilterId> tupleDomain)
                        {
                            throw new UnsupportedOperationException();
                        }

                        @Override
                        public void setPartitionCount(int partitionCount) {}

                        @Override
                        public boolean isDomainCollectionComplete()
                        {
                            return true;
                        }
                    },
                    channels,
                    maxDistinctValues,
                    maxFilterSize,
                    minMaxCollectionLimit,
                    blockTypeOperators);
        }
    }

    private final OperatorContext context;
    private boolean finished;
    private Page current;
    private final DynamicFilterSourceConsumer dynamicPredicateConsumer;

    private final List<Channel> channels;
    private final ChannelFilter[] channelFilters;

    private int minMaxCollectionLimit;
    private boolean isDomainCollectionComplete;

    private DynamicFilterSourceOperator(
            OperatorContext context,
            DynamicFilterSourceConsumer dynamicPredicateConsumer,
            List<Channel> channels,
            PlanNodeId planNodeId,
            int maxDistinctValues,
            DataSize maxFilterSize,
            int minMaxCollectionLimit,
            BlockTypeOperators blockTypeOperators)
    {
        this.context = requireNonNull(context, "context is null");
        this.minMaxCollectionLimit = minMaxCollectionLimit;
        this.dynamicPredicateConsumer = requireNonNull(dynamicPredicateConsumer, "dynamicPredicateConsumer is null");
        this.channels = requireNonNull(channels, "channels is null");
        this.channelFilters = new ChannelFilter[channels.size()];

        for (int channelIndex = 0; channelIndex < channels.size(); ++channelIndex) {
            channelFilters[channelIndex] = new ChannelFilter(
                    blockTypeOperators,
                    minMaxCollectionLimit > 0,
                    planNodeId,
                    maxDistinctValues,
                    maxFilterSize.toBytes(),
                    this::finishDomainCollectionIfNecessary,
                    channels.get(channelIndex));
        }
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return context;
    }

    @Override
    public boolean needsInput()
    {
        return current == null && !finished;
    }

    @Override
    public void addInput(Page page)
    {
        verify(!finished, "DynamicFilterSourceOperator: addInput() may not be called after finish()");
        current = page;

        if (isDomainCollectionComplete) {
            return;
        }

        if (minMaxCollectionLimit >= 0) {
            minMaxCollectionLimit -= page.getPositionCount();
            if (minMaxCollectionLimit < 0) {
                for (int channelIndex = 0; channelIndex < channels.size(); channelIndex++) {
                    channelFilters[channelIndex].disableMinMax();
                }
                finishDomainCollectionIfNecessary();
            }
        }

        // Collect only the columns which are relevant for the JOIN.
        for (int channelIndex = 0; channelIndex < channels.size(); ++channelIndex) {
            Block block = page.getBlock(channels.get(channelIndex).index);
            channelFilters[channelIndex].process(block);
        }
    }

    @Override
    public Page getOutput()
    {
        Page result = current;
        current = null;
        return result;
    }

    @Override
    public void finish()
    {
        if (finished) {
            // NOTE: finish() may be called multiple times (see comment at Driver::processInternal).
            return;
        }
        finished = true;
        if (isDomainCollectionComplete) {
            // dynamicPredicateConsumer was already notified with 'all'
            return;
        }

        ImmutableMap.Builder<DynamicFilterId, Domain> domainsBuilder = ImmutableMap.builder();
        for (int channelIndex = 0; channelIndex < channels.size(); ++channelIndex) {
            DynamicFilterId filterId = channels.get(channelIndex).filterId;
            domainsBuilder.put(filterId, channelFilters[channelIndex].getDomain());
        }
        dynamicPredicateConsumer.addPartition(TupleDomain.withColumnDomains(domainsBuilder.buildOrThrow()));
    }

    @Override
    public boolean isFinished()
    {
        return current == null && finished;
    }

    private void finishDomainCollectionIfNecessary()
    {
        if (!isDomainCollectionComplete && stream(channelFilters).allMatch(channel -> channel.state == ChannelState.NONE)) {
            // allow all probe-side values to be read.
            dynamicPredicateConsumer.addPartition(TupleDomain.all());
            isDomainCollectionComplete = true;
        }
    }

    private static boolean isMinMaxPossible(Type type)
    {
        // Skipping DOUBLE and REAL in collectMinMaxValues to avoid dealing with NaN values
        return type.isOrderable() && type != DOUBLE && type != REAL;
    }

    private static class PassthroughDynamicFilterSourceOperator
            implements Operator
    {
        private final OperatorContext operatorContext;
        private boolean finished;
        private Page current;

        private PassthroughDynamicFilterSourceOperator(OperatorContext operatorContext)
        {
            this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        }

        @Override
        public OperatorContext getOperatorContext()
        {
            return operatorContext;
        }

        @Override
        public boolean needsInput()
        {
            return current == null && !finished;
        }

        @Override
        public void addInput(Page page)
        {
            verify(!finished, "DynamicFilterSourceOperator: addInput() may not be called after finish()");
            current = page;
        }

        @Override
        public Page getOutput()
        {
            Page result = current;
            current = null;
            return result;
        }

        @Override
        public void finish()
        {
            if (finished) {
                // NOTE: finish() may be called multiple times (see comment at Driver::processInternal).
                return;
            }
            finished = true;
        }

        @Override
        public boolean isFinished()
        {
            return current == null && finished;
        }
    }

    private static class ChannelFilter
    {
        private final Type type;
        private final int maxDistinctValues;
        private final long maxFilterSizeInBytes;
        private final Runnable notifyStateChange;

        private ChannelState state;
        private boolean collectMinMax;
        // May be dropped if the predicate becomes too large.
        @Nullable
        private BlockBuilder blockBuilder;
        @Nullable
        private TypedSet valueSet;
        @Nullable
        private Block minValues;
        @Nullable
        private Block maxValues;
        @Nullable
        private BlockPositionComparison minMaxComparison;

        private ChannelFilter(
                BlockTypeOperators blockTypeOperators,
                boolean minMaxEnabled,
                PlanNodeId planNodeId,
                int maxDistinctValues,
                long maxFilterSizeInBytes,
                Runnable notifyStateChange,
                Channel channel)
        {
            this.maxDistinctValues = maxDistinctValues;
            this.maxFilterSizeInBytes = maxFilterSizeInBytes;
            this.notifyStateChange = requireNonNull(notifyStateChange, "notifyStateChange is null");
            type = channel.type;
            state = ChannelState.SET;
            collectMinMax = minMaxEnabled && isMinMaxPossible(type);
            if (collectMinMax) {
                minMaxComparison = blockTypeOperators.getComparisonUnorderedLastOperator(type);
            }
            blockBuilder = type.createBlockBuilder(null, EXPECTED_BLOCK_BUILDER_SIZE);
            valueSet = createUnboundedEqualityTypedSet(
                    type,
                    blockTypeOperators.getEqualOperator(type),
                    blockTypeOperators.getHashCodeOperator(type),
                    blockBuilder,
                    EXPECTED_BLOCK_BUILDER_SIZE,
                    format("DynamicFilterSourceOperator_%s_%d", planNodeId, channel.index));
        }

        private void process(Block block)
        {
            // TODO: we should account for the memory used for collecting build-side values using MemoryContext
            switch (state) {
                case SET:
                    for (int position = 0; position < block.getPositionCount(); ++position) {
                        valueSet.add(block, position);
                    }
                    if (valueSet.size() > maxDistinctValues || valueSet.getRetainedSizeInBytes() > maxFilterSizeInBytes) {
                        if (collectMinMax) {
                            state = ChannelState.MIN_MAX;
                            updateMinMaxValues(blockBuilder.build(), minMaxComparison);
                        }
                        else {
                            state = ChannelState.NONE;
                            notifyStateChange.run();
                        }
                        valueSet = null;
                        blockBuilder = null;
                    }
                    break;
                case MIN_MAX:
                    updateMinMaxValues(block, minMaxComparison);
                    break;
                case NONE:
                    break;
            }
        }

        private Domain getDomain()
        {
            return switch (state) {
                case SET -> convertToDomain();
                case MIN_MAX -> {
                    if (minValues == null) {
                        // all values were null
                        yield Domain.none(type);
                    }
                    Object min = blockToNativeValue(type, minValues);
                    Object max = blockToNativeValue(type, maxValues);
                    yield Domain.create(ValueSet.ofRanges(range(type, min, true, max, true)), false);
                }
                case NONE -> Domain.all(type);
            };
        }

        private void disableMinMax()
        {
            collectMinMax = false;
            if (state == ChannelState.MIN_MAX) {
                state = ChannelState.NONE;
            }
            // Drop references to collected values.
            minValues = null;
            maxValues = null;
        }

        private Domain convertToDomain()
        {
            Block block = blockBuilder.build();
            ImmutableList.Builder<Object> values = ImmutableList.builder();
            for (int position = 0; position < block.getPositionCount(); ++position) {
                Object value = readNativeValue(type, block, position);
                if (value != null) {
                    // join doesn't match rows with NaN values.
                    if (!isFloatingPointNaN(type, value)) {
                        values.add(value);
                    }
                }
            }

            // Inner and right join doesn't match rows with null key column values.
            return Domain.create(ValueSet.copyOf(type, values.build()), false);
        }

        private void updateMinMaxValues(Block block, BlockPositionComparison comparison)
        {
            int minValuePosition = -1;
            int maxValuePosition = -1;

            for (int position = 0; position < block.getPositionCount(); ++position) {
                if (block.isNull(position)) {
                    continue;
                }
                if (minValuePosition == -1) {
                    // First non-null value
                    minValuePosition = position;
                    maxValuePosition = position;
                    continue;
                }
                if (comparison.compare(block, position, block, minValuePosition) < 0) {
                    minValuePosition = position;
                }
                else if (comparison.compare(block, position, block, maxValuePosition) > 0) {
                    maxValuePosition = position;
                }
            }

            if (minValuePosition == -1) {
                // all block values are nulls
                return;
            }
            if (minValues == null) {
                // First Page with non-null value for this block
                minValues = block.getSingleValueBlock(minValuePosition);
                maxValues = block.getSingleValueBlock(maxValuePosition);
                return;
            }
            // Compare with min/max values from previous Pages
            Block currentMin = minValues;
            Block currentMax = maxValues;

            if (comparison.compare(block, minValuePosition, currentMin, 0) < 0) {
                minValues = block.getSingleValueBlock(minValuePosition);
            }
            if (comparison.compare(block, maxValuePosition, currentMax, 0) > 0) {
                maxValues = block.getSingleValueBlock(maxValuePosition);
            }
        }
    }

    private enum ChannelState
    {
        SET,
        MIN_MAX,
        NONE,
    }
}
```



## HashAggregationOperator

```java
public class HashAggregationOperator
        implements Operator
{
    private static final double MERGE_WITH_MEMORY_RATIO = 0.9;

    public static class HashAggregationOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Type> groupByTypes;
        private final List<Integer> groupByChannels;
        private final List<Integer> globalAggregationGroupIds;
        private final Step step;
        private final boolean produceDefaultOutput;
        private final List<AggregatorFactory> aggregatorFactories;
        private final Optional<Integer> hashChannel;
        private final Optional<Integer> groupIdChannel;

        private final int expectedGroups;
        private final Optional<DataSize> maxPartialMemory;
        private final boolean spillEnabled;
        private final DataSize memoryLimitForMerge;
        private final DataSize memoryLimitForMergeWithMemory;
        private final SpillerFactory spillerFactory;
        private final JoinCompiler joinCompiler;
        private final BlockTypeOperators blockTypeOperators;
        private final Optional<PartialAggregationController> partialAggregationController;

        private boolean closed;

        @VisibleForTesting
        public HashAggregationOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                List<? extends Type> groupByTypes,
                List<Integer> groupByChannels,
                List<Integer> globalAggregationGroupIds,
                Step step,
                List<AggregatorFactory> aggregatorFactories,
                Optional<Integer> hashChannel,
                Optional<Integer> groupIdChannel,
                int expectedGroups,
                Optional<DataSize> maxPartialMemory,
                JoinCompiler joinCompiler,
                BlockTypeOperators blockTypeOperators,
                Optional<PartialAggregationController> partialAggregationController)
        {
            this(operatorId,
                    planNodeId,
                    groupByTypes,
                    groupByChannels,
                    globalAggregationGroupIds,
                    step,
                    false,
                    aggregatorFactories,
                    hashChannel,
                    groupIdChannel,
                    expectedGroups,
                    maxPartialMemory,
                    false,
                    DataSize.of(0, MEGABYTE),
                    DataSize.of(0, MEGABYTE),
                    (types, spillContext, memoryContext) -> {
                        throw new UnsupportedOperationException();
                    },
                    joinCompiler,
                    blockTypeOperators,
                    partialAggregationController);
        }

        public HashAggregationOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                List<? extends Type> groupByTypes,
                List<Integer> groupByChannels,
                List<Integer> globalAggregationGroupIds,
                Step step,
                boolean produceDefaultOutput,
                List<AggregatorFactory> aggregatorFactories,
                Optional<Integer> hashChannel,
                Optional<Integer> groupIdChannel,
                int expectedGroups,
                Optional<DataSize> maxPartialMemory,
                boolean spillEnabled,
                DataSize unspillMemoryLimit,
                SpillerFactory spillerFactory,
                JoinCompiler joinCompiler,
                BlockTypeOperators blockTypeOperators,
                Optional<PartialAggregationController> partialAggregationController)
        {
            this(operatorId,
                    planNodeId,
                    groupByTypes,
                    groupByChannels,
                    globalAggregationGroupIds,
                    step,
                    produceDefaultOutput,
                    aggregatorFactories,
                    hashChannel,
                    groupIdChannel,
                    expectedGroups,
                    maxPartialMemory,
                    spillEnabled,
                    unspillMemoryLimit,
                    DataSize.succinctBytes((long) (unspillMemoryLimit.toBytes() * MERGE_WITH_MEMORY_RATIO)),
                    spillerFactory,
                    joinCompiler,
                    blockTypeOperators,
                    partialAggregationController);
        }

        @VisibleForTesting
        HashAggregationOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                List<? extends Type> groupByTypes,
                List<Integer> groupByChannels,
                List<Integer> globalAggregationGroupIds,
                Step step,
                boolean produceDefaultOutput,
                List<AggregatorFactory> aggregatorFactories,
                Optional<Integer> hashChannel,
                Optional<Integer> groupIdChannel,
                int expectedGroups,
                Optional<DataSize> maxPartialMemory,
                boolean spillEnabled,
                DataSize memoryLimitForMerge,
                DataSize memoryLimitForMergeWithMemory,
                SpillerFactory spillerFactory,
                JoinCompiler joinCompiler,
                BlockTypeOperators blockTypeOperators,
                Optional<PartialAggregationController> partialAggregationController)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.hashChannel = requireNonNull(hashChannel, "hashChannel is null");
            this.groupIdChannel = requireNonNull(groupIdChannel, "groupIdChannel is null");
            this.groupByTypes = ImmutableList.copyOf(groupByTypes);
            this.groupByChannels = ImmutableList.copyOf(groupByChannels);
            this.globalAggregationGroupIds = ImmutableList.copyOf(globalAggregationGroupIds);
            this.step = step;
            this.produceDefaultOutput = produceDefaultOutput;
            this.aggregatorFactories = ImmutableList.copyOf(aggregatorFactories);
            this.expectedGroups = expectedGroups;
            this.maxPartialMemory = requireNonNull(maxPartialMemory, "maxPartialMemory is null");
            this.spillEnabled = spillEnabled;
            this.memoryLimitForMerge = requireNonNull(memoryLimitForMerge, "memoryLimitForMerge is null");
            this.memoryLimitForMergeWithMemory = requireNonNull(memoryLimitForMergeWithMemory, "memoryLimitForMergeWithMemory is null");
            this.spillerFactory = requireNonNull(spillerFactory, "spillerFactory is null");
            this.joinCompiler = requireNonNull(joinCompiler, "joinCompiler is null");
            this.blockTypeOperators = requireNonNull(blockTypeOperators, "blockTypeOperators is null");
            this.partialAggregationController = requireNonNull(partialAggregationController, "partialAggregationController is null");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");

            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, HashAggregationOperator.class.getSimpleName());
            HashAggregationOperator hashAggregationOperator = new HashAggregationOperator(
                    operatorContext,
                    groupByTypes,
                    groupByChannels,
                    globalAggregationGroupIds,
                    step,
                    produceDefaultOutput,
                    aggregatorFactories,
                    hashChannel,
                    groupIdChannel,
                    expectedGroups,
                    maxPartialMemory,
                    spillEnabled,
                    memoryLimitForMerge,
                    memoryLimitForMergeWithMemory,
                    spillerFactory,
                    joinCompiler,
                    blockTypeOperators,
                    partialAggregationController);
            return hashAggregationOperator;
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new HashAggregationOperatorFactory(
                    operatorId,
                    planNodeId,
                    groupByTypes,
                    groupByChannels,
                    globalAggregationGroupIds,
                    step,
                    produceDefaultOutput,
                    aggregatorFactories,
                    hashChannel,
                    groupIdChannel,
                    expectedGroups,
                    maxPartialMemory,
                    spillEnabled,
                    memoryLimitForMerge,
                    memoryLimitForMergeWithMemory,
                    spillerFactory,
                    joinCompiler,
                    blockTypeOperators,
                    partialAggregationController.map(PartialAggregationController::duplicate));
        }
    }

    private final OperatorContext operatorContext;
    private final Optional<PartialAggregationController> partialAggregationController;
    private final List<Type> groupByTypes;
    private final List<Integer> groupByChannels;
    private final List<Integer> globalAggregationGroupIds;
    private final Step step;
    private final boolean produceDefaultOutput;
    private final List<AggregatorFactory> aggregatorFactories;
    private final Optional<Integer> hashChannel;
    private final Optional<Integer> groupIdChannel;
    private final int expectedGroups;
    private final Optional<DataSize> maxPartialMemory;
    private final boolean spillEnabled;
    private final DataSize memoryLimitForMerge;
    private final DataSize memoryLimitForMergeWithMemory;
    private final SpillerFactory spillerFactory;
    private final JoinCompiler joinCompiler;
    private final BlockTypeOperators blockTypeOperators;

    private final List<Type> types;

    private HashAggregationBuilder aggregationBuilder;
    private final LocalMemoryContext memoryContext;
    private WorkProcessor<Page> outputPages;
    private boolean inputProcessed;
    private boolean finishing;
    private boolean finished;

    // for yield when memory is not available
    private Work<?> unfinishedWork;
    private long numberOfInputRowsProcessed;
    private long numberOfUniqueRowsProduced;

    private HashAggregationOperator(
            OperatorContext operatorContext,
            List<Type> groupByTypes,
            List<Integer> groupByChannels,
            List<Integer> globalAggregationGroupIds,
            Step step,
            boolean produceDefaultOutput,
            List<AggregatorFactory> aggregatorFactories,
            Optional<Integer> hashChannel,
            Optional<Integer> groupIdChannel,
            int expectedGroups,
            Optional<DataSize> maxPartialMemory,
            boolean spillEnabled,
            DataSize memoryLimitForMerge,
            DataSize memoryLimitForMergeWithMemory,
            SpillerFactory spillerFactory,
            JoinCompiler joinCompiler,
            BlockTypeOperators blockTypeOperators,
            Optional<PartialAggregationController> partialAggregationController)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.partialAggregationController = requireNonNull(partialAggregationController, "partialAggregationControl is null");
        requireNonNull(step, "step is null");
        requireNonNull(aggregatorFactories, "aggregatorFactories is null");
        requireNonNull(operatorContext, "operatorContext is null");
        checkArgument(partialAggregationController.isEmpty() || step.isOutputPartial(), "partialAggregationController should be present only for partial aggregation");

        this.groupByTypes = ImmutableList.copyOf(groupByTypes);
        this.groupByChannels = ImmutableList.copyOf(groupByChannels);
        this.globalAggregationGroupIds = ImmutableList.copyOf(globalAggregationGroupIds);
        this.aggregatorFactories = ImmutableList.copyOf(aggregatorFactories);
        this.hashChannel = requireNonNull(hashChannel, "hashChannel is null");
        this.groupIdChannel = requireNonNull(groupIdChannel, "groupIdChannel is null");
        this.step = step;
        this.produceDefaultOutput = produceDefaultOutput;
        this.expectedGroups = expectedGroups;
        this.maxPartialMemory = requireNonNull(maxPartialMemory, "maxPartialMemory is null");
        this.types = toTypes(groupByTypes, aggregatorFactories, hashChannel);
        this.spillEnabled = spillEnabled;
        this.memoryLimitForMerge = requireNonNull(memoryLimitForMerge, "memoryLimitForMerge is null");
        this.memoryLimitForMergeWithMemory = requireNonNull(memoryLimitForMergeWithMemory, "memoryLimitForMergeWithMemory is null");
        this.spillerFactory = requireNonNull(spillerFactory, "spillerFactory is null");
        this.joinCompiler = requireNonNull(joinCompiler, "joinCompiler is null");
        this.blockTypeOperators = requireNonNull(blockTypeOperators, "blockTypeOperators is null");

        this.memoryContext = operatorContext.localUserMemoryContext();
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        finishing = true;
    }

    @Override
    public boolean isFinished()
    {
        return finished;
    }

    @Override
    public boolean needsInput()
    {
        if (finishing || outputPages != null) {
            return false;
        }
        if (aggregationBuilder != null && aggregationBuilder.isFull()) {
            return false;
        }
        return unfinishedWork == null;
    }

    @Override
    public void addInput(Page page)
    {
        checkState(unfinishedWork == null, "Operator has unfinished work");
        checkState(!finishing, "Operator is already finishing");
        requireNonNull(page, "page is null");
        inputProcessed = true;

        if (aggregationBuilder == null) {
            boolean partialAggregationDisabled = partialAggregationController
                    .map(PartialAggregationController::isPartialAggregationDisabled)
                    .orElse(false);
            if (step.isOutputPartial() && partialAggregationDisabled) {
                aggregationBuilder = new SkipAggregationBuilder(groupByChannels, hashChannel, aggregatorFactories, memoryContext);
            }
            else if (step.isOutputPartial() || !spillEnabled || !isSpillable()) {
                // TODO: We ignore spillEnabled here if any aggregate has ORDER BY clause or DISTINCT because they are not yet implemented for spilling.
                aggregationBuilder = new InMemoryHashAggregationBuilder(
                        aggregatorFactories,
                        step,
                        expectedGroups,
                        groupByTypes,
                        groupByChannels,
                        hashChannel,
                        operatorContext,
                        maxPartialMemory,
                        joinCompiler,
                        blockTypeOperators,
                        () -> {
                            memoryContext.setBytes(((InMemoryHashAggregationBuilder) aggregationBuilder).getSizeInMemory());
                            if (step.isOutputPartial() && maxPartialMemory.isPresent()) {
                                // do not yield on memory for partial aggregations
                                return true;
                            }
                            return operatorContext.isWaitingForMemory().isDone();
                        });
            }
            else {
                aggregationBuilder = new SpillableHashAggregationBuilder(
                        aggregatorFactories,
                        step,
                        expectedGroups,
                        groupByTypes,
                        groupByChannels,
                        hashChannel,
                        operatorContext,
                        memoryLimitForMerge,
                        memoryLimitForMergeWithMemory,
                        spillerFactory,
                        joinCompiler,
                        blockTypeOperators);
            }

            // assume initial aggregationBuilder is not full
        }
        else {
            checkState(!aggregationBuilder.isFull(), "Aggregation buffer is full");
        }

        // process the current page; save the unfinished work if we are waiting for memory
        unfinishedWork = aggregationBuilder.processPage(page);
        if (unfinishedWork.process()) {
            unfinishedWork = null;
        }
        aggregationBuilder.updateMemory();
        numberOfInputRowsProcessed += page.getPositionCount();
    }

    private boolean isSpillable()
    {
        return aggregatorFactories.stream().allMatch(AggregatorFactory::isSpillable);
    }

    @Override
    public ListenableFuture<Void> startMemoryRevoke()
    {
        if (aggregationBuilder != null) {
            return aggregationBuilder.startMemoryRevoke();
        }
        return NOT_BLOCKED;
    }

    @Override
    public void finishMemoryRevoke()
    {
        if (aggregationBuilder != null) {
            aggregationBuilder.finishMemoryRevoke();
        }
    }

    @Override
    public Page getOutput()
    {
        if (finished) {
            return null;
        }

        // process unfinished work if one exists
        if (unfinishedWork != null) {
            boolean workDone = unfinishedWork.process();
            aggregationBuilder.updateMemory();
            if (!workDone) {
                return null;
            }
            unfinishedWork = null;
        }

        if (outputPages == null) {
            if (finishing) {
                if (!inputProcessed && produceDefaultOutput) {
                    // global aggregations always generate an output row with the default aggregation output (e.g. 0 for COUNT, NULL for SUM)
                    finished = true;
                    return getGlobalAggregationOutput();
                }

                if (aggregationBuilder == null) {
                    finished = true;
                    return null;
                }
            }

            // only flush if we are finishing or the aggregation builder is full
            if (!finishing && (aggregationBuilder == null || !aggregationBuilder.isFull())) {
                return null;
            }

            outputPages = aggregationBuilder.buildResult();
        }

        if (!outputPages.process()) {
            return null;
        }

        if (outputPages.isFinished()) {
            closeAggregationBuilder();
            return null;
        }

        Page result = outputPages.getResult();
        numberOfUniqueRowsProduced += result.getPositionCount();
        return result;
    }

    @Override
    public void close()
    {
        closeAggregationBuilder();
    }

    @VisibleForTesting
    public HashAggregationBuilder getAggregationBuilder()
    {
        return aggregationBuilder;
    }

    private void closeAggregationBuilder()
    {
        outputPages = null;
        if (aggregationBuilder != null) {
            aggregationBuilder.close();
            // aggregationBuilder.close() will release all memory reserved in memory accounting.
            // The reference must be set to null afterwards to avoid unaccounted memory.
            aggregationBuilder = null;
        }
        memoryContext.setBytes(0);
        partialAggregationController.ifPresent(
                controller -> controller.onFlush(numberOfInputRowsProcessed, numberOfUniqueRowsProduced));
        numberOfInputRowsProcessed = 0;
        numberOfUniqueRowsProduced = 0;
    }

    private Page getGlobalAggregationOutput()
    {
        // global aggregation output page will only be constructed once,
        // so a new PageBuilder is constructed (instead of using PageBuilder.reset)
        PageBuilder output = new PageBuilder(globalAggregationGroupIds.size(), types);

        for (int groupId : globalAggregationGroupIds) {
            output.declarePosition();
            int channel = 0;

            while (channel < groupByTypes.size()) {
                if (channel == groupIdChannel.orElseThrow()) {
                    output.getBlockBuilder(channel).writeLong(groupId);
                }
                else {
                    output.getBlockBuilder(channel).appendNull();
                }
                channel++;
            }

            if (hashChannel.isPresent()) {
                long hashValue = calculateDefaultOutputHash(groupByTypes, groupIdChannel.orElseThrow(), groupId);
                output.getBlockBuilder(channel).writeLong(hashValue);
                channel++;
            }

            for (AggregatorFactory aggregatorFactory : aggregatorFactories) {
                aggregatorFactory.createAggregator().evaluate(output.getBlockBuilder(channel));
                channel++;
            }
        }

        if (output.isEmpty()) {
            return null;
        }
        return output.build();
    }

    private static long calculateDefaultOutputHash(List<Type> groupByChannels, int groupIdChannel, int groupId)
    {
        // Default output has NULLs on all columns except of groupIdChannel
        long result = INITIAL_HASH_VALUE;
        for (int channel = 0; channel < groupByChannels.size(); channel++) {
            if (channel != groupIdChannel) {
                result = CombineHashFunction.getHash(result, NULL_HASH_CODE);
            }
            else {
                result = CombineHashFunction.getHash(result, BigintType.hash(groupId));
            }
        }
        return result;
    }
}
```



## TopNOperator

```java
/**
 * Returns the top N rows from the source sorted according to the specified ordering in the keyChannelIndex channel.
 */
public class TopNOperator
        implements WorkProcessorOperator
{
    public static OperatorFactory createOperatorFactory(
            int operatorId,
            PlanNodeId planNodeId,
            List<? extends Type> types,
            int n,
            List<Integer> sortChannels,
            List<SortOrder> sortOrders,
            TypeOperators typeOperators)
    {
        return createAdapterOperatorFactory(new Factory(operatorId, planNodeId, types, n, sortChannels, sortOrders, typeOperators));
    }

    private static class Factory
            implements BasicAdapterWorkProcessorOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Type> sourceTypes;
        private final int n;
        private final List<Integer> sortChannels;
        private final List<SortOrder> sortOrders;
        private final TypeOperators typeOperators;
        private boolean closed;

        private Factory(
                int operatorId,
                PlanNodeId planNodeId,
                List<? extends Type> types,
                int n,
                List<Integer> sortChannels,
                List<SortOrder> sortOrders,
                TypeOperators typeOperators)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.sourceTypes = ImmutableList.copyOf(requireNonNull(types, "types is null"));
            this.n = n;
            this.sortChannels = ImmutableList.copyOf(requireNonNull(sortChannels, "sortChannels is null"));
            this.sortOrders = ImmutableList.copyOf(requireNonNull(sortOrders, "sortOrders is null"));
            this.typeOperators = typeOperators;
        }

        @Override
        public WorkProcessorOperator create(
                ProcessorContext processorContext,
                WorkProcessor<Page> sourcePages)
        {
            checkState(!closed, "Factory is already closed");
            return new TopNOperator(
                    processorContext.getMemoryTrackingContext(),
                    sourcePages,
                    sourceTypes,
                    n,
                    sortChannels,
                    sortOrders,
                    typeOperators);
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return planNodeId;
        }

        @Override
        public String getOperatorType()
        {
            return TopNOperator.class.getSimpleName();
        }

        @Override
        public void close()
        {
            closed = true;
        }

        @Override
        public Factory duplicate()
        {
            return new Factory(operatorId, planNodeId, sourceTypes, n, sortChannels, sortOrders, typeOperators);
        }
    }

    private final TopNProcessor topNProcessor;
    private final WorkProcessor<Page> pages;

    private TopNOperator(
            MemoryTrackingContext memoryTrackingContext,
            WorkProcessor<Page> sourcePages,
            List<Type> types,
            int n,
            List<Integer> sortChannels,
            List<SortOrder> sortOrders,
            TypeOperators typeOperators)
    {
        this.topNProcessor = new TopNProcessor(
                memoryTrackingContext.aggregateUserMemoryContext(),
                types,
                n,
                sortChannels,
                sortOrders,
                typeOperators);

        if (n == 0) {
            pages = WorkProcessor.of();
        }
        else {
            pages = sourcePages.transform(new TopNPages());
        }
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    private class TopNPages
            implements WorkProcessor.Transformation<Page, Page>
    {
        @Override
        public TransformationState<Page> process(Page inputPage)
        {
            if (inputPage != null) {
                topNProcessor.addInput(inputPage);
                return TransformationState.needsMoreData();
            }

            // no more input, return results
            Page page = null;
            while (page == null && !topNProcessor.noMoreOutput()) {
                page = topNProcessor.getOutput();
            }

            if (page != null) {
                return TransformationState.ofResult(page, false);
            }

            return TransformationState.finished();
        }
    }
}
```



## MergeOperator

```java
public class MergeOperator
        implements SourceOperator
{
    public static class MergeOperatorFactory
            implements SourceOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId sourceId;
        private final DirectExchangeClientSupplier directExchangeClientSupplier;
        private final PagesSerdeFactory serdeFactory;
        private final List<Type> types;
        private final List<Integer> outputChannels;
        private final List<Type> outputTypes;
        private final List<Integer> sortChannels;
        private final List<SortOrder> sortOrder;
        private final OrderingCompiler orderingCompiler;
        private boolean closed;

        public MergeOperatorFactory(
                int operatorId,
                PlanNodeId sourceId,
                DirectExchangeClientSupplier directExchangeClientSupplier,
                PagesSerdeFactory serdeFactory,
                OrderingCompiler orderingCompiler,
                List<Type> types,
                List<Integer> outputChannels,
                List<Integer> sortChannels,
                List<SortOrder> sortOrder)
        {
            this.operatorId = operatorId;
            this.sourceId = requireNonNull(sourceId, "sourceId is null");
            this.directExchangeClientSupplier = requireNonNull(directExchangeClientSupplier, "directExchangeClientSupplier is null");
            this.serdeFactory = requireNonNull(serdeFactory, "serdeFactory is null");
            this.types = requireNonNull(types, "types is null");
            this.outputChannels = requireNonNull(outputChannels, "outputChannels is null");
            this.outputTypes = mappedCopy(outputChannels, types::get);
            this.sortChannels = requireNonNull(sortChannels, "sortChannels is null");
            this.sortOrder = requireNonNull(sortOrder, "sortOrder is null");
            this.orderingCompiler = requireNonNull(orderingCompiler, "orderingCompiler is null");
        }

        @Override
        public PlanNodeId getSourceId()
        {
            return sourceId;
        }

        @Override
        public SourceOperator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, sourceId, MergeOperator.class.getSimpleName());

            return new MergeOperator(
                    operatorContext,
                    sourceId,
                    directExchangeClientSupplier,
                    serdeFactory.createDeserializer(driverContext.getSession().getExchangeEncryptionKey().map(Ciphers::deserializeAesEncryptionKey)),
                    orderingCompiler.compilePageWithPositionComparator(types, sortChannels, sortOrder),
                    outputChannels,
                    outputTypes);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }
    }

    private final OperatorContext operatorContext;
    private final PlanNodeId sourceId;
    private final DirectExchangeClientSupplier directExchangeClientSupplier;
    private final PageDeserializer deserializer;
    private final PageWithPositionComparator comparator;
    private final List<Integer> outputChannels;
    private final List<Type> outputTypes;

    private final SettableFuture<Void> blockedOnSplits = SettableFuture.create();

    private final List<WorkProcessor<Page>> pageProducers = new ArrayList<>();
    private final Closer closer = Closer.create();

    private WorkProcessor<Page> mergedPages;
    private boolean closed;

    public MergeOperator(
            OperatorContext operatorContext,
            PlanNodeId sourceId,
            DirectExchangeClientSupplier directExchangeClientSupplier,
            PageDeserializer deserializer,
            PageWithPositionComparator comparator,
            List<Integer> outputChannels,
            List<Type> outputTypes)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.sourceId = requireNonNull(sourceId, "sourceId is null");
        this.directExchangeClientSupplier = requireNonNull(directExchangeClientSupplier, "directExchangeClientSupplier is null");
        this.deserializer = requireNonNull(deserializer, "deserializer is null");
        this.comparator = requireNonNull(comparator, "comparator is null");
        this.outputChannels = requireNonNull(outputChannels, "outputChannels is null");
        this.outputTypes = requireNonNull(outputTypes, "outputTypes is null");

        LocalMemoryContext memoryContext = operatorContext.newLocalUserMemoryContext(MergeOperator.class.getSimpleName());
        // memory footprint of deserializer does not change over time
        memoryContext.setBytes(deserializer.getRetainedSizeInBytes());
        closer.register(memoryContext::close);
    }

    @Override
    public PlanNodeId getSourceId()
    {
        return sourceId;
    }

    @Override
    public void addSplit(Split split)
    {
        requireNonNull(split, "split is null");
        checkArgument(split.getConnectorSplit() instanceof RemoteSplit, "split is not a remote split");
        checkState(!blockedOnSplits.isDone(), "noMoreSplits has been called already");

        TaskContext taskContext = operatorContext.getDriverContext().getPipelineContext().getTaskContext();
        DirectExchangeClient client = closer.register(directExchangeClientSupplier.get(
                taskContext.getTaskId().getQueryId(),
                new ExchangeId(format("direct-exchange-merge-%s-%s", taskContext.getTaskId().getStageId().getId(), sourceId)),
                operatorContext.localUserMemoryContext(),
                taskContext::sourceTaskFailed,
                RetryPolicy.NONE));
        RemoteSplit remoteSplit = (RemoteSplit) split.getConnectorSplit();
        // Only fault tolerant execution mode is expected to execute external exchanges.
        // MergeOperator is used for distributed sort only and it is not compatible (and disabled) with fault tolerant execution mode.
        DirectExchangeInput exchangeInput = (DirectExchangeInput) remoteSplit.getExchangeInput();
        client.addLocation(exchangeInput.getTaskId(), URI.create(exchangeInput.getLocation()));
        client.noMoreLocations();
        pageProducers.add(client.pages()
                .map(serializedPage -> {
                    Page page = deserializer.deserialize(serializedPage);
                    operatorContext.recordNetworkInput(serializedPage.length(), page.getPositionCount());
                    return page;
                }));
    }

    @Override
    public void noMoreSplits()
    {
        mergedPages = mergeSortedPages(
                pageProducers,
                comparator,
                outputChannels,
                outputTypes,
                (pageBuilder, pageWithPosition) -> pageBuilder.isFull(),
                false,
                operatorContext.aggregateUserMemoryContext(),
                operatorContext.getDriverContext().getYieldSignal());
        blockedOnSplits.set(null);
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        close();
    }

    @Override
    public boolean isFinished()
    {
        return closed || (mergedPages != null && mergedPages.isFinished());
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        if (!blockedOnSplits.isDone()) {
            return blockedOnSplits;
        }

        if (mergedPages.isBlocked()) {
            return mergedPages.getBlockedFuture();
        }

        return NOT_BLOCKED;
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException(getClass().getName() + " cannot take input");
    }

    @Override
    public Page getOutput()
    {
        if (closed || mergedPages == null || !mergedPages.process() || mergedPages.isFinished()) {
            return null;
        }

        Page page = mergedPages.getResult();
        operatorContext.recordProcessedInput(page.getSizeInBytes(), page.getPositionCount());
        return page;
    }

    @Override
    public void close()
    {
        try {
            closer.close();
            closed = true;
        }
        catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }
}
```



## OrderByOperator

```java
public class OrderByOperator
        implements Operator
{
    public static class OrderByOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Type> sourceTypes;
        private final List<Integer> outputChannels;
        private final int expectedPositions;
        private final List<Integer> sortChannels;
        private final List<SortOrder> sortOrder;
        private final PagesIndex.Factory pagesIndexFactory;
        private final boolean spillEnabled;
        private final Optional<SpillerFactory> spillerFactory;
        private final OrderingCompiler orderingCompiler;

        private boolean closed;

        public OrderByOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                List<? extends Type> sourceTypes,
                List<Integer> outputChannels,
                int expectedPositions,
                List<Integer> sortChannels,
                List<SortOrder> sortOrder,
                PagesIndex.Factory pagesIndexFactory,
                boolean spillEnabled,
                Optional<SpillerFactory> spillerFactory,
                OrderingCompiler orderingCompiler)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.sourceTypes = ImmutableList.copyOf(requireNonNull(sourceTypes, "sourceTypes is null"));
            this.outputChannels = requireNonNull(outputChannels, "outputChannels is null");
            this.expectedPositions = expectedPositions;
            this.sortChannels = ImmutableList.copyOf(requireNonNull(sortChannels, "sortChannels is null"));
            this.sortOrder = ImmutableList.copyOf(requireNonNull(sortOrder, "sortOrder is null"));

            this.pagesIndexFactory = requireNonNull(pagesIndexFactory, "pagesIndexFactory is null");
            this.spillEnabled = spillEnabled;
            this.spillerFactory = requireNonNull(spillerFactory, "spillerFactory is null");
            this.orderingCompiler = requireNonNull(orderingCompiler, "orderingCompiler is null");
            checkArgument(!spillEnabled || spillerFactory.isPresent(), "Spiller Factory is not present when spill is enabled");
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");

            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, OrderByOperator.class.getSimpleName());
            return new OrderByOperator(
                    operatorContext,
                    sourceTypes,
                    outputChannels,
                    expectedPositions,
                    sortChannels,
                    sortOrder,
                    pagesIndexFactory,
                    spillEnabled,
                    spillerFactory,
                    orderingCompiler);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new OrderByOperatorFactory(
                    operatorId,
                    planNodeId,
                    sourceTypes,
                    outputChannels,
                    expectedPositions,
                    sortChannels,
                    sortOrder,
                    pagesIndexFactory,
                    spillEnabled,
                    spillerFactory,
                    orderingCompiler);
        }
    }

    private enum State
    {
        NEEDS_INPUT,
        HAS_OUTPUT,
        FINISHED
    }

    private final OperatorContext operatorContext;
    private final List<Integer> sortChannels;
    private final List<SortOrder> sortOrder;
    private final int[] outputChannels;
    private final LocalMemoryContext revocableMemoryContext;
    private final LocalMemoryContext localUserMemoryContext;

    private final PagesIndex pageIndex;

    private final List<Type> sourceTypes;

    private final boolean spillEnabled;
    private final Optional<SpillerFactory> spillerFactory;
    private final OrderingCompiler orderingCompiler;

    private Optional<Spiller> spiller = Optional.empty();
    private ListenableFuture<Void> spillInProgress = immediateVoidFuture();
    private Runnable finishMemoryRevoke = () -> {};

    private Iterator<Optional<Page>> sortedPages;

    private State state = State.NEEDS_INPUT;

    public OrderByOperator(
            OperatorContext operatorContext,
            List<Type> sourceTypes,
            List<Integer> outputChannels,
            int expectedPositions,
            List<Integer> sortChannels,
            List<SortOrder> sortOrder,
            PagesIndex.Factory pagesIndexFactory,
            boolean spillEnabled,
            Optional<SpillerFactory> spillerFactory,
            OrderingCompiler orderingCompiler)
    {
        requireNonNull(pagesIndexFactory, "pagesIndexFactory is null");

        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.outputChannels = Ints.toArray(requireNonNull(outputChannels, "outputChannels is null"));
        this.sortChannels = ImmutableList.copyOf(requireNonNull(sortChannels, "sortChannels is null"));
        this.sortOrder = ImmutableList.copyOf(requireNonNull(sortOrder, "sortOrder is null"));
        this.sourceTypes = ImmutableList.copyOf(requireNonNull(sourceTypes, "sourceTypes is null"));
        this.localUserMemoryContext = operatorContext.localUserMemoryContext();
        this.revocableMemoryContext = operatorContext.localRevocableMemoryContext();

        this.pageIndex = pagesIndexFactory.newPagesIndex(sourceTypes, expectedPositions);
        this.spillEnabled = spillEnabled;
        this.spillerFactory = requireNonNull(spillerFactory, "spillerFactory is null");
        this.orderingCompiler = requireNonNull(orderingCompiler, "orderingCompiler is null");
        checkArgument(!spillEnabled || spillerFactory.isPresent(), "Spiller Factory is not present when spill is enabled");
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        if (!spillInProgress.isDone()) {
            return;
        }
        checkSuccess(spillInProgress, "spilling failed");

        if (state == State.NEEDS_INPUT) {
            state = State.HAS_OUTPUT;

            // Convert revocable memory to user memory as sortedPages holds on to memory so we no longer can revoke.
            if (revocableMemoryContext.getBytes() > 0) {
                long currentRevocableBytes = revocableMemoryContext.getBytes();
                revocableMemoryContext.setBytes(0);
                if (!localUserMemoryContext.trySetBytes(localUserMemoryContext.getBytes() + currentRevocableBytes)) {
                    // TODO: this might fail (even though we have just released memory), but we don't
                    // have a proper way to atomically convert memory reservations
                    revocableMemoryContext.setBytes(currentRevocableBytes);
                    // spill since revocable memory could not be converted to user memory immediately
                    // TODO: this should be asynchronous
                    getFutureValue(spillToDisk());
                    finishMemoryRevoke.run();
                }
            }

            pageIndex.sort(sortChannels, sortOrder);
            Iterator<Page> sortedPagesIndex = pageIndex.getSortedPages();

            List<WorkProcessor<Page>> spilledPages = getSpilledPages();
            if (spilledPages.isEmpty()) {
                sortedPages = transform(sortedPagesIndex, Optional::of);
            }
            else {
                sortedPages = mergeSpilledAndMemoryPages(spilledPages, sortedPagesIndex).yieldingIterator();
            }
        }
    }

    @Override
    public boolean isFinished()
    {
        return state == State.FINISHED;
    }

    @Override
    public boolean needsInput()
    {
        return state == State.NEEDS_INPUT;
    }

    @Override
    public void addInput(Page page)
    {
        checkState(state == State.NEEDS_INPUT, "Operator is already finishing");
        requireNonNull(page, "page is null");
        checkSuccess(spillInProgress, "spilling failed");

        // TODO: remove when retained memory accounting for pages does not
        // count shared data structures multiple times
        page.compact();
        pageIndex.addPage(page);
        updateMemoryUsage();
    }

    @Override
    public Page getOutput()
    {
        checkSuccess(spillInProgress, "spilling failed");
        if (state != State.HAS_OUTPUT) {
            return null;
        }

        verifyNotNull(sortedPages, "sortedPages is null");
        if (!sortedPages.hasNext()) {
            state = State.FINISHED;
            return null;
        }

        Optional<Page> next = sortedPages.next();
        if (next.isEmpty()) {
            return null;
        }
        Page nextPage = next.get();
        Block[] blocks = new Block[outputChannels.length];
        for (int i = 0; i < outputChannels.length; i++) {
            blocks[i] = nextPage.getBlock(outputChannels[i]);
        }
        return new Page(nextPage.getPositionCount(), blocks);
    }

    @Override
    public ListenableFuture<Void> startMemoryRevoke()
    {
        verify(state == State.NEEDS_INPUT || revocableMemoryContext.getBytes() == 0, "Cannot spill in state: %s", state);
        return spillToDisk();
    }

    private ListenableFuture<Void> spillToDisk()
    {
        checkSuccess(spillInProgress, "spilling failed");

        if (revocableMemoryContext.getBytes() == 0) {
            verify(pageIndex.getPositionCount() == 0 || state == State.HAS_OUTPUT);
            finishMemoryRevoke = () -> {};
            return immediateVoidFuture();
        }

        // TODO try pageIndex.compact(); before spilling, as in HashBuilderOperator.startMemoryRevoke()

        if (spiller.isEmpty()) {
            spiller = Optional.of(spillerFactory.get().create(
                    sourceTypes,
                    operatorContext.getSpillContext(),
                    operatorContext.newAggregateUserMemoryContext()));
        }

        pageIndex.sort(sortChannels, sortOrder);
        spillInProgress = spiller.get().spill(pageIndex.getSortedPages());
        finishMemoryRevoke = () -> {
            pageIndex.clear();
            updateMemoryUsage();
        };

        return spillInProgress;
    }

    @Override
    public void finishMemoryRevoke()
    {
        finishMemoryRevoke.run();
        finishMemoryRevoke = () -> {};
    }

    private List<WorkProcessor<Page>> getSpilledPages()
    {
        if (spiller.isEmpty()) {
            return ImmutableList.of();
        }

        return spiller.get().getSpills().stream()
                .map(WorkProcessor::fromIterator)
                .collect(toImmutableList());
    }

    private WorkProcessor<Page> mergeSpilledAndMemoryPages(List<WorkProcessor<Page>> spilledPages, Iterator<Page> sortedPagesIndex)
    {
        List<WorkProcessor<Page>> sortedStreams = ImmutableList.<WorkProcessor<Page>>builder()
                .addAll(spilledPages)
                .add(WorkProcessor.fromIterator(sortedPagesIndex))
                .build();

        return mergeSortedPages(
                sortedStreams,
                orderingCompiler.compilePageWithPositionComparator(sourceTypes, sortChannels, sortOrder),
                sourceTypes,
                operatorContext.aggregateUserMemoryContext(),
                operatorContext.getDriverContext().getYieldSignal());
    }

    private void updateMemoryUsage()
    {
        if (spillEnabled && state == State.NEEDS_INPUT) {
            if (pageIndex.getPositionCount() == 0) {
                localUserMemoryContext.setBytes(pageIndex.getEstimatedSize().toBytes());
                revocableMemoryContext.setBytes(0L);
            }
            else {
                localUserMemoryContext.setBytes(0);
                revocableMemoryContext.setBytes(pageIndex.getEstimatedSize().toBytes());
            }
        }
        else {
            revocableMemoryContext.setBytes(0);
            if (!localUserMemoryContext.trySetBytes(pageIndex.getEstimatedSize().toBytes())) {
                pageIndex.compact();
                localUserMemoryContext.setBytes(pageIndex.getEstimatedSize().toBytes());
            }
        }
    }

    @Override
    public void close()
    {
        pageIndex.clear();
        sortedPages = null;
        spiller.ifPresent(Spiller::close);
    }
}
```



## AggregationOperator

```java
/**
 * Group input data and produce a single block for each sequence of identical values.
 */
public class AggregationOperator
        implements Operator
{
    public static class AggregationOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<AggregatorFactory> aggregatorFactories;
        private boolean closed;

        public AggregationOperatorFactory(int operatorId, PlanNodeId planNodeId, List<AggregatorFactory> aggregatorFactories)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.aggregatorFactories = ImmutableList.copyOf(aggregatorFactories);
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, AggregationOperator.class.getSimpleName());
            return new AggregationOperator(operatorContext, aggregatorFactories);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new AggregationOperatorFactory(operatorId, planNodeId, aggregatorFactories);
        }
    }

    private enum State
    {
        NEEDS_INPUT,
        HAS_OUTPUT,
        FINISHED
    }

    private final OperatorContext operatorContext;
    private final LocalMemoryContext userMemoryContext;
    private final List<Aggregator> aggregates;

    private State state = State.NEEDS_INPUT;

    public AggregationOperator(OperatorContext operatorContext, List<AggregatorFactory> aggregatorFactories)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.userMemoryContext = operatorContext.localUserMemoryContext();

        aggregates = aggregatorFactories.stream()
                .map(AggregatorFactory::createAggregator)
                .collect(toImmutableList());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        if (state == State.NEEDS_INPUT) {
            state = State.HAS_OUTPUT;
        }
    }

    @Override
    public void close()
    {
        userMemoryContext.setBytes(0);
    }

    @Override
    public boolean isFinished()
    {
        return state == State.FINISHED;
    }

    @Override
    public boolean needsInput()
    {
        return state == State.NEEDS_INPUT;
    }

    @Override
    public void addInput(Page page)
    {
        checkState(needsInput(), "Operator is already finishing");
        requireNonNull(page, "page is null");

        long memorySize = 0;
        for (Aggregator aggregate : aggregates) {
            aggregate.processPage(page);
            memorySize += aggregate.getEstimatedSize();
        }
        userMemoryContext.setBytes(memorySize);
    }

    @Override
    public Page getOutput()
    {
        if (state != State.HAS_OUTPUT) {
            return null;
        }

        // project results into output blocks
        List<Type> types = aggregates.stream().map(Aggregator::getType).collect(toImmutableList());

        // output page will only be constructed once,
        // so a new PageBuilder is constructed (instead of using PageBuilder.reset)
        PageBuilder pageBuilder = new PageBuilder(1, types);

        pageBuilder.declarePosition();
        for (int i = 0; i < aggregates.size(); i++) {
            Aggregator aggregator = aggregates.get(i);
            BlockBuilder blockBuilder = pageBuilder.getBlockBuilder(i);
            aggregator.evaluate(blockBuilder);
        }

        state = State.FINISHED;
        return pageBuilder.build();
    }
}
```



## ValuesOperator

```java
public class ValuesOperator
        implements Operator
{
    public static class ValuesOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Page> pages;
        private boolean closed;

        public ValuesOperatorFactory(int operatorId, PlanNodeId planNodeId, List<Page> pages)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.pages = ImmutableList.copyOf(requireNonNull(pages, "pages is null"));
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, ValuesOperator.class.getSimpleName());
            return new ValuesOperator(operatorContext, pages);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new ValuesOperatorFactory(operatorId, planNodeId, pages);
        }
    }

    private final OperatorContext operatorContext;
    private final Iterator<Page> pages;

    public ValuesOperator(OperatorContext operatorContext, List<Page> pages)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");

        requireNonNull(pages, "pages is null");

        this.pages = ImmutableList.copyOf(pages).iterator();
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        int ignored = Iterators.size(pages);
    }

    @Override
    public boolean isFinished()
    {
        return !pages.hasNext();
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException();
    }

    @Override
    public Page getOutput()
    {
        if (!pages.hasNext()) {
            return null;
        }
        Page page = pages.next();
        if (page != null) {
            operatorContext.recordProcessedInput(page.getSizeInBytes(), page.getPositionCount());
        }
        return page;
    }
}
```



## StreamingAggregationOperator

```java
public class StreamingAggregationOperator
        implements WorkProcessorOperator
{
    public static OperatorFactory createOperatorFactory(
            int operatorId,
            PlanNodeId planNodeId,
            List<Type> sourceTypes,
            List<Type> groupByTypes,
            List<Integer> groupByChannels,
            List<AggregatorFactory> aggregatorFactories,
            JoinCompiler joinCompiler)
    {
        return createAdapterOperatorFactory(new Factory(
                operatorId,
                planNodeId,
                sourceTypes,
                groupByTypes,
                groupByChannels,
                aggregatorFactories,
                joinCompiler));
    }

    private static class Factory
            implements BasicAdapterWorkProcessorOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final List<Type> sourceTypes;
        private final List<Type> groupByTypes;
        private final List<Integer> groupByChannels;
        private final List<AggregatorFactory> aggregatorFactories;
        private final JoinCompiler joinCompiler;
        private boolean closed;

        private Factory(
                int operatorId,
                PlanNodeId planNodeId,
                List<Type> sourceTypes,
                List<Type> groupByTypes,
                List<Integer> groupByChannels,
                List<AggregatorFactory> aggregatorFactories,
                JoinCompiler joinCompiler)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.sourceTypes = ImmutableList.copyOf(requireNonNull(sourceTypes, "sourceTypes is null"));
            this.groupByTypes = ImmutableList.copyOf(requireNonNull(groupByTypes, "groupByTypes is null"));
            this.groupByChannels = ImmutableList.copyOf(requireNonNull(groupByChannels, "groupByChannels is null"));
            this.aggregatorFactories = ImmutableList.copyOf(requireNonNull(aggregatorFactories, "aggregatorFactories is null"));
            this.joinCompiler = requireNonNull(joinCompiler, "joinCompiler is null");
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return planNodeId;
        }

        @Override
        public String getOperatorType()
        {
            return StreamingAggregationOperator.class.getSimpleName();
        }

        @Override
        public WorkProcessorOperator create(ProcessorContext processorContext, WorkProcessor<Page> sourcePages)
        {
            checkState(!closed, "Factory is already closed");
            return new StreamingAggregationOperator(processorContext, sourcePages, sourceTypes, groupByTypes, groupByChannels, aggregatorFactories, joinCompiler);
        }

        @Override
        public void close()
        {
            closed = true;
        }

        @Override
        public Factory duplicate()
        {
            return new Factory(operatorId, planNodeId, sourceTypes, groupByTypes, groupByChannels, aggregatorFactories, joinCompiler);
        }
    }

    private final WorkProcessor<Page> pages;

    private StreamingAggregationOperator(
            ProcessorContext processorContext,
            WorkProcessor<Page> sourcePages,
            List<Type> sourceTypes,
            List<Type> groupByTypes,
            List<Integer> groupByChannels,
            List<AggregatorFactory> aggregatorFactories,
            JoinCompiler joinCompiler)
    {
        pages = sourcePages
                .transform(new StreamingAggregation(
                        processorContext,
                        sourceTypes,
                        groupByTypes,
                        groupByChannels,
                        aggregatorFactories,
                        joinCompiler));
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    private static class StreamingAggregation
            implements Transformation<Page, Page>
    {
        private final LocalMemoryContext userMemoryContext;
        private final List<Type> groupByTypes;
        private final int[] groupByChannels;
        private final List<AggregatorFactory> aggregatorFactories;
        private final PagesHashStrategy pagesHashStrategy;

        private List<Aggregator> aggregates;
        private final PageBuilder pageBuilder;
        private final Deque<Page> outputPages = new LinkedList<>();
        private Page currentGroup;

        private StreamingAggregation(
                ProcessorContext processorContext,
                List<Type> sourceTypes,
                List<Type> groupByTypes,
                List<Integer> groupByChannels,
                List<AggregatorFactory> aggregatorFactories,
                JoinCompiler joinCompiler)
        {
            requireNonNull(processorContext, "processorContext is null");
            this.userMemoryContext = processorContext.getMemoryTrackingContext().localUserMemoryContext();
            this.groupByTypes = ImmutableList.copyOf(requireNonNull(groupByTypes, "groupByTypes is null"));
            this.groupByChannels = Ints.toArray(requireNonNull(groupByChannels, "groupByChannels is null"));
            this.aggregatorFactories = requireNonNull(aggregatorFactories, "aggregatorFactories is null");

            this.aggregates = aggregatorFactories.stream()
                    .map(AggregatorFactory::createAggregator)
                    .collect(toImmutableList());
            this.pageBuilder = new PageBuilder(toTypes(groupByTypes, aggregates));
            requireNonNull(joinCompiler, "joinCompiler is null");

            requireNonNull(sourceTypes, "sourceTypes is null");
            pagesHashStrategy = joinCompiler.compilePagesHashStrategyFactory(sourceTypes, groupByChannels, Optional.empty())
                    .createPagesHashStrategy(
                            sourceTypes.stream()
                                    .map(type -> new ObjectArrayList<Block>())
                                    .collect(toImmutableList()), OptionalInt.empty());
        }

        @Override
        public TransformationState<Page> process(@Nullable Page inputPage)
        {
            if (inputPage == null) {
                if (currentGroup != null) {
                    evaluateAndFlushGroup(currentGroup, 0);
                    currentGroup = null;
                }

                if (!pageBuilder.isEmpty()) {
                    outputPages.add(pageBuilder.build());
                    pageBuilder.reset();
                }

                if (outputPages.isEmpty()) {
                    return finished();
                }

                return ofResult(outputPages.removeFirst(), false);
            }

            // flush remaining output pages before requesting next input page
            // invariant: queued output pages are produced from argument `inputPage`
            if (!outputPages.isEmpty()) {
                Page outputPage = outputPages.removeFirst();
                return ofResult(outputPage, outputPages.isEmpty());
            }

            processInput(inputPage);
            updateMemoryUsage();

            if (outputPages.isEmpty()) {
                return needsMoreData();
            }

            Page outputPage = outputPages.removeFirst();
            return ofResult(outputPage, outputPages.isEmpty());
        }

        private void updateMemoryUsage()
        {
            long memorySize = pageBuilder.getRetainedSizeInBytes();
            for (Page output : outputPages) {
                memorySize += output.getRetainedSizeInBytes();
            }
            for (Aggregator aggregator : aggregates) {
                memorySize += aggregator.getEstimatedSize();
            }

            if (currentGroup != null) {
                memorySize += currentGroup.getRetainedSizeInBytes();
            }

            userMemoryContext.setBytes(memorySize);
        }

        private void processInput(Page page)
        {
            requireNonNull(page, "page is null");

            Page groupByPage = page.getColumns(groupByChannels);
            if (currentGroup != null) {
                if (!pagesHashStrategy.rowNotDistinctFromRow(0, currentGroup.getColumns(groupByChannels), 0, groupByPage)) {
                    // page starts with new group, so flush it
                    evaluateAndFlushGroup(currentGroup, 0);
                }
                currentGroup = null;
            }

            int startPosition = 0;
            while (true) {
                // may be equal to page.getPositionCount() if the end is not found in this page
                int nextGroupStart = findNextGroupStart(startPosition, groupByPage);
                addRowsToAggregates(page, startPosition, nextGroupStart - 1);

                if (nextGroupStart < page.getPositionCount()) {
                    // current group stops somewhere in the middle of the page, so flush it
                    evaluateAndFlushGroup(page, startPosition);
                    startPosition = nextGroupStart;
                }
                else {
                    // late materialization requires that page being locally stored is materialized before the next one is fetched
                    currentGroup = page.getRegion(page.getPositionCount() - 1, 1).getLoadedPage();
                    return;
                }
            }
        }

        private void addRowsToAggregates(Page page, int startPosition, int endPosition)
        {
            Page region = page.getRegion(startPosition, endPosition - startPosition + 1);
            for (Aggregator aggregator : aggregates) {
                aggregator.processPage(region);
            }
        }

        private void evaluateAndFlushGroup(Page page, int position)
        {
            pageBuilder.declarePosition();
            for (int i = 0; i < groupByTypes.size(); i++) {
                Block block = page.getBlock(groupByChannels[i]);
                Type type = groupByTypes.get(i);
                type.appendTo(block, position, pageBuilder.getBlockBuilder(i));
            }
            int offset = groupByTypes.size();
            for (int i = 0; i < aggregates.size(); i++) {
                aggregates.get(i).evaluate(pageBuilder.getBlockBuilder(offset + i));
            }

            if (pageBuilder.isFull()) {
                outputPages.add(pageBuilder.build());
                pageBuilder.reset();
            }

            aggregates = aggregatorFactories.stream()
                    .map(AggregatorFactory::createAggregator)
                    .collect(toImmutableList());
        }

        private int findNextGroupStart(int startPosition, Page page)
        {
            for (int i = startPosition + 1; i < page.getPositionCount(); i++) {
                if (!pagesHashStrategy.rowNotDistinctFromRow(startPosition, page, i, page)) {
                    return i;
                }
            }

            return page.getPositionCount();
        }

        private static List<Type> toTypes(List<Type> groupByTypes, List<Aggregator> aggregates)
        {
            ImmutableList.Builder<Type> builder = ImmutableList.builder();
            builder.addAll(groupByTypes);
            aggregates.stream()
                    .map(Aggregator::getType)
                    .forEach(builder::add);
            return builder.build();
        }
    }
}
```



## HashSemiJoinOperator

```java
public class HashSemiJoinOperator
        implements WorkProcessorOperator
{
    public static OperatorFactory createOperatorFactory(
            int operatorId,
            PlanNodeId planNodeId,
            SetSupplier setSupplier,
            List<? extends Type> probeTypes,
            int probeJoinChannel,
            Optional<Integer> probeJoinHashChannel)
    {
        return createAdapterOperatorFactory(new Factory(operatorId, planNodeId, setSupplier, probeTypes, probeJoinChannel, probeJoinHashChannel));
    }

    private static class Factory
            implements BasicAdapterWorkProcessorOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final SetSupplier setSupplier;
        private final List<Type> probeTypes;
        private final int probeJoinChannel;
        private final Optional<Integer> probeJoinHashChannel;
        private boolean closed;

        private Factory(int operatorId, PlanNodeId planNodeId, SetSupplier setSupplier, List<? extends Type> probeTypes, int probeJoinChannel, Optional<Integer> probeJoinHashChannel)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            this.setSupplier = setSupplier;
            this.probeTypes = ImmutableList.copyOf(probeTypes);
            checkArgument(probeJoinChannel >= 0, "probeJoinChannel is negative");
            this.probeJoinChannel = probeJoinChannel;
            this.probeJoinHashChannel = probeJoinHashChannel;
        }

        @Override
        public WorkProcessorOperator create(ProcessorContext processorContext, WorkProcessor<Page> sourcePages)
        {
            checkState(!closed, "Factory is already closed");
            return new HashSemiJoinOperator(sourcePages, setSupplier, probeJoinChannel, probeJoinHashChannel, processorContext.getMemoryTrackingContext());
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return planNodeId;
        }

        @Override
        public String getOperatorType()
        {
            return HashSemiJoinOperator.class.getSimpleName();
        }

        @Override
        public void close()
        {
            closed = true;
        }

        @Override
        public Factory duplicate()
        {
            return new Factory(operatorId, planNodeId, setSupplier, probeTypes, probeJoinChannel, probeJoinHashChannel);
        }
    }

    private final WorkProcessor<Page> pages;

    private HashSemiJoinOperator(
            WorkProcessor<Page> sourcePages,
            SetSupplier channelSetFuture,
            int probeJoinChannel,
            Optional<Integer> probeHashChannel,
            MemoryTrackingContext memoryTrackingContext)
    {
        pages = sourcePages
                .transform(new SemiJoinPages(
                        channelSetFuture,
                        probeJoinChannel,
                        probeHashChannel,
                        memoryTrackingContext.aggregateUserMemoryContext()));
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    private static class SemiJoinPages
            implements WorkProcessor.Transformation<Page, Page>
    {
        private static final int NO_PRECOMPUTED_HASH_CHANNEL = -1;

        private final int probeJoinChannel;
        private final int probeHashChannel; // when >= 0, this is the precomputed hash channel
        private final ListenableFuture<ChannelSet> channelSetFuture;
        private final LocalMemoryContext localMemoryContext;

        @Nullable
        private ChannelSet channelSet;

        public SemiJoinPages(SetSupplier channelSetFuture, int probeJoinChannel, Optional<Integer> probeHashChannel, AggregatedMemoryContext aggregatedMemoryContext)
        {
            checkArgument(probeJoinChannel >= 0, "probeJoinChannel is negative");

            this.channelSetFuture = channelSetFuture.getChannelSet();
            this.probeJoinChannel = probeJoinChannel;
            this.probeHashChannel = probeHashChannel.orElse(NO_PRECOMPUTED_HASH_CHANNEL);
            this.localMemoryContext = aggregatedMemoryContext.newLocalMemoryContext(SemiJoinPages.class.getSimpleName());
        }

        @Override
        public TransformationState<Page> process(Page inputPage)
        {
            if (inputPage == null) {
                return finished();
            }

            if (channelSet == null) {
                if (!channelSetFuture.isDone()) {
                    // This will materialize page but it shouldn't matter for the first page
                    localMemoryContext.setBytes(inputPage.getSizeInBytes());
                    return blocked(asVoid(channelSetFuture));
                }
                checkSuccess(channelSetFuture, "ChannelSet building failed");
                channelSet = getFutureValue(channelSetFuture);
                localMemoryContext.setBytes(0);
            }
            // use an effectively-final local variable instead of the non-final instance field inside of the loop
            ChannelSet channelSet = requireNonNull(this.channelSet, "channelSet is null");

            // create the block builder for the new boolean column
            // we know the exact size required for the block
            BlockBuilder blockBuilder = BOOLEAN.createFixedSizeBlockBuilder(inputPage.getPositionCount());

            Page probeJoinPage = inputPage.getLoadedPage(probeJoinChannel);
            Block probeJoinNulls = probeJoinPage.getBlock(0).mayHaveNull() ? probeJoinPage.getBlock(0) : null;
            Block hashBlock = probeHashChannel >= 0 ? inputPage.getBlock(probeHashChannel) : null;

            // update hashing strategy to use probe cursor
            for (int position = 0; position < inputPage.getPositionCount(); position++) {
                if (probeJoinNulls != null && probeJoinNulls.isNull(position)) {
                    if (channelSet.isEmpty()) {
                        BOOLEAN.writeBoolean(blockBuilder, false);
                    }
                    else {
                        blockBuilder.appendNull();
                    }
                }
                else {
                    boolean contains;
                    if (hashBlock != null) {
                        long rawHash = BIGINT.getLong(hashBlock, position);
                        contains = channelSet.contains(position, probeJoinPage, rawHash);
                    }
                    else {
                        contains = channelSet.contains(position, probeJoinPage);
                    }
                    if (!contains && channelSet.containsNull()) {
                        blockBuilder.appendNull();
                    }
                    else {
                        BOOLEAN.writeBoolean(blockBuilder, contains);
                    }
                }
            }
            // add the new boolean column to the page
            return ofResult(inputPage.appendColumn(blockBuilder.build()));
        }
    }

    private static <T> ListenableFuture<Void> asVoid(ListenableFuture<T> future)
    {
        return Futures.transform(future, v -> null, directExecutor());
    }
}
```



## SetBuilderOperator

```java
@ThreadSafe
public class SetBuilderOperator
        implements Operator
{
    public static class SetSupplier
    {
        private final Type type;
        private final SettableFuture<ChannelSet> channelSetFuture = SettableFuture.create();

        public SetSupplier(Type type)
        {
            this.type = requireNonNull(type, "type is null");
        }

        public Type getType()
        {
            return type;
        }

        public ListenableFuture<ChannelSet> getChannelSet()
        {
            return channelSetFuture;
        }

        void setChannelSet(ChannelSet channelSet)
        {
            boolean wasSet = channelSetFuture.set(requireNonNull(channelSet, "channelSet is null"));
            checkState(wasSet, "ChannelSet already set");
        }
    }

    public static class SetBuilderOperatorFactory
            implements OperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private final Optional<Integer> hashChannel;
        private final SetSupplier setProvider;
        private final int setChannel;
        private final int expectedPositions;
        private boolean closed;
        private final JoinCompiler joinCompiler;
        private final BlockTypeOperators blockTypeOperators;

        public SetBuilderOperatorFactory(
                int operatorId,
                PlanNodeId planNodeId,
                Type type,
                int setChannel,
                Optional<Integer> hashChannel,
                int expectedPositions,
                JoinCompiler joinCompiler,
                BlockTypeOperators blockTypeOperators)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
            checkArgument(setChannel >= 0, "setChannel is negative");
            this.setProvider = new SetSupplier(requireNonNull(type, "type is null"));
            this.setChannel = setChannel;
            this.hashChannel = requireNonNull(hashChannel, "hashChannel is null");
            this.expectedPositions = expectedPositions;
            this.joinCompiler = requireNonNull(joinCompiler, "joinCompiler is null");
            this.blockTypeOperators = requireNonNull(blockTypeOperators, "blockTypeOperators is null");
        }

        public SetSupplier getSetProvider()
        {
            return setProvider;
        }

        @Override
        public Operator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, planNodeId, SetBuilderOperator.class.getSimpleName());
            return new SetBuilderOperator(operatorContext, setProvider, setChannel, hashChannel, expectedPositions, joinCompiler, blockTypeOperators);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }

        @Override
        public OperatorFactory duplicate()
        {
            return new SetBuilderOperatorFactory(operatorId, planNodeId, setProvider.getType(), setChannel, hashChannel, expectedPositions, joinCompiler, blockTypeOperators);
        }
    }

    private final OperatorContext operatorContext;
    private final SetSupplier setSupplier;
    private final int[] sourceChannels;

    private final ChannelSetBuilder channelSetBuilder;

    private boolean finished;

    @Nullable
    private Work<?> unfinishedWork;  // The pending work for current page.

    public SetBuilderOperator(
            OperatorContext operatorContext,
            SetSupplier setSupplier,
            int setChannel,
            Optional<Integer> hashChannel,
            int expectedPositions,
            JoinCompiler joinCompiler,
            BlockTypeOperators blockTypeOperators)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.setSupplier = requireNonNull(setSupplier, "setSupplier is null");

        if (hashChannel.isPresent()) {
            this.sourceChannels = new int[] {setChannel, hashChannel.get()};
        }
        else {
            this.sourceChannels = new int[] {setChannel};
        }
        // Set builder is has a single channel which goes in channel 0, if hash is present, add a hachBlock to channel 1
        Optional<Integer> channelSetHashChannel = hashChannel.isPresent() ? Optional.of(1) : Optional.empty();
        this.channelSetBuilder = new ChannelSetBuilder(
                setSupplier.getType(),
                channelSetHashChannel,
                expectedPositions,
                requireNonNull(operatorContext, "operatorContext is null"),
                requireNonNull(joinCompiler, "joinCompiler is null"),
                requireNonNull(blockTypeOperators, "blockTypeOperators is null"));
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public void finish()
    {
        if (finished) {
            return;
        }

        ChannelSet channelSet = channelSetBuilder.build();
        setSupplier.setChannelSet(channelSet);
        operatorContext.recordOutput(channelSet.getEstimatedSizeInBytes(), channelSet.size());
        finished = true;
    }

    @Override
    public boolean isFinished()
    {
        return finished;
    }

    @Override
    public boolean needsInput()
    {
        // Since SetBuilderOperator doesn't produce any output, the getOutput()
        // method may never be called. We need to handle any unfinished work
        // before addInput() can be called again.
        return !finished && (unfinishedWork == null || processUnfinishedWork());
    }

    @Override
    public void addInput(Page page)
    {
        requireNonNull(page, "page is null");
        checkState(!isFinished(), "Operator is already finished");

        unfinishedWork = channelSetBuilder.addPage(page.getColumns(sourceChannels));
        processUnfinishedWork();
    }

    @Override
    public Page getOutput()
    {
        return null;
    }

    private boolean processUnfinishedWork()
    {
        // Processes the unfinishedWork for this page by adding the data to the hash table. If this page
        // can't be fully consumed (e.g. rehashing fails), the unfinishedWork will be left with non-empty value.
        checkState(unfinishedWork != null, "unfinishedWork is empty");
        boolean done = unfinishedWork.process();
        if (done) {
            unfinishedWork = null;
        }
        // We need to update the memory reservation again since the page builder memory may also be increasing.
        channelSetBuilder.updateMemoryReservation();
        return done;
    }

    @VisibleForTesting
    public int getCapacity()
    {
        return channelSetBuilder.getCapacity();
    }
}
```



## TableScanOperator

```java
public class TableScanOperator
        implements SourceOperator
{
    public static class TableScanOperatorFactory
            implements SourceOperatorFactory, WorkProcessorSourceOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId sourceId;
        private final PageSourceProvider pageSourceProvider;
        private final TableHandle table;
        private final List<ColumnHandle> columns;
        private final DynamicFilter dynamicFilter;
        private boolean closed;

        public TableScanOperatorFactory(
                int operatorId,
                PlanNodeId sourceId,
                PageSourceProvider pageSourceProvider,
                TableHandle table,
                Iterable<ColumnHandle> columns,
                DynamicFilter dynamicFilter)
        {
            this.operatorId = operatorId;
            this.sourceId = requireNonNull(sourceId, "sourceId is null");
            this.pageSourceProvider = requireNonNull(pageSourceProvider, "pageSourceProvider is null");
            this.table = requireNonNull(table, "table is null");
            this.columns = ImmutableList.copyOf(requireNonNull(columns, "columns is null"));
            this.dynamicFilter = requireNonNull(dynamicFilter, "dynamicFilter is null");
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getSourceId()
        {
            return sourceId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return sourceId;
        }

        @Override
        public String getOperatorType()
        {
            return TableScanOperator.class.getSimpleName();
        }

        @Override
        public SourceOperator createOperator(DriverContext driverContext)
        {
            checkState(!closed, "Factory is already closed");
            OperatorContext operatorContext = driverContext.addOperatorContext(operatorId, sourceId, getOperatorType());
            return new TableScanOperator(
                    operatorContext,
                    sourceId,
                    pageSourceProvider,
                    table,
                    columns,
                    dynamicFilter);
        }

        @Override
        public WorkProcessorSourceOperator create(
                Session session,
                MemoryTrackingContext memoryTrackingContext,
                DriverYieldSignal yieldSignal,
                WorkProcessor<Split> splits)
        {
            return new TableScanWorkProcessorOperator(
                    session,
                    memoryTrackingContext,
                    splits,
                    pageSourceProvider,
                    table,
                    columns,
                    dynamicFilter);
        }

        @Override
        public void noMoreOperators()
        {
            closed = true;
        }
    }
		
    //关于算子执行的所有信息
    private final OperatorContext operatorContext;
    //执行计划树中该节点的ID号
    private final PlanNodeId planNodeId;
    //实现类是PageSourceManager，可根据catalog类别创建对应的ConnectorPageSource
    private final PageSourceProvider pageSourceProvider;
    //表的信息，包括其catalog、schema、partitions、columns等
    private final TableHandle table;
    //列信息，包括列名、列索引、列类型等
    private final List<ColumnHandle> columns;
    //动态过滤信息
    private final DynamicFilter dynamicFilter;
    //该算子使用的内存信息等
    private final LocalMemoryContext memoryContext;
    private final SettableFuture<Void> blocked = SettableFuture.create();

  	//该算子处理的数据切片split信息
    @Nullable
    private Split split;
    //数据源信息，可由pageSourceProvider创建
    @Nullable
    private ConnectorPageSource source;
    //算子是否已经处理结束
    private boolean finished;
    //算子读取到的数据大小
    private long completedBytes;
    //
    private long completedPositions;
    //算子读取的时间（纳秒）
    private long readTimeNanos;

    public TableScanOperator(
            OperatorContext operatorContext,
            PlanNodeId planNodeId,
            PageSourceProvider pageSourceProvider,
            TableHandle table,
            Iterable<ColumnHandle> columns,
            DynamicFilter dynamicFilter)
    {
        this.operatorContext = requireNonNull(operatorContext, "operatorContext is null");
        this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
        this.pageSourceProvider = requireNonNull(pageSourceProvider, "pageSourceProvider is null");
        this.table = requireNonNull(table, "table is null");
        this.columns = ImmutableList.copyOf(requireNonNull(columns, "columns is null"));
        this.dynamicFilter = requireNonNull(dynamicFilter, "dynamicFilter is null");
        this.memoryContext = operatorContext.newLocalUserMemoryContext(TableScanOperator.class.getSimpleName());
    }

    @Override
    public OperatorContext getOperatorContext()
    {
        return operatorContext;
    }

    @Override
    public PlanNodeId getSourceId()
    {
        return planNodeId;
    }

    //添加需要处理的split
    @Override
    public void addSplit(Split split)
    {
        requireNonNull(split, "split is null");
        checkState(this.split == null, "Table scan split already set");

        if (finished) {
            return;
        }

        this.split = split;

        Object splitInfo = split.getInfo();
        if (splitInfo != null) {
            operatorContext.setInfoSupplier(Suppliers.ofInstance(new SplitOperatorInfo(split.getCatalogHandle(), splitInfo)));
        }

        blocked.set(null);

        if (split.getConnectorSplit() instanceof EmptySplit) {
            source = new EmptyPageSource();
        }
    }

    @Override
    public void noMoreSplits()
    {
        if (split == null) {
            finished = true;
        }
        blocked.set(null);
    }

    @Override
    public void close()
    {
        finish();
    }

    @Override
    public void finish()
    {
        finished = true;
        blocked.set(null);

        if (source != null) {
            try {
                source.close();
            }
            catch (IOException e) {
                throw new UncheckedIOException(e);
            }
            memoryContext.setBytes(source.getMemoryUsage());
            operatorContext.setLatestConnectorMetrics(source.getMetrics());
        }
    }

    @Override
    public boolean isFinished()
    {
        if (!finished) {
            finished = (source != null) && source.isFinished();
            if (source != null) {
                memoryContext.setBytes(source.getMemoryUsage());
            }
        }

        return finished;
    }

    @Override
    public ListenableFuture<Void> isBlocked()
    {
        if (!blocked.isDone()) {
            return blocked;
        }
        if (source != null) {
            CompletableFuture<?> pageSourceBlocked = source.isBlocked();
            return pageSourceBlocked.isDone() ? NOT_BLOCKED : asVoid(toListenableFuture(pageSourceBlocked));
        }
        return NOT_BLOCKED;
    }

    private static <T> ListenableFuture<Void> asVoid(ListenableFuture<T> future)
    {
        return Futures.transform(future, v -> null, directExecutor());
    }

    @Override
    public boolean needsInput()
    {
        return false;
    }

    @Override
    public void addInput(Page page)
    {
        throw new UnsupportedOperationException(getClass().getName() + " cannot take input");
    }
    
    //该算子的输出
    @Override
    public Page getOutput()
    {
        if (split == null) {
            return null;
        }
        if (source == null) {
            if (!dynamicFilter.getCurrentPredicate().isAll()) {
                operatorContext.recordDynamicFilterSplitProcessed(1L);
            }
            source = pageSourceProvider.createPageSource(operatorContext.getSession(), split, table, columns, dynamicFilter);
        }

        Page page = source.getNextPage();
        if (page != null) {
            // assure the page is in memory before handing to another operator
            page = page.getLoadedPage();

            // update operator stats
            long endCompletedBytes = source.getCompletedBytes();
            long endReadTimeNanos = source.getReadTimeNanos();
            long positionCount = page.getPositionCount();
            long endCompletedPositions = source.getCompletedPositions().orElse(completedPositions + positionCount);
            operatorContext.recordPhysicalInputWithTiming(
                    endCompletedBytes - completedBytes,
                    endCompletedPositions - completedPositions,
                    endReadTimeNanos - readTimeNanos);
            operatorContext.recordProcessedInput(page.getSizeInBytes(), positionCount);
            completedBytes = endCompletedBytes;
            completedPositions = endCompletedPositions;
            readTimeNanos = endReadTimeNanos;
        }

        // updating memory usage should happen after page is loaded.
        memoryContext.setBytes(source.getMemoryUsage());
        operatorContext.setLatestConnectorMetrics(source.getMetrics());
        return page;
    }
}
```



## AssignUniqueIdOperator

```java
public class AssignUniqueIdOperator
        implements WorkProcessorOperator
{
    private static final long ROW_IDS_PER_REQUEST = 1L << 20L;
    private static final long MAX_ROW_ID = 1L << 40L;

    public static OperatorFactory createOperatorFactory(int operatorId, PlanNodeId planNodeId)
    {
        return createAdapterOperatorFactory(new Factory(operatorId, planNodeId));
    }

    private static class Factory
            implements BasicAdapterWorkProcessorOperatorFactory
    {
        private final int operatorId;
        private final PlanNodeId planNodeId;
        private boolean closed;
        private final AtomicLong valuePool = new AtomicLong();

        private Factory(int operatorId, PlanNodeId planNodeId)
        {
            this.operatorId = operatorId;
            this.planNodeId = requireNonNull(planNodeId, "planNodeId is null");
        }

        @Override
        public WorkProcessorOperator create(ProcessorContext processorContext, WorkProcessor<Page> sourcePages)
        {
            checkState(!closed, "Factory is already closed");
            return new AssignUniqueIdOperator(processorContext, sourcePages, valuePool);
        }

        @Override
        public int getOperatorId()
        {
            return operatorId;
        }

        @Override
        public PlanNodeId getPlanNodeId()
        {
            return planNodeId;
        }

        @Override
        public String getOperatorType()
        {
            return AssignUniqueIdOperator.class.getSimpleName();
        }

        @Override
        public void close()
        {
            closed = true;
        }

        @Override
        public Factory duplicate()
        {
            return new Factory(operatorId, planNodeId);
        }
    }

    private final WorkProcessor<Page> pages;

    private AssignUniqueIdOperator(ProcessorContext context, WorkProcessor<Page> sourcePages, AtomicLong rowIdPool)
    {
        pages = sourcePages
                .transform(new AssignUniqueId(
                        context.getTaskId(),
                        rowIdPool));
    }

    @Override
    public WorkProcessor<Page> getOutputPages()
    {
        return pages;
    }

    private static class AssignUniqueId
            implements WorkProcessor.Transformation<Page, Page>
    {
        private final AtomicLong rowIdPool;
        private final long uniqueValueMask;

        private long rowIdCounter;
        private long maxRowIdCounterValue;

        private AssignUniqueId(TaskId taskId, AtomicLong rowIdPool)
        {
            this.rowIdPool = requireNonNull(rowIdPool, "rowIdPool is null");

            uniqueValueMask = (((long) taskId.getStageId().getId()) << 54) | (((long) taskId.getPartitionId()) << 40);
            requestValues();
        }

        @Override
        public TransformationState<Page> process(@Nullable Page inputPage)
        {
            if (inputPage == null) {
                return finished();
            }

            return ofResult(inputPage.appendColumn(generateIdColumn(inputPage.getPositionCount())));
        }

        private Block generateIdColumn(int positionCount)
        {
            BlockBuilder block = BIGINT.createFixedSizeBlockBuilder(positionCount);
            for (int currentPosition = 0; currentPosition < positionCount; currentPosition++) {
                if (rowIdCounter >= maxRowIdCounterValue) {
                    requestValues();
                }
                long rowId = rowIdCounter++;
                verify((rowId & uniqueValueMask) == 0, "RowId and uniqueValue mask overlaps");
                BIGINT.writeLong(block, uniqueValueMask | rowId);
            }
            return block.build();
        }

        private void requestValues()
        {
            rowIdCounter = rowIdPool.getAndAdd(ROW_IDS_PER_REQUEST);
            maxRowIdCounterValue = Math.min(rowIdCounter + ROW_IDS_PER_REQUEST, MAX_ROW_ID);
            checkState(rowIdCounter < MAX_ROW_ID, "Unique row id exceeds a limit: %s", MAX_ROW_ID);
        }
    }
}
```