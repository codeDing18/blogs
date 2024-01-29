# AggregationNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final Map<Symbol, Aggregation> aggregations;
private final GroupingSetDescriptor groupingSets;
private final List<Symbol> preGroupedSymbols;
//枚举类，PARTIAL、FINAL、INTERMEDIATE、SINGLE，其中PARTIAL(true（inputRaw）, true（outputPartial）)，FINAL(false, false)，INTERMEDIATE(false, true)，SINGLE(true, false)，表示当前AggregationNode属于处理数据的哪一步
private final Step step;
private final Optional<Symbol> hashSymbol;
private final Optional<Symbol> groupIdSymbol;
private final List<Symbol> outputs;//该节点输出的列
```



# ApplyNode

```java
private final PlanNodeId id;
private final PlanNode input;
private final PlanNode subquery;
private final List<Symbol> correlation;
private final Assignments subqueryAssignments;
private final Node originSubquery;
```



# AssignUniqueId

```java
private final PlanNodeId id;
private final PlanNode source;
private final Symbol idColumn;
```



# CorrelatedJoinNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode input;//该节点的输入
private final PlanNode subquery;//
private final List<Symbol> correlation;
private final Type type;
private final Expression filter;
private final Node originSubquery;
```



# DistinctLimitNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final long limit;
private final boolean partial;
private final List<Symbol> distinctSymbols;
private final Optional<Symbol> hashSymbol;
```



# DynamicFilterSourceNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final Map<DynamicFilterId, Symbol> dynamicFilters;
```



# EnforceSingleRowNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
```



# ExceptNode

```java
private final PlanNodeId id;
private final List<PlanNode> sources;
private final ListMultimap<Symbol, Symbol> outputToInputs;
private final List<Symbol> outputs;
private final boolean distinct;
```



# ExchangeNode

```java
private final PlanNodeId id;
private final Type type;
private final Scope scope;
private final List<PlanNode> sources;
private final PartitioningScheme partitioningScheme;
private final List<List<Symbol>> inputs;
private final Optional<OrderingScheme> orderingScheme;
```



# ExplainAnalyzeNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final Symbol outputSymbol;
private final List<Symbol> actualOutputs;
private final boolean verbose;
```



# FilterNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;////孩子节点
private final Expression predicate;//谓词表达式

//具体介绍下predicate（LogicalExpression）
private final Operator operator;//枚举类AND、OR，表示在where中用的and还是or
private final List<Expression> terms;//where中具体的表达式语句
```



# GroupIdNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final List<List<Symbol>> groupingSets;
private final Map<Symbol, Symbol> groupingColumns;
private final List<Symbol> aggregationArguments;
private final Symbol groupIdSymbol;
```



# GroupReference

```java
private final PlanNodeId id;
private final int groupId;
private final List<Symbol> outputs;
```



# IndexJoinNode

```java
private final PlanNodeId id;
private final Type type;
private final PlanNode probeSource;
private final PlanNode indexSource;
private final List<EquiJoinClause> criteria;
private final Optional<Symbol> probeHashSymbol;
private final Optional<Symbol> indexHashSymbol;
```



# IndexSourceNode

```java
private final PlanNodeId id;
private final IndexHandle indexHandle;
private final TableHandle tableHandle;
private final Set<Symbol> lookupSymbols;
private final List<Symbol> outputSymbols;
private final Map<Symbol, ColumnHandle> assignments;
```



# IntersectNode

```java
private final PlanNodeId id;
private final boolean distinct;
private final List<PlanNode> sources;
private final ListMultimap<Symbol, Symbol> outputToInputs;
private final List<Symbol> outputs;
```



# JoinNode

```java
private final PlanNodeId id;
private final Type type;
private final PlanNode left;
private final PlanNode right;
private final List<EquiJoinClause> criteria;
private final List<Symbol> leftOutputSymbols;
private final List<Symbol> rightOutputSymbols;
private final boolean maySkipOutputDuplicates;
private final Optional<Expression> filter;
private final Optional<Symbol> leftHashSymbol;
private final Optional<Symbol> rightHashSymbol;
private final Optional<DistributionType> distributionType;
private final Optional<Boolean> spillable;
private final Map<DynamicFilterId, Symbol> dynamicFilters;
private final Optional<PlanNodeStatsAndCostSummary> reorderJoinStatsAndCost;
```



# LimitNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final long count;
private final Optional<OrderingScheme> tiesResolvingScheme;
private final boolean partial;
private final List<Symbol> preSortedInputs;
```



# MarkDistinctNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final Symbol markerSymbol;
private final Optional<Symbol> hashSymbol;
private final List<Symbol> distinctSymbols;
```



# MergeProcessorNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final MergeTarget target;
private final Symbol rowIdSymbol;
private final Symbol mergeRowSymbol;
private final List<Symbol> dataColumnSymbols;
private final List<Symbol> redistributionColumnSymbols;
private final List<Symbol> outputs;
```



# MergeWriterNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final MergeTarget target;
private final List<Symbol> projectedSymbols;
private final Optional<PartitioningScheme> partitioningScheme;
private final List<Symbol> outputs;
```



# OffsetNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final long count;
```



# OutputNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final List<String> columnNames;
private final List<Symbol> outputs;
```



# PatternRecognitionNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final DataOrganizationSpecification specification;
private final Optional<Symbol> hashSymbol;
private final Set<Symbol> prePartitionedInputs;
private final int preSortedOrderPrefix;
private final Map<Symbol, WindowNode.Function> windowFunctions;
private final Map<Symbol, Measure> measures;
```



# ProjectNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final Assignments assignments;//该节点被分配到的symbols

Assignments
private final Map<Symbol, Expression> assignments;//记录Symbol与其对应的Expression
```



# RefreshMaterializedViewNode

```java
private final PlanNodeId id;
private final QualifiedObjectName viewName;
```



# RemoteSourceNode

```java
private final PlanNodeId id;
private final List<PlanFragmentId> sourceFragmentIds;
private final List<Symbol> outputs;
private final Optional<OrderingScheme> orderingScheme;
private final ExchangeNode.Type exchangeType; 
private final RetryPolicy retryPolicy;
```



# RowNumberNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final List<Symbol> partitionBy;
private final boolean orderSensitive;
private final Optional<Integer> maxRowCountPerPartition;
private final Symbol rowNumberSymbol;
private final Optional<Symbol> hashSymbol;
```



# SampleNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final double sampleRatio;
private final Type sampleType;
```



# SemiJoinNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final PlanNode filteringSource;
private final Symbol sourceJoinSymbol;
private final Symbol filteringSourceJoinSymbol;
private final Symbol semiJoinOutput;
private final Optional<Symbol> sourceHashSymbol;
private final Optional<Symbol> filteringSourceHashSymbol;
private final Optional<DistributionType> distributionType;
private final Optional<DynamicFilterId> dynamicFilterId;
```



# SimpleTableExecuteNode

```java
private final PlanNodeId id;
private final Symbol output;
private final TableExecuteHandle executeHandle;
```



# SortNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final OrderingScheme orderingScheme;
private final boolean partial;
```



# SpatialJoinNode

```java
private final PlanNodeId id;
private final Type type;
private final PlanNode left;
private final PlanNode right;
private final List<Symbol> outputSymbols;
private final Expression filter;
private final Optional<Symbol> leftPartitionSymbol;
private final Optional<Symbol> rightPartitionSymbol;
private final Optional<String> kdbTree;
private final DistributionType distributionType;
```



# StatisticsWriterNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final Symbol rowCountSymbol;
private final WriteStatisticsTarget target;
private final boolean rowCountEnabled;
private final StatisticAggregationsDescriptor<Symbol> descriptor;
```



# TableDeleteNode

```java
private final PlanNodeId id;
private final TableHandle target;
private final Symbol output;
```



# TableExecuteNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final TableExecuteTarget target;
private final Symbol rowCountSymbol;
private final Symbol fragmentSymbol;
private final List<Symbol> columns;
private final List<String> columnNames;
private final Optional<PartitioningScheme> partitioningScheme;
private final Optional<PartitioningScheme> preferredPartitioningScheme;
private final List<Symbol> outputs;
```



# TableFinishNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final WriterTarget target;
private final Symbol rowCountSymbol;
private final Optional<StatisticAggregations> statisticsAggregation;
private final Optional<StatisticAggregationsDescriptor<Symbol>> statisticsAggregationDescriptor;
```



# TableFunctionNode

```java
private final PlanNodeId id;
private final String name;
private final Map<String, Argument> arguments;
private final List<Symbol> properOutputs;
private final List<PlanNode> sources;
private final List<TableArgumentProperties> tableArgumentProperties;
private final List<List<String>> copartitioningLists;
private final TableFunctionHandle handle;
```



# TableScanNode

```java
private final PlanNodeId id;//node节点的ID号
private final TableHandle table;//该节点扫描的表
private final List<Symbol> outputSymbols;//输出的符号（相当于column）
private final Map<Symbol, ColumnHandle> assignments; // symbol -> column
private final TupleDomain<ColumnHandle> enforcedConstraint;
private final Optional<PlanNodeStatsEstimate> statistics;//该节点的统计数据估计
private final boolean updateTarget;
private final Optional<Boolean> useConnectorNodePartitioning;
```



# TableWriterNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final WriterTarget target;
private final Symbol rowCountSymbol;
private final Symbol fragmentSymbol;
private final List<Symbol> columns;
private final List<String> columnNames;
private final Optional<PartitioningScheme> partitioningScheme;
private final Optional<PartitioningScheme> preferredPartitioningScheme;
private final Optional<StatisticAggregations> statisticsAggregation;
private final Optional<StatisticAggregationsDescriptor<Symbol>> statisticsAggregationDescriptor;
private final List<Symbol> outputs;
```



# TopNNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final long count;
private final OrderingScheme orderingScheme;
private final Step step;
```



# TopNRankingNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final DataOrganizationSpecification specification;
private final RankingType rankingType;
private final Symbol rankingSymbol;
private final int maxRankingPerPartition;
private final boolean partial;
private final Optional<Symbol> hashSymbol;
```



# UnionNode

```java
private final PlanNodeId id;
private final List<PlanNode> sources;
private final ListMultimap<Symbol, Symbol> outputToInputs;
private final List<Symbol> outputs;
```



# UnnestNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final List<Symbol> replicateSymbols;
private final List<Mapping> mappings;
private final Optional<Symbol> ordinalitySymbol;
private final Type joinType;
private final Optional<Expression> filter;
```



# ValuesNode

```java
private final PlanNodeId id;
private final List<Symbol> outputSymbols;
private final int rowCount;
private final Optional<List<Expression>> rows;
```



# WindowNode

```java
private final PlanNodeId id;
private final PlanNode source;
private final Set<Symbol> prePartitionedInputs;
private final DataOrganizationSpecification specification;
private final int preSortedOrderPrefix;
private final Map<Symbol, Function> windowFunctions;
private final Optional<Symbol> hashSymbol;
```