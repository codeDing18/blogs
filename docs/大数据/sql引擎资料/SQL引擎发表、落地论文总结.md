## **简介**

大概用了9个月的时间，研读了优化器、执行器经典论文，论文主要来源于[梁辰：优化器论文列表](https://zhuanlan.zhihu.com/p/363997416)和[https://15721.courses.cs.cmu.edu](https://link.zhihu.com/?target=https%3A//15721.courses.cs.cmu.edu/spring2018/schedule.html%23warning)，辅助性来自一些国外明显的topic。研读期间，学习了

[梁辰](https://www.zhihu.com/people/yaoling-lc)、[https://loopjump.com/](https://link.zhihu.com/?target=https%3A//loopjump.com/)对优化器、执行器的论文解读，收益很大。

同时，在研读论文过程中，收集整理了论文的发表单位，以及论文的落地产品。个人理解，论文更多的是体现出一个idea以及这个idea的实现思路，如果想融汇贯通、运用自如，还是需要进一步研究分析相关的开源数据库源码、闭源数据库的功能特性。

由于个人能力有限，对论文的解读、按数据库进行整理的内容可能存在一定偏差或错误，期待能够和感兴趣的同学学习交流、共同进步。

## **Oracle**

[[SIGMOD 1979\] Access Path Selection in a Relational Database Management System 论文阅读](https://zhuanlan.zhihu.com/p/590776990)

本文虽然发表于1979年，但在System R这个关系型数据库研究项目中，提出的制定SQL查询计划思想（自下而上+启发式+基于代价评估）至今仍被主流数据库所使用，例如Oracle、DB2、PostgreSQL等。

本文解决的主要问题是，用于在不了解数据库存储细节的情况下，执行了一个SQL查询，数据库优化器如何将SQL转换为最优的执行路径，即最高效的执行。

[[VLDB 06\]Cost-based query transformation in Oracle 论文学习](https://zhuanlan.zhihu.com/p/595947967)

本篇论文虽是2006年发表，但其中的启发式规则、代价评估规则仍被主流数据库使用。在Heuristic Transformations转换规则中，Join Elimination、Filter Predicate Move Around、Group Pruning规则必然能获得更加的查询计划，但Subquery Unnesting则需要依赖表中具体的数据行数决定。代价评估规则中，给出了8中转换规则，并结合实例说明如何使用。论文也提出了一个代价评估框架，包括框架的组件、state搜索算法、transformation规则的执行方式以及框架的性能优化方法。

[[VLDB 2017\]Adaptive Statistics In Oracle 12c --学习笔记](https://zhuanlan.zhihu.com/p/622026398)

论文介绍了查询执行过程中，中间结果统计信息错误导致的问题，并给出了Oracle 12c的解决方案。该解决方案包括使用表的数据和通过自动创建辅助统计信息来验证已生成的统计信息的准确性。

[[SIGMOD 2004\]Parallel SQL Execution in Oracle 10g --学习笔记](https://zhuanlan.zhihu.com/p/621200670)

论文介绍了Oracle10g在并行执行方面的新架构和优化。自Oracle7开始，Oracle就基于share-disk架构实现了sql并行执行框架。在Oracle10g对并行执行引擎进行了重构，通过使用全局并行计划模型，使得并行执行引擎更高效、更易于动态扩展、更易于管理。论文不是很好理解，有些内容只是简单的说了下what，没有解释how，另外也涉及到了许多Oracle的相关名词术语。

[[VLDB 2015\]Query Optimization in Oracle 12c Database In-Memory--学习笔记](https://zhuanlan.zhihu.com/p/620535500)

Oracle 12c内存数据库是业界第一个 dual-format（同时支持行存、列存）的数据库。在IO层，仍然使用行存格式。在内存中，同时支持行存和列存。数据存储方式的变化，使得数据查询处理算法也要进行扩展，这就要求查询优化器同步做扩展和适配，以便在支持列存的存储、查询计算的情况下，能够产生最佳的查询计划。论文给出了引入Database In-Memory后，查询优化器的扩展，具体包括统： statistics, cost model, query transformation, access path 和 join 优化, parallelism, 以及 cluster-awareness优化。

[[VLDB 2013\]Adaptive and Big Data Scale Parallel Execution in Oracle--学习笔记](https://zhuanlan.zhihu.com/p/619664281)

论文介绍了Oracle数据库在执行join、groupby、窗口函数方面引入的新的并行+自适应方法。这套技术的创新点是采用了多阶段并发模型（multi-stage parallelization models），能够在运行时根据实时统计信息，将由于优化器统计信息不准导致的次优plan，动态调整为更优的plan，主要策略是：动态扩展并行执行进程的数量，以及调整数据的重分布策略，避免data skew。论文还讨论了Oracle原有的串行化执行的算子（如top-N）的并行化执行的方法，Oracle从这些自适应并行化技术中实现了巨大的性能提高。

[[VLDB 2009\]Enhanced Subquery Optimizations in Oracle-Oracle去相关子查询论文学习](https://zhuanlan.zhihu.com/p/617059579)

paper从优化器、执行器两方面提出了子查询优化的方法：

- 优化器：子查询优化方法，包括子查询合并、基于窗口函数的子查询删除、在groupby查询中删除view，目标是删除冗余子查询块，并将子查询转换为更容易优化的形式。
- 执行器：提出了一种并行执行技术，具有普遍适用性和可扩展性，并可以同经历了某些transform的查询共同使用

另外，paper针对antijoin处理null值的问题，提出了一个anntijoin算子的变种Null-Aware Antijoin，用于优化包含可能为null的column的子查询。

[[SIGMOD '03\] WinMagic Subquery Elimination Using Window Aggregation--DB2去相关子查询论文学习](https://zhuanlan.zhihu.com/p/618026383)

paper介绍了IBM DB2中利用窗口函数针对agg场景下去相关子查询的方法，该方法虽然有些约束限制，但如果满足的话，还是很高效的。目前，PolarDB MySQL、OceanBase都实现了这个算法。据了解，Oracle也实现了这个算法。

[[SIGMOD 2015\]Rethinking SIMD Vectorization for In-Memory Databases--学习笔记](https://zhuanlan.zhihu.com/p/640818697)

论文作者来自哥伦比亚大学和Oracle。论文基于先进的SIMD指令，例如gathers和scatters，提出了一种新的向量化执行引擎，并给出了selection scans、hash tables、partition的向量化实现代码，在这些基础上，实现了sort和join等算子。论文将提出的向量化实现技术与最先进的scalar和vectorized技术进行了对比，通过在Xeon Phi以及主流CPU的实测结果，证实论文提出的技术的高性能。

## **SQL Server**

[[IEEE Data engineering Bulletin 1995\] The Cascades Framework for Query Optimization论文阅读](https://zhuanlan.zhihu.com/p/590793684)

本篇论文已经在SQL Server中落地，论文对Volcano Optimizer Generator论文的后续，作者提出了Cascades优化器框架，对Volcano Optimizer Generator在功能、易用性和健壮性方面的实质性改进，主要的不同点在于volcano是先等价替换出所有的逻辑计划，然后计算物理计划。而cascade是没有区分逻辑计划和物理计划，统一处理。

本论文也是抽象化的提出了一种Cascades优化器框架，给出了一些工程化实现的说明。与Volcano优化器相比，Cascades性能更加。因为Volcano采用了穷举方法，在父节点对子节点优化过程中，采用了广度优先算法，先等价替换出所有的逻辑计划，然后计算物理计划。Cascades是没有区分逻辑计划和物理计划，统一处理，采用了类似深度优先的算法。并且使用了指导规则，一些搜索工作是可以避免的，Cascades的搜索策略似乎更好。

[[ICDE 2010\]Incorporating Partitioning and Parallel Plans into the SCOPE optimizer--学习笔记](https://zhuanlan.zhihu.com/p/642248107)

在大规模集群中进行大量数据的分析为查询优化带来了新的机会和挑战。这种场景下，使用数据分区是提升性能的关键。然而，数据重分布的代价非常高，因此，在查询计划中，如果可以在算子中最大程度的减少数据重分布操作，那么可以显著的提升性能。在这个存在分区的环境下。查询优化器需要能够利用数据分区的位置、排序、分组等元数据信息，制定更高效的查询计划。

论文讨论的是微软的Cosmos。Cosmos是微软的一个运行在大规模服务器集群上的分布式技术平台，专门用来存储和分析Massive Data，Scope是运行在Cosmos环境下的数据处理工具

- Scope is the Query Language for Cosmos（Scope是在Cosmos平台下运行的Query语言）
- Scope Is Not SQL （Scope像SQL 但不同于SQL）
- Scope的优化器负责生成最优的执行计划
- scope优化器能够利用数据分区信息能够在代价评估中考虑并行计划、串行计划、二者混合的计划，选出最优解

这篇论文虽是基于scope的，但已经在SQL Server中落地。

[[SIGMOD '00\]Counting, enumerating, and sampling of execution plans in a cost-based query optimizer学习](https://zhuanlan.zhihu.com/p/626372143)

在对优化器的测试中，一项重要的测试内容是，对比分析优化器选出的plan的执行时间与优化器中其他候选plans的执行时间，进而判断优化器从候选plans中选择较佳的plan的能力，那么问题就来了，如何才能知道优化器中，其他候选plans的执行时间？

这篇论文给出了一个很好的方法，就是在优化器制定query的执行计划过程中，建立好数字和候选plans的对应关系，在SQL语句执行时，用户可以输入数字，指定执行plan。这个技术已经应用到SQL Server的优化器测试验证中了。

由于SQL Server采用了Cascade框架，因此，整个实现是围绕MEMO结构进行的。

[[Data Engineering Bulletin '08\]Testing SQL Server's Query Optimizer--学习笔记](https://zhuanlan.zhihu.com/p/626015214)

查询优化本身就很复杂，因此，验证优化器的正确性和有效性是一项非常复杂的工作。正确性是说生成的查询计划对标原sql，能不能保证结果集的正确性；有效性是说，对原sql执行RBO+CBO变换后，得到的执行计划是不是高效的。

这篇论文描述了测试查询优化器时遇到的一些独特问题，概述了验证微软SQL Server的查询优化器的测试技术，给出了优化器测试过程中的成功经验和教训，最后讨论了测试器测试中持续存在的挑战。

论文更多的是从测试工程的角度说明优化器测试问题，以及微软SQL Server是如何测试优化器的，给出的方法相对要宏观一些，或者说更像是一种方向性的指引。很遗憾，可能是出于商业保密的原因，论文并没有公开优化器测试的实现机制。论文[DBTest 2012] Testing the Accuracy of Query Optimizers，给出了优化器测试的数学模型，但没有从系统工程的角度说明如何进行优化器的测试。这两篇论文有很强的互补性，都是很值得学习的。

[[Microsoft Research 2000\] Parameterized queries and nesting equivalences--SQL Server去相关子查询](https://zhuanlan.zhihu.com/p/601494429)

本文为解决SQL关联子查询问题，提出了Apply算子，用于描述SQL子查询，并给出了Apply算子向join转换的恒等式，基于这些恒等式，可以实现SQL子查询去关联。Apply算子已经应用到SQL Server7.0。

[[SIGMOD '01\] Orthogonal Optimization of Subqueries and Aggregation --SQL Server去相关子查询](https://zhuanlan.zhihu.com/p/601449775)

论文提出了一种小的、独立的Correlation removal和有效处理outer join、group by的源语（rule），通过正交组成丰富的执行计划，高效解决子查询问题。

[[SIGMOD 07\] Execution strategies for SQL subqueries --SQL Server去相关子查询](https://zhuanlan.zhihu.com/p/601499565)

本文描述了SQL Server 2005中用于高效计算包含子查询的查询的大量策略。论文从必不可少的先决条件开始，例如子查询的相关性检测和删除。然后确定了一系列基本的子查询执行策略，例如正向查找和反向查找，以及set-based的方法，解释了在SQL Server中实现的子查询的不同执行策略，并将它们与当前的技术状态联系起来。实验评价对论文进行了补充。它量化了不同方法的性能特征，并表明在不同的情况下确实需要不同的执行策略，这使得基于成本的查询优化器对于提升查询性能必不可少。

[[SIGMOD 2012\]Query optimization in Microsoft SQL Server PDW--学习笔记](https://zhuanlan.zhihu.com/p/618780370)

PDW QO是paper描述了SQL Server MPP查询优化器的设计与实现，Paralled Data Warehouse产品的查询优化器（PDW QO）。PDW QO是基于SQL Server单机Cascade优化器技术实现的针对分布式查询的CBO优化器。PDW QO在控制节点采用两阶段方式制定DPlan（分布式执行计划）：

1. 基于单机Cascade优化器架构，使用全局统计信息，产生若干个潜在winner的memo结构
2. 根据数据的分布信息，对这些候选plans进行枚举和剪枝，选择最佳，生成分布式的执行计划。并将这个分布式执行计划树转换为DSQL语句（扩展SQL），分发到各计算节点执行。

在控制节点制定分布式执行计划时，会根据数据重分布的操作将整棵查询执行树切割成不同的子树。每个子树对应查询计划的一个阶段，就是GreenPlum中的slice。每个slice都被转换成DSQL语句，在各个计算节点的sql server上执行，互相之间通过网络，发送到远端时要全物化，同时会收集物化的中间结果的统计信息，然后计算节点执行当前slice对应的dsql语句时，会基于这些统计信息重新进行查询优化，可能会有本地更优的计划。

与基于成本的transform相关的一个基本问题是，这些transform是否会导致需要评估的替代方案的组合数量爆炸，以及它们是否会在查询优化的成本和SQL执行的成本之间提供权衡。PDW优化器在这方面有一个非常重要的搜索优化，即timeout机制+从候选plans选取“seed”plans。

SQL Server优化器的timeout机制，不会生成所有可能的plans。在MEMO中基于全局统计信息的可能winner的plans对所考虑的Search space有很大的影响。为了提升PDW查询优化的效率，尽量不穷举所有的备选plans，而是从备选plans中考虑table的数据分布信息以高效配置DMS算子，选择部分"优质plans"，即“seed”plans。这个考虑数据分布的方式和interesting order类似，从上到下提出require distribution，从下向上提出provide distribution，进行匹配，不满足增加exchange，这个选种子plans时，尽量选择不需要增加exchange的候选plans。

## **CockroachDB**

目前，没有见到CockroachDB（简称CRDB）公开发表的优化器、执行器方面的论文，但其源码或设计文档中，明确告知了优化器执行器依赖的论文，具体如下：

### （1）去相关子查询，主要实现了SQL Server的apply算子

[[Microsoft Research 2000\] Parameterized queries and nesting equivalences--SQL Server去相关子查询](https://zhuanlan.zhihu.com/p/601494429)

[[SIGMOD '01\] Orthogonal Optimization of Subqueries and Aggregation --SQL Server去相关子查询](https://zhuanlan.zhihu.com/p/601449775)

[[SIGMOD 07\] Execution strategies for SQL subqueries --SQL Server去相关子查询](https://zhuanlan.zhihu.com/p/601499565)

此外，去相关子查询中，实现了[[VLDB 06\]Cost-based query transformation in Oracle 论文学习](https://zhuanlan.zhihu.com/p/595947967)中提出的规则。

### （2）Outerjoin Simplification

[[ACM Trans'97\] Outerjoin Simplification and Reordering for Query Optimization--论文学习](https://zhuanlan.zhihu.com/p/601454148)

论文的主要贡献如下：

1. Outerjoin Simplification，不改变join次序，提出了算法A （基于null reject的outerjoin Simplification，见2.1）

2. 1. selection算子下推
   2. leftouter join 或 rightouter join 转 inner join

3. join reorder

4. 1. 提出广义outerjoin算法**（Generalized Outerjoin）**，
   2. join reorder变换的恒等式

这里重点介绍Outerjoin Simplification（outerjoin 化简）算法，因为这个算法至今仍在主流数据库优化器中使用。

简单说明join reorder算法**（Generalized Outerjoin）**，GOJ。因为这个算法在使用中存在很多局限性，例如仅适用于**simple outerjoin**，基于query graph进行实现，实现难度大。

对于join reorder算法感兴趣的同学可以阅读《[SIGMOD 13]On the correct and complete enumeration of the core search space》，论文主体算法已经应用于TiDB、PolarDB、CockroachDB等多款主流数据库中。

### （3）Join Reorder

[[VLDB 2006\]Analysis of Two Existing and One New Dynamic Programming--论文学习](https://zhuanlan.zhihu.com/p/632872022)

论文描述了现有文献中的两种构造join order tree的动态规划算法的方法，DPSize算法、DPsub算法，通过分析和实验表明，这两种算法对于不同的query graph表现出非常不同的效果。更具体地说，对于chain （链型）Join graph查询，DPsize优于DPsub，对于clique（集团型）Join graph查询，DPsub优于DPsize。另外，这两种算法都不能很好的处理star（星型）查询。为解决这些问题，论文提出了一种优于DPSize和DPsub的算法，即DPCpp算法，该算法适用于所有类型的query graph。

CRDB主要实现了DPSube算法，其最初的来源为DPsub算法。

[[SIGMOD 13\]On the correct and complete enumeration of the core search space--学习](https://zhuanlan.zhihu.com/p/595870481)

本文针对多表join，基于交换律、结合律、左结合律、右结合律，进行等价转换，产生正确且完整的plan集合(core search space)，论文主体算法已经应用于TiDB、PolarDB、CockroachDB等多款主流数据库中。

[再读[SIGMOD 13\]On the correct and complete enumeration of the core search space](https://zhuanlan.zhihu.com/p/632565261)

本文针对当前主流两种基于动态规划的plan生成器（DB-based plan generator）算法，即NEL/EEL方法算法、SES/TES方法等会在Join Reorder中产生无效plans的问题，论文提出了三种Join Reorder过程中的冲突检测算法（CD-A、CD-B、CD-C），其中CD-C最为完备，能够用于生成完整的core search space（搜索空间），即根据初始的plan，基于Join Reorder算法，生成所有合法的候选plans的集合。

### （4）统计

[[ACM,2013\]HyperLogLog in Practice Algorithmic Engineering of a State of The Art 学习笔记](https://zhuanlan.zhihu.com/p/632565939)

基数估计（Cardinality Estimation），也称为 count-distinct problem，是为了估算在一批数据中，它的不重复元素的个数，distinct value(DV)，具有广泛的应用前景，在数据库系统中尤为重要。虽然基数可以很容易地使用基数中的线性差值来计算，但对于许多应用程序，这是完全不切实际的，并且消耗大量内存。因此，许多近似基数而使用较少资源的算法已经被开发出来。这些算法在网络监控系统、数据挖掘应用程序以及数据库系统中发挥着重要的作用，HyperLogLog算法就是其中之一。

目前，HyperLogLog算法已经应用于Redis、PolarDB PostgreSQL、OpenGauss、CockroachDB、AnalyticDB for PostgreSQL、PostgreSQL、Spark、Flink、Presto等。

在本文中，作者提出了对该算法的一系列改进，以减少了其内存需求，并显著提高了其对一个重要的基数范围的精度。作者已经为谷歌的一个系统实现了这个新提出的算法，并对其进行了经验评估，并与改进前的HyperLogLog算法进行了比较。与改进前的HyperLogLog一样，作者改进的算法可以完美地并行化执行，并在single-pass中完了基数估计的计算。

### （5）基础设施框架

[University of Waterloo 2000] Exploiting Functional Dependence in Query Optimization

ANSI SQL关系模型中已经包括函数依赖分析，但函数依赖分析具有复杂性，因为存在如下情况：

- 空值
- 三值逻辑（three-valued logic），即TRUE、FALSE、UNKNOWN
- outer joins
- 重复的rows

那么如何简化函数依赖分析，并充分利用函数依赖分析进行查询优化呢？

本文是滑铁卢大学2000年的一篇博士论文，主要解决了这方面的问题。论文虽然有些老，但学完之后还是感觉收益匪浅，特别是通过研读本文提供的函数依赖定义、定理、推理等，有助于理解函数依赖在如下SQL优化中的作用：

- selectivity estimation
- estimation of (intermediate) result sizes
- order optimization(in particular sort avoidance)
- cost estimation
- various problems in the area of semantic query optimization

[不会游泳的鱼：[University of Waterloo 2000\] Exploiting Functional Dependence in Query Optimization--学习笔记上-基本认知](https://zhuanlan.zhihu.com/p/603358443)

[不会游泳的鱼：[University of Waterloo 2000\] Exploiting Functional Dependence in Query Optimization--学习笔记中-数学定义](https://zhuanlan.zhihu.com/p/603362330)

[不会游泳的鱼：[University of Waterloo 2000\] Exploiting Functional Dependence in Query Optimization--学习笔记下-函数依赖生成算法](https://zhuanlan.zhihu.com/p/603364178)

[[SIGMOD 1996\] Fundamental Techniques for Order Optimization 论文学习](https://zhuanlan.zhihu.com/p/595949653)

这是DB2 96年的一篇关于join中order（排序）优化论文，提出了具体的优化方法，并概述了DB2的工程实现。这篇论文虽然比较老，但至今仍有很大的指导作用。

在数据库优化器中，排序是一个非常消耗资源的算子，如果能够利用数据中的索引或排序的属性，降低对排序算子的使用，可以大幅提升SQL执行效率。具体来说，在Group By、Order By、Distinct子句中，父节点都会对子节点提出 Interesting Order ,优化器如果能够利用底层已有的序（例如，主键、索引），满足 Interesting Order，就不需要对底层扫描的行做排序，还可以消除 ORDER BY。

论文提出了一种在join中下推sort算子的方法，以减少按照columns对数据的排序，基于谓词、索引、主键信息，在等效情况下，消除排序算子。

为了便于对论文的理解，这里对Interesting Order、Physical Property进行解释。

- Interesting Order:（也许叫Interesting Property更合适，因为多数Interesting的都是排序属性。）由父节点传给子节点，要求其返回满足特定Physical Property的最优计划，例如升序、降序。另一个能够解释“Interesting Order”名称的是**Join Order**。在多表互Join时，表的Join顺序（Join Order）排列组合数量巨大，影响查询性能；父节点需选择哪些Join Order应被进一步探索（Join Enumeration）。它们成了“Interesting Order”。
- Physical Property：Operator返回的结果带有的属性，最典型的例子是排序与否；还例如结果是否已计算Hash；结果在单服务器上，还是分散在多服务器（需要额外GatherMerge）。

在开始做CBO之前，优化器会top-down的遍历QGM，针对不同算子可能的有序性需求（join/group by/distinct/order by），建立算子的interesting order，这个过程称为对QGM的**order scan**。

### （5）优化器框架

[[IEEE Data engineering Bulletin 1995\] The Cascades Framework for Query Optimization论文阅读](https://zhuanlan.zhihu.com/p/590793684)，CRDB采用了Cascade架构。

[[SIGMOD 2014\] Orca: A Modular Query Optimizer Architecture for Big Data 论文阅读](https://zhuanlan.zhihu.com/p/590795795)，Cascade架构实现类似ORCA。

### （6）SQL System

[[SIGMOD 2017\]Spanner Becoming a SQL System--学习笔记](https://zhuanlan.zhihu.com/p/625805991)

Spanner是一个面向全球的分布式DBMS，为谷歌数百项关键服务提供数据支撑。谷歌在OSDI‘12上发表了第一篇Spanner的论文，那个时候的Spanner可以理解分布式KV数据库，支持动态扩缩容、自动分区、多版本、全球部署、基于Paxos的多副本机制、支持外部一致性的分布式事务（SSI），详见[OSDI '12] Spanner Google’s Globally-Distributed Database。本文讲的是Spanner后续实现的分布式SQL引擎，包括分布式查询执行、临时故障时查询重启、如果根据查询数据的范围将请求路由到数据所在的range（数据分片）、索引的查询机制以及在存储方面的优化。

目前，两个著名的分布式NewSQL数据库tidb和cockroach，就是inspire by Spanner 和 F1。

### （7）向量化执行引擎

CRDB向量化执行引擎inspire by MonetDB-X100，但未使用SIMD。

[[CIDR 2005\]MonetDB-X100 Hyper-Pipelining Query Execution--学习笔记](https://zhuanlan.zhihu.com/p/640004245)

[Balancing Vectorized Query Execution with Bandwidth-Optimized Storage-第四章MonetDB/X100 overview--学习笔记](https://zhuanlan.zhihu.com/p/640267685)

[Balancing Vectorized Query Execution with Bandwidth-Optimized Storage-第五章MonetDB/X100向量化执行引擎](https://zhuanlan.zhihu.com/p/640518615)

## **Hyper/DuckDB**

Hyper提出了很多经典算法，但未开源，不过这些算法基本都在DuckDB中实现了。

[Unnesting Arbitrary Queries--Hyper去相关子查询论文学习](https://zhuanlan.zhihu.com/p/613010202)

SQL 99中允许子查询出现在SELECT、FROM和WHERE子句中，从用户使用的角度看，嵌套子查询能够极大地简化复杂查询的形式。但是，在相关子查询中，优化器不加处理，直接传递给执行器计算，其算法复杂度为N方，数据量大的情况下，性能极差。paper提出来了一种去相关子查询的通用且高效算法。

[[BTW 2017\]The Complete Story of Joins (in HyPer)--HyPer去相关子查询论文学习](https://zhuanlan.zhihu.com/p/614953165)

这篇论文是《[BTW 2015] Unnesting Arbitrary Queries》的延续，在去相关子查询方面，提出了两个新的Join扩展算子mark-join和single-join，并给出了逻辑上的等价变换规则，物理上的实现算法。

[[SIGMOD 2008\]Dynamic Programming Strikes Back--读书笔记](https://zhuanlan.zhihu.com/p/632565843)

DPCCP算法与DPsize和DPsub算法相比优势在于能够高效获取csg-cmp-pairs：

- DPCCP算法

- - 连通子图，在query graph中，依次以base tables为种子，步长为1，采用广度优先的方法，对每个种子节点求邻近节点集合，邻近节点的编号必须大于当前节点，这样保证去重，然后基于种子节点和第1级邻近节点集合中的所有非空子集构成种子节点步长为1的所有连通子图，然后对于其中的每个连通子图，再采用类似的方法，步长为1，扩展获得下一级所有的连通子图，以此类推，就可以获取query graph中的所有连通子图
  - 连通子图的补图，对于任意的连通子图，以其邻居节点集合中的每个节点为种子节点，采用了和连通子图扩展相似的方式，获取所有的连通子图，其中，扩展时要排除掉原连通子图，并且仅选择比种子节点编号大的节点
  - DPCCP通过上述方法，直接获取了csg-cmp-pairs，高效先进

- DPsize和DPsub

- - 采用了枚举方式，获取节点集合对，(S1,S2)，再判断S1和S2各自是否是连通的，是否存在交集，S1和S2之间是否连通，这样才能得到csg-cmp-pairs，效率要低

DPCCP算法存在如下两个缺陷，使其不能直接应用于数据库实践中，可以理解为是一种学术性的创新。

1. 仅能处理inner join，不支持full join，left join，right join，semi join，anti join等
2. 仅能处理简单的join谓词，即join的left input或right input仅包含1个base table，无法处理这种join两个是多个base tables的复杂join谓词，t1.a = t2.b + t3.d

本文是DPCCP算法的延续，开发了一种新的算法DPhyp，能够保留DPCCP在寻找csg-cmp-pairs方面的优势，并解决DPCCP的2个缺陷。

MySQL 8.0中已经实现了DPhyp算法。CockroachDB采用了DPSube算法查找best join order，但将DPhyp列为了TODO。

[[VLDB 2015\] How Good Are Query Optimizers, Really](https://zhuanlan.zhihu.com/p/628138716)

论文的作者是大名鼎鼎的CWI实验室Peter Boncz教授、慕尼黑工业大学数据库研究组长Thomas Neumann教授和Alfons Kemper博士，其中，Peter Boncz教授是MonetDB的创始人之一，Thomas Neumann是HyPer的创始人之一。

论文认为，Join orders对查询的性能非常重要。优化器中，cardinality estimator（基数估计）组件、cost model(代价模型)、 plan enumeration（plan的枚举技术）是查找好的Join order的核心组件。论文设计并实施了Join Order Benchmark (JOB)，即Join Order基准测试，实验结果表明：

- cardinality estimation（基数估计）组件：往往产生较大的误差
- cost model(代价模型)：其对性能的影响要比cardinality estimator（基数估计）小的多
- plan enumeration（plan的枚举技术）：对比了穷举动态规划和启发式算法，发现在次优的基数估计情况下，穷举方法能够改善查询性能

最终，论文得出结论，过度依赖cardinality estimation（基数估计）组件、cost model(代价模型)、plan enumeration（plan的枚举技术），不能得到让人满意的性能。

[[VLDB 2011\]Efficiently Compiling Efficient Query Plans for Modern Hardware --学习笔记](https://zhuanlan.zhihu.com/p/623906536)

这篇Paper的作者是著名的Thomas Neumann，慕尼黑工业大学（TMU）的数据库研究组组长，HyPer数据库的初始人之一。论文讲述了如何在执行器中，编译执行查询计划，CodeGen概念就是出自这篇论文。如果对PostgreSQL、Opengauss等数据库的JIT技术感兴趣的话，推荐阅读原文。

论文认为，随着内存硬件的进步，查询性能越来越取决于查询处理本身的原始CPU成本。经典的火山迭代器模型虽然技术非常简单和灵活，但由于缺乏局部性（locality）和频繁的指令错误预测，在现代cpu上的性能较差。过去已经提出了一些技术，如面向批处理的处理或向量化tuples处理策略来改善这种情况，但即使是这些技术也经常被hand-written计划执行。

这篇论文提出了一种新的编译策略，它使用LLVM编译器框架将查询转换为紧凑而高效的机器代码。通过针对良好的代码和数据局部性和可预测的分支布局，生成的代码经常可以与手写的C++代码相媲美。将这些技术已经集成到HyPer数据库系统中，获得了很好的查询性能，而只需要增加适度的编译时间。

## **Calcite**

[[SIGMOD 2018\] Apache Calcite 论文学习笔记](https://zhuanlan.zhihu.com/p/624607251)

Apache Calcite是一个基础的软件框架，提供SQL 解析、查询优化、数据虚拟化、数据联合查询 和 物化视图重写等功能，目前已经应用于多款开源数据库或大数据处理系统中，例如

- Apache Hive，Apache Storm，Apache Flink，Druid 和 MapD
- Polar-X、Doris、StarRocks、SelectDB

Calcite框架组成如下：

- 查询优化器，内置几百种优化规则，模块化且可扩展
- 查询处理器，支持多种查询语言
- 适配器模式的体系结构，支持多种异构数据模型（关系结构格式、半结构化格式、流格式、地理信息格式）

Calcite框架具有灵活性、可嵌入性、可扩展性等特点，因此，对于涉及多源异构类型的大数据框架来说，很具吸引力。Apache Calcite项目非常活跃，持续支持新型数据源、查询语言、查询处理以及查询优化方法。

[[ICDE 1993\] The Volcano Optimiz](https://zhuanlan.zhihu.com/p/590785118)er Generator- Extensibility and Efficient Search （Calcite优化器架构）

Volcano模型旨在解决扩展和性能问题。EXODUS是同一个作者在Volcano的一个优化器设计的版本，EXODUS虽然已经证明了优化器生成器范式的可行性和有效性，但很难构建高效的、满足生产环境质量的优化器。Volcano也是在EXODUS的基础之上优化而来。Volvano在EXODUS上面的做的一些优化： 前者(EXODUS)没有区分逻辑表达式和物理表达式，也没有物理Properties的概念，没有一个通用的Cost函数等等。Volcano优化器要满足如下目标：

1. 这个新的优化器生成器必须能够使用Volcano Executor，也可以在其他项目中作为独立的工具
2. 必须更加高效，比较低的时间消耗和内存消耗
3. 必须提供有效的，高效的，和可扩展的物理属性支持，如sort order、compression
4. 必须允许使用启发式和数据模型语义来指导搜索并删除搜索空间中无用的部分（剪枝）
5. 它必须支持灵活的代价模型，对于查询中的子项能够产生动态查询计划

## **GreenPlum**

[[SIGMOD 2014\] Orca: A Modular Query Optimizer Architecture for Big Data 论文阅读](https://zhuanlan.zhihu.com/p/590795795)

Orca是Pivotal公司基于Cascade优化器框架开发的SQL优化器服务，以独立的Service形态单独存在，并不依附于特定的数据库产品，对外是标准化的接口和协议，调用者通过标准化的输入，由Orca负责根据输入指定SQL执行计划，由调用者将这个基于Orca的SQL执行计划转换成自身数据库的SQL执行计划，这样理论上可以被集成到任何数据库系统中。已经应用于Greenplum和基于Hadoop的HAWQ中。是Volcano/Cascades的论文思想的一个业界著名的工程化实现。

[[IEEE Data engineering Bulletin 1995\] The Cascades Framework for Query Optimization论文阅读](https://zhuanlan.zhihu.com/p/590793684)，支持Cascade架构

[[SIGMOD 1979\] Access Path Selection in a Relational Database Management System 论文阅读](https://zhuanlan.zhihu.com/p/590776990)，支持System R架构

## **MemSQL**

[[VLDB 2016\] The MemSQL Query Optimizer，HTAP优化器论文阅读](https://zhuanlan.zhihu.com/p/594101717)

当前主流优化器有2种，[System R](https://zhuanlan.zhihu.com/p/590776990)（bottom-up）和[Volcano/Cascades](https://zhuanlan.zhihu.com/p/590793684)（top-down），二者各有千秋。 但对于MemSQL来说，皆无法满足需求。

这是因为，MemSQL是一个高性能、分布式HTAP数据库，需要解决OLAP实时负载分析问题，要求SQL优化时间非常有限，现有优化器框架不能满足这个需求，集成已有优化器框架还意味着继承框架的所有缺点和集成挑战。因此，MemSQL从头开始构建，采用了创新的工程，如内存优化锁(lock - free)和能够运行实时流分析的列存储引擎(columnstore engine)，开发了一个功能丰富的查询优化器，能够跨各种复杂的企业工作负载生成高质量的查询执行计划。

## **MonetDB/X100**

[[CIDR 2005\]MonetDB-X100 Hyper-Pipelining Query Execution--学习笔记](https://zhuanlan.zhihu.com/p/640004245)，简介见上文

[Balancing Vectorized Query Execution with Bandwidth-Optimized Storage-第四章MonetDB/X100 overview--学习笔记](https://zhuanlan.zhihu.com/p/640267685)，简介见上文

[Balancing Vectorized Query Execution with Bandwidth-Optimized Storage-第五章MonetDB/X100向量化执行引擎](https://zhuanlan.zhihu.com/p/640518615)，简介见上文

[[DaMoN 2011\]Vectorization vs. Compilation in Query Execution--学习笔记](https://zhuanlan.zhihu.com/p/641268363)

在关系数据库中，查询执行一般有三种策略，解释型执行、编译执行和向量化执行。在OLAP领域，编译执行和向量化执行要优于解释执行的方式。但编译执行和向量化执行哪种更佳？如何基于现代CPU，为向量化执行引擎配置好编译执行的策略，获取最佳性能？围绕这个问题，这篇论文就行了研究分析。作者针对Project、Select、Hash join算子深入调研分析了VectorWise（MonetDB X100的商业化版本）数据库的向量化执行引擎和编译策略。调研发现

- 对于block-wise 查询执行引擎来说，最好使用编译执行的方式

- 找到了三种场景下，“loop-compilation”策略不如向量化执行引擎

- 如果能较好的将编译执行和向量化执行结合，可以获取更好的性能

- - either by incorporating vectorized execution principles into compiled query plans
  - using query compilation to create building blocks for vectorized processing

## **优化器测试论文**

[[DBTest 2012\] Testing the Accuracy of Query Optimizers论文阅读](https://zhuanlan.zhihu.com/p/590798843)

查询优化器的准确性与数据库系统性能及其操作成本密切相关:优化器的成本模型越准确，得到的执行计划就越好。数据库应用程序程序员和其他从业者长期以来提供了一些轶事证据，表明数据库系统在优化器的质量方面存在很大差异，但迄今为止，数据库用户还没有正式的方法来评估或反驳这种说法。

在本文中，我们开发了一个TAQO 框架来量化给定工作负载下优化器的准确性。我们利用优化器公开开关或hint，让用户控制查询计划，并生成除默认计划之外的计划。使用这些实现，我们为每个测试用例强制生成多个替代计划，记录所有替代计划的执行时间，并根据它们的有效成本对执行计划进行排序。我们将这个排名与估计成本的排名进行比较，并为优化器的准确性计算一个分数。

我们给出了对几个主要商业数据库系统进行匿名比较的初步结果，表明系统之间实际上存在实质性差异。我们还建议将这些知识纳入商用数据库开发过程。

[[CIKM 2016\] OptMark- A Toolkit for Benchmarking Query Optimizers--优化器评价论文学习](https://zhuanlan.zhihu.com/p/626262614)

如何评估数据库优化器的质量，这个问题非常有挑战性。目前，现有数据库的benchmarks测试（如TPC benchmarks）是对整体的query的性能评估，并不是针对优化器。这篇论文描述了OptMark，一个用于评估优化器质量的toolkit，可惜没有开源。论文从Effectiveness（有效性） 和Efficiency（效率）两个维度，评价优化器的质量，并给出了具体的评价公式。

[[SIGMOD '00\]Counting, enumerating, and sampling of execution plans in a cost-based query optimizer学习](https://zhuanlan.zhihu.com/p/626372143)

在对优化器的测试中，一项重要的测试内容是，对比分析优化器选出的plan的执行时间与优化器中其他候选plans的执行时间，进而判断优化器从候选plans中选择较佳的plan的能力，那么问题就来了，如何才能知道优化器中，其他候选plans的执行时间？

这篇论文给出了一个很好的方法，就是在优化器制定query的执行计划过程中，建立好数字和候选plans的对应关系，在SQL语句执行时，用户可以输入数字，指定执行plan。这个技术已经应用到SQL Server的优化器测试验证中了。

由于SQL Server采用了Cascade框架，因此，整个实现是围绕MEMO结构进行的。

[[Data Engineering Bulletin '08\]Testing SQL Server's Query Optimizer--学习笔记](https://zhuanlan.zhihu.com/p/626015214)

查询优化本身就很复杂，因此，验证优化器的正确性和有效性是一项非常复杂的工作。正确性是说生成的查询计划对标原sql，能不能保证结果集的正确性；有效性是说，对原sql执行RBO+CBO变换后，得到的执行计划是不是高效的。

这篇论文描述了测试查询优化器时遇到的一些独特问题，概述了验证微软SQL Server的查询优化器的测试技术，给出了优化器测试过程中的成功经验和教训，最后讨论了测试器测试中持续存在的挑战。

论文更多的是从测试工程的角度说明优化器测试问题，以及微软SQL Server是如何测试优化器的，给出的方法相对要宏观一些，或者说更像是一种方向性的指引。很遗憾，可能是出于商业保密的原因，论文并没有公开优化器测试的实现机制。论文[DBTest 2012] Testing the Accuracy of Query Optimizers，给出了优化器测试的数学模型，但没有从系统工程的角度说明如何进行优化器的测试。这两篇论文有很强的互补性，都是很值得学习的。

## **统计信息**

[[SIGMOD 1996\] Improved Histograms for Selectivity Estimation of Range Predicates](https://zhuanlan.zhihu.com/p/634919204)

许多商业数据库会维护直方图，用直方图来概括关系表的内容的特征，以此高效评估结果集大小，以及评估各种查询计划的代价。尽管过去已经提出了几种直方图，但还没有关于直方图的系统研究的论文，还有直方图的选型对效率的影响等。本文中作者提出了一个直方图的分类方法，这个分类能包括现有的直方图，并指出了一些新的直方图的可能性。我们为几个分类维度引入了新的选项，并通过以有效的方式组合这些选项来推导出新的直方图类型。我们还展示了如何使用采样技术来降低直方图构建的成本。最后，我们给出了用于范围谓词选择性估计的直方图类型的实证研究结果，并确定了具有最佳整体性能的直方图类型。

[[VLDB 2003\] The History of Histograms--学习笔记](https://zhuanlan.zhihu.com/p/634028032)

直方图的历史悠久而且内容丰富，每一步都有详细的信息。直方图的历史包括

- 不同科学领域的直方图的过程
- 直方图在近似和压缩信息方面的成功经验和失败教训
- 直方图的工业应用
- 以及在各种直方图相关问题上给出的解决方案

在本文中，本着与直方图技术本身的相同精神，我们将直方图的整个历史（包括它们目前预期的“未来历史”）压缩到给定/固定的budget中，主要periods、events和results的最高（个人偏见）兴趣的细节。在一组有限的实验中，发现直方图的历史的压缩形式和完整形式之间的语义距离相对较小！

[[VLDB 2009\] Preventing Bad Plans by Bounding the Impact of Cardinality Estimation Errors](https://zhuanlan.zhihu.com/p/635468681)

对于基数估计来说，q-error的指标Iq是一个更有意义的度量指标

- 如果构建的直方图能够保证q-error的阈值，则优化器能够对查询优化结果进行一定程度的保证
- 如果q-error不能满足阈值，代价评估的误差也能够在q^4

论文开发了一种能够最小化Iq的直方图构建算法

- 基于快速迭代的算法构造直方图的单桶
- 采用了多段拟合方式，适用范围更广
- 使用二分法构建直方图的多个桶
- 提供了比其他方法更好的近似值
- 特别是，更稳健，plan的质量是有边界的！

## **其他**

[[SIGMOD 2015\]Rethinking SIMD Vectorization for In-Memory Databases--学习笔记](https://zhuanlan.zhihu.com/p/640818697)

论文作者来自哥伦比亚大学和Oracle。论文基于先进的SIMD指令，例如gathers和scatters，提出了一种新的向量化执行引擎，并给出了selection scans、hash tables、partition的向量化实现代码，在这些基础上，实现了sort和join等算子。论文将提出的向量化实现技术与最先进的scalar和vectorized技术进行了对比，通过在Xeon Phi以及主流CPU的实测结果，证实论文提出的技术的高性能。

[从TPC-H分析论文学习优化器的挑战与应对思路](https://zhuanlan.zhihu.com/p/608897702)

论文[[tpctc 2013\]TPC-H Analyzed Hidden Messages and Lessons Learned from an Influential Benchmark](https://link.zhihu.com/?target=https%3A//homepages.cwi.nl/~boncz/snb-challenge/chokepoints-tpctc.pdf%3Fspm%3Da2c6h.12873639.article-detail.68.4829427f4yUJQ5%26file%3Dchokepoints-tpctc.pdf)更像是查询优化器核心算法的骨架，虽然没有深入描述某个技术，但以TPC-H的query为例，系统且深入的分析了复杂query中提升性能的关键点，并给出了对应的优化思路。

[[SIGMOD 2018\]How to Architect a Query Compiler, Revisited --学习笔记](https://zhuanlan.zhihu.com/p/622929275)

为了充分利用现代硬件平台，越来越多的数据库系统将执行计划编译为机器码，例如引入llvm。在学术界，关于构建此类查询编译器的最佳方法，一直存在争论，认为这是一项非常困难的工作，与传统的解释型执行计划相比，需要采用完全不同的实现技术。

论文的目标是引入Futamura projections（二村映射）技术，它从根本上将解释器和编译器联系起来。在这个想法的指导下，作者开发了LB2，一种高级的查询编译期，它的性能与标准TPC-H基准测试上的最佳编译查询引擎相当，有时甚至更好。

[从优化器综述论文学习System-R框架和Cascade框架](https://zhuanlan.zhihu.com/p/611439035)

在数据库中，优化器之所以重要，是因为SQL是一种声明式的语言，只是告诉用户需要什么数据，但获取数据往往有多条路径，路径间代价差异可能会很大，优化器则是负责快速、高效的找出获取数据的最佳执行路径，是数据库的大脑，可谓得优化器者得数据库天下。

论文An Overview of Query Optimization in Relational Systems写于1998年，虽然古老，但切实抓住了70年代以来的优化器的要点，综述了2种优化器框架，System-R（bottom-up）和Cascade（top-down）。

[[VLDB,1997\] SEEKing the truth about ad hoc join costs--学习笔记](https://zhuanlan.zhihu.com/p/638581252)

在本文中，作者开发了一个详细的代价模型，可以来预测join算法的性能。作者使用这个代价模型研究每种join方法的最优的缓冲区分配方案。实验结果表明，在模型情况下，性能可以提升400%。

这篇论文细致的去分析了多种Join算法的代价模型，主要是给出了分析的过程，这对于学习数据库代价评估模型非常有帮助，可以知其所以然。否则，只是知道代码里的一系列公式，为什么是这样子，理解的不一定到位。

个人理解：这篇论文的题目中

- “seek the tuth”：其实就是说ad hoc join查询的代价都有哪些，为什么会有这些代价，这些代价在不同的可用内存情况下，是什么表现，因此最终可以给出各种join算法的最优内存评估，这些可以构成资源管理中，不同join算法内存预估的理论基础，研读这方面的实现机制时，可以用来参考
- ad hoc join：即席查询，也就是说一次query执行一次编译并生成执行计划，不会使用查询计划的缓存。即席查询主要用于AP场景，对于跑批业务来说，业务是固定的，很多时候，可以使用plan cache，但对于其中，突然新增的一个sql且仅执行一次，即常规业务外的sql，需要直接编译执行

学习这篇论文花了好多精力，但在理解代价评估模型的原理方面，收益很大。之前也看过一些关于数据库原理、优化器原理以及实现的方面的书籍、论文、技术博文，包括一些数据库的实现，但在代价模型原理方面总有一种总感觉不够透彻，这篇论文个人感觉还是很深刻的。

[开源优化器实现资料汇总](https://zhuanlan.zhihu.com/p/609987395)



 ## 参考

[SQL引擎发表、落地论文总结](https://zhuanlan.zhihu.com/p/642501923)