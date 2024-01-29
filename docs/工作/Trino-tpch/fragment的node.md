# AggregationNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
//这里的key一般是sum、count、avg等aggregation类的symbol，value是对应的aggregation函数的信息
private final Map<Symbol, Aggregation> aggregations;
private final GroupingSetDescriptor groupingSets;//描述sql中groupby后面的字段信息
private final List<Symbol> preGroupedSymbols;
//枚举类，PARTIAL、FINAL、INTERMEDIATE、SINGLE，其中PARTIAL(true（inputRaw）, true（outputPartial）)，FINAL(false, false)，INTERMEDIATE(false, true)，SINGLE(true, false)，表示当前AggregationNode属于处理数据的哪一步
private final Step step;
//对symbol取的hash value。这里的hashSymbol对应explain sql中Aggregate的hash = []
//具体可见最下面的理解hashsymbol
private final Optional<Symbol> hashSymbol;
private final Optional<Symbol> groupIdSymbol;
private final List<Symbol> outputs;//该节点输出的Symbol
```



# FilterNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;////孩子节点
private final Expression predicate;//谓词表达式，Expression是一个抽象类
```



# RemoteSourceNode

```java
private final PlanNodeId id;//node节点的ID号
private final List<PlanFragmentId> sourceFragmentIds;//记录数据来源的Fragment的ID号
private final List<Symbol> outputs;//该节点输出的Symbols
private final Optional<OrderingScheme> orderingScheme;//表示order by的信息
private final ExchangeNode.Type exchangeType; //枚举类，GATHER、REPARTITION、REPLICATE
private final RetryPolicy retryPolicy;//枚举类,TASK(RetryMode.RETRIES_ENABLED)、QUERY(RetryMode.RETRIES_ENABLED)、NONE(RetryMode.NO_RETRIES)
```



# SortNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final OrderingScheme orderingScheme;//表示order by的信息
private final boolean partial;//表示是否是部分排序
```



# ProjectNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final Assignments assignments;//该节点输出的symbols

Assignments
private final Map<Symbol, Expression> assignments;//记录Symbol与其对应的Expression。比如我们select sum(l.extendedprice * (1 - l.discount)) AS revenue，此时Symbol是expr，所以得记录其对应的具体的Expression。如果只是单个列名，其Expression也是列名
```



# TopNNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final long count;//表示sql中limit的数量
private final OrderingScheme orderingScheme;//表示order by的信息
//枚举类，SINGLE、PARTIAL、FINAL
private final Step step;
```



# ExchangeNode

```java
//ExchangeNode主要用来从远端或本地获取数据
private final PlanNodeId id;//node节点的ID号
//枚举类，GATHER、REPARTITION、REPLICATE
//表示当前ExchangeNode是属于哪种状态
private final Type type;
//枚举类，LOCAL、REMOTE，表示当前ExchangeNode执行的范围
private final Scope scope;
private final List<PlanNode> sources;//孩子节点
private final PartitioningScheme partitioningScheme;//表示分区的相关信息
private final List<List<Symbol>> inputs;//表示该节点的输入的Symbols
private final Optional<OrderingScheme> orderingScheme;//表示order by的信息
```



# JoinNode

```java
private final PlanNodeId id;//node节点的ID号
//枚举类型，INNER、LEFT、RIGHT、FULL，表示当前join节点的连接类型
private final Type type;
private final PlanNode left;//表示当前节点的左孩子节点
private final PlanNode right;//表示当前节点的右孩子节点
private final List<EquiJoinClause> criteria;//表示join的条件
private final List<Symbol> leftOutputSymbols;//两表根据条件join后，左侧需输出的Symbol
private final List<Symbol> rightOutputSymbols;//两表根据条件join后，右侧需输出的Symbol
private final boolean maySkipOutputDuplicates;
private final Optional<Expression> filter;//过滤表达式
private final Optional<Symbol> leftHashSymbol;//左侧的hash symbol
private final Optional<Symbol> rightHashSymbol;//右侧的hash symbol
//枚举类型，PARTITIONED、REPLICATED
private final Optional<DistributionType> distributionType;
private final Optional<Boolean> spillable;
private final Map<DynamicFilterId, Symbol> dynamicFilters;//记录动态过滤的ID与对应的symbol
private final Optional<PlanNodeStatsAndCostSummary> reorderJoinStatsAndCost;//用于连接重新排序的统计数据和成本
```



# SemiJoinNode

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//类似左孩子节点
//类似右孩子节点，表示的是in subquery那一块的source
private final PlanNode filteringSource;
private final Symbol sourceJoinSymbol;//source侧join的字段的symbol
private final Symbol filteringSourceJoinSymbol;//filteringSource侧join的字段的symbol
private final Symbol semiJoinOutput;//该节点输出的symbol
private final Optional<Symbol> sourceHashSymbol;//source侧join的字段的hash value
//filteringSource侧join的字段的hash value
private final Optional<Symbol> filteringSourceHashSymbol;
//枚举类，PARTITIONED、REPLICATED
private final Optional<DistributionType> distributionType;
//动态过滤的ID
private final Optional<DynamicFilterId> dynamicFilterId;
```



# ValuesNode

```java
private final PlanNodeId id;//node节点的ID号
private final List<Symbol> outputSymbols;//该节点的输出symbol
private final int rowCount;//行数
private final Optional<List<Expression>> rows;//row的相关信息
```



# AssignUniqueId

```java
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final Symbol idColumn;
```



# TableScanNode

```java
private final PlanNodeId id;//node节点的ID号
private final TableHandle table;//该节点扫描的表
private final List<Symbol> outputSymbols;//在表扫描时用到的的Symbol（相当于column）
private final Map<Symbol, ColumnHandle> assignments; // symbol -> column
private final TupleDomain<ColumnHandle> enforcedConstraint;
private final Optional<PlanNodeStatsEstimate> statistics;//该节点的统计数据估计
private final boolean updateTarget;
private final Optional<Boolean> useConnectorNodePartitioning;
```



# OutputNode

```java
//OutputNode是计划的root节点，表示最终结果的输出
private final PlanNodeId id;//node节点的ID号
private final PlanNode source;//孩子节点
private final List<String> columnNames;//最终输出的列名
private final List<Symbol> outputs;//输出的Symbols
```



# Planfragment demo

![img](https://codeding18.github.io/blogs/img/6f90ec69fef6927fdfafe67d05dd3a9a.png)



## 理解hashsymbol

看explain后的fragment理解下为什么有hash value

```sql
explain
SELECT
   l.orderkey,
   sum(l.extendedprice * (1 - l.discount)) AS revenue,
   o.orderdate,
   o.shippriority
         FROM
           customer AS c,
           orders AS o,
           lineitem AS l
         WHERE
           c.mktsegment = 'BUILDING'
           AND c.custkey = o.custkey
           AND l.orderkey = o.orderkey
           AND o.orderdate < DATE '1995-03-15'
           AND l.shipdate > DATE '1995-03-15'
         GROUP BY
           l.orderkey,
           o.orderdate,
           o.shippriority
         ORDER BY
           revenue DESC,
           o.orderdate
         LIMIT 10
         ;
```



Fragment 0 [SINGLE]                                                                                                
     Output layout: [orderkey_4, sum, orderdate, shippriority]                                                        
     Output partitioning: SINGLE []                                                                                   
     Output[columnNames = [orderkey, revenue, orderdate, shippriority]]                                               
     │   Layout: [orderkey_4:bigint, sum:double, orderdate:date, shippriority:integer]                                
     │   Estimates: {rows: 10 (280B), cpu: 0, memory: 0B, network: 0B}                                                
     │   orderkey := orderkey_4                                                                                       
     │   revenue := sum                                                                                               
     └─ TopN[count = 10, orderBy = [sum DESC NULLS LAST, orderdate ASC NULLS LAST]]                                   
        │   Layout: [orderkey_4:bigint, orderdate:date, shippriority:integer, sum:double]                             
        │   Estimates: {rows: 10 (280B), cpu: ?, memory: ?, network: ?}                                               
        └─ LocalExchange[partitioning = SINGLE]                                                                       
           │   Layout: [orderkey_4:bigint, orderdate:date, shippriority:integer, sum:double]                          
           │   Estimates: {rows: 10 (280B), cpu: 0, memory: 0B, network: 0B}                                          
           └─ RemoteSource[sourceFragmentIds = [1]]                                                                   
                  Layout: [orderkey_4:bigint, orderdate:date, shippriority:integer, sum:double]



Fragment 1 [tpch:orders:1500000]                                                                                     
     Output layout: [orderkey_4, orderdate, shippriority, sum]                                                       
     Output partitioning: SINGLE []                                                                                   
     TopNPartial[count = 10, orderBy = [sum DESC NULLS LAST, orderdate ASC NULLS LAST]]                               
     │   Layout: [orderkey_4:bigint, orderdate:date, shippriority:integer, sum:double]                                
     │   Estimates: {rows: 10 (280B), cpu: ?, memory: ?, network: ?}                                                  
     └─ Project[]                                                                                                     
        │   Layout: [orderkey_4:bigint, orderdate:date, shippriority:integer, sum:double]                             
        │   Estimates: {rows: 470322 (12.56MB), cpu: 12.56M, memory: 0B, network: 0B}                                 
        └─ Aggregate[keys = [orderkey_4, orderdate, shippriority], hash = [$hashvalue_13]]                            
           │   Layout: [orderkey_4:bigint, orderdate:date, shippriority:integer, $hashvalue_13:bigint, sum:double]    
           │   Estimates: {rows: 470322 (16.60MB), cpu: 16.60M, memory: 16.60MB, network: 0B}                         
           │   sum := sum("expr")                                                                                     
           └─ Project[]                                                                                               
              │   Layout: [orderdate:date, expr:double, orderkey_4:bigint, shippriority:integer, $hashvalue_13:bigint]
              │   Estimates: {rows: 470322 (16.60MB), cpu: 16.60M, memory: 0B, network: 0B}                           
              │   expr := ("extendedprice" * (1E0 - "discount"))                                                      
              │   $hashvalue_13 := combine_hash(combine_hash(combine_hash(bigint '0', COALESCE("$operator$hash_code"("orderkey_4"), 0)), COALESCE("$operator$hash_code"("orderdate"), 0)), COALESCE("$operator$hash_code"("shippriority"),0))
              └─ InnerJoin[criteria = ("orderkey_4" = "orderkey"), hash = [$hashvalue, $hashvalue_6], distribution =  PARTITIONED]
                 │   Layout: [orderkey_4:bigint, extendedprice:double, discount:double, orderdate:date, shippriority:integer]
                 │   Estimates: {rows: 470322 (16.60MB), cpu: 133.17M, memory: 5.84MB, network: 0B}                   
                 │   Distribution: PARTITIONED                                                                        
                 │   dynamicFilterAssignments = {orderkey -> #df_810}                                                 
                 ├─ ScanFilterProject[table = tpch:sf1:lineitem, filterPredicate = ("shipdate" > DATE '1995-03-15'), dynamicFilters = {"orderkey_4" = #df_810}]
                 │      Layout: [orderkey_4:bigint, extendedprice:double, discount:double, $hashvalue:bigint]         
                 │      Estimates: {rows: 6001215 (206.04MB), cpu: 183.14M, memory: 0B, network: 0B}/{rows: 3225207 (110.73MB), cpu: 183.14M, memory: 0B, network: 0B}/{rows: 3225207 (110.73MB), cpu: 110.73M, memory: 0B, network: 0B}
                 │      $hashvalue := combine_hash(bigint '0', COALESCE("$operator$hash_code"("orderkey_4"), 0))      
                 │      extendedprice := tpch:extendedprice                                                           
                 │      discount := tpch:discount                                                                     
                 │      orderkey_4 := tpch:orderkey                                                                   
                 │      shipdate := tpch:shipdate                                                                     
                 └─ LocalExchange[partitioning = HASH, hashColumn = [$hashvalue_6], arguments = ["orderkey"]]         
                    │   Layout: [orderkey:bigint, orderdate:date, shippriority:integer, $hashvalue_6:bigint]          
                    │   Estimates: {rows: 218741 (5.84MB), cpu: 5.84M, memory: 0B, network: 0B}                       
                    └─ RemoteSource[sourceFragmentIds = [2]]                                                          
                           Layout: [orderkey:bigint, orderdate:date, shippriority:integer, $hashvalue_7:bigint]



Fragment 2 [tpch:orders:1500000]                                                                                     
     Output layout: [orderkey, orderdate, shippriority, $hashvalue_12]                                                
     Output partitioning: tpch:orders:1500000 [orderkey]                                                              
     Project[]                                                                                                        
     │   Layout: [orderkey:bigint, orderdate:date, shippriority:integer, $hashvalue_12:bigint]                        
     │   Estimates: {rows: 218741 (5.84MB), cpu: 5.84M, memory: 0B, network: 0B}                                      
     │   $hashvalue_12 := combine_hash(bigint '0', COALESCE("$operator$hash_code"("orderkey"), 0))                    
     └─ InnerJoin[criteria = ("custkey_1" = "custkey"), hash = [$hashvalue_8, $hashvalue_9], distribution = REPLICATED
        │   Layout: [orderkey:bigint, orderdate:date, shippriority:integer]                                           
        │   Estimates: {rows: 218741 (3.96MB), cpu: 30.21M, memory: 527.34kB, network: 0B}                            
        │   Distribution: REPLICATED                                                                                  
        │   dynamicFilterAssignments = {custkey -> #df_812}                                                           
        ├─ ScanFilterProject[table = tpch:sf1:orders, filterPredicate = ("orderdate" < DATE '1995-03-15'), dynamicFilters = {"custkey_1" = #df_812}]
        │      Layout: [orderkey:bigint, custkey_1:bigint, orderdate:date, shippriority:integer, $hashvalue_8:bigint] 
        │      Estimates: {rows: 1500000 (52.93MB), cpu: 40.05M, memory: 0B, network: 0B}/{rows: 729106 (25.73MB), cpu: 40.05M, memory: 0B, network: 0B}/{rows: 729106 (25.73MB), cpu: 25.73M, memory: 0B, network: 0B}
        │      $hashvalue_8 := combine_hash(bigint '0', COALESCE("$operator$hash_code"("custkey_1"), 0))              
        │      orderkey := tpch:orderkey                                                                              
        │      custkey_1 := tpch:custkey                                                                              
        │      orderdate := tpch:orderdate                                                                            
        │      shippriority := tpch:shippriority                                                                      
        │      tpch:orderstatus                                                                                      
        │          :: [[F], [O], [P]]                                                                                 
        └─ LocalExchange[partitioning = SINGLE]                                                                       
           │   Layout: [custkey:bigint, $hashvalue_9:bigint]                                                          
           │   Estimates: {rows: 30000 (527.34kB), cpu: 0, memory: 0B, network: 0B}                                   
           └─ RemoteSource[sourceFragmentIds = [3]]                                                                   
                  Layout: [custkey:bigint, $hashvalue_10:bigint]



Fragment 3 [SOURCE]                                                                                                  
     Output layout: [custkey, $hashvalue_11]                                                                          
     Output partitioning: BROADCAST []                                                                                
     ScanFilterProject[table = tpch:sf1:customer, filterPredicate = ("mktsegment" = CAST('BUILDING' AS varchar(10)))] 
      │  Layout: [custkey:bigint, $hashvalue_11:bigint]                                                               
      │  Estimates: {rows: 150000 (2.57MB), cpu: 2.00M, memory: 0B, network: 0B}/{rows: 30000 (527.34kB), cpu: 2.00M, memory: 0B, network: 0B}/{rows: 30000 (527.34kB), cpu: 527.34k, memory: 0B, network: 0B}
      │  $hashvalue_11 := combine_hash(bigint '0', COALESCE("$operator$hash_code"("custkey"), 0))                     
      │   mktsegment := tpch:mktsegment                                                                                
      │  custkey := tpch:custkey