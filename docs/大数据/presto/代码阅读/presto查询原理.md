# presto查询引擎

---



> 注：本文参考知乎这篇文章，https://zhuanlan.zhihu.com/p/293775390
>
> 本文的trino版本是423

## 从架构上看SQL Query的执行流程

![Snipaste_2023-08-21_16-55-53.png](../../img/Snipaste_2023-08-21_16-55-53.png)

请参见上面的架构图，从用户开始写SQL开始到查询结果返回，我们划分出以下几个部分：

- SQL Client：用户可以在这里输入SQL，它负责提交SQL Query给Presto集群。SQL Client一般用Presto自带的Presto Client比较多，它可以处理分批返回的结果，并在终端展示给用户。
- External Storage System：由于Presto自身不存储数据，计算涉及到的数据及元数据都来自于外部存储系统，如HDFS，AWS S3等分布式系统。在企业实践经验中，经常使用HiveMetaStore来存存储元数据，使用HDFS来存储数据，通过Presto执行计算的方式来加快Hive表查询速度。
- Presto Coordinator：负责接收SQL Query，生成执行计划，拆分Stage和Task，调度分布式执行的任务到Presto Worker上。
- Presto Worker：负责执行收到的HttpRemoteTask，根据执行计划确定好都有哪些Operator以及它们的执行顺序，之后通过TaskExecutor和Driver完成所有Operator的计算。如果第一个要执行的Operator是SourceOperator，当前Task会先从External Storage System中拉取数据再进行后续的计算。如果最后一个执行的Operator是TaskOutputOperator，当前Task会将计算结果输出到OutputBuffer，等待依赖当前Stage的Stage来拉取结算结果。整个Query的所有Stage中的所有Task执行完后，将最终结果返回给SQL Client。















## 参考资料

- 本文参考知乎这篇文章，https://zhuanlan.zhihu.com/p/293775390

- 本文用于举例的SQL来自于TPC-DS Benchmark标准Query55，详见 [http://www.tpc.org/tpcds/](https://link.zhihu.com/?target=http%3A//www.tpc.org/tpcds/)

- Presto技术源码解析总结-一个SQL的奇幻之旅(上) [https://www.jianshu.com/p/3fccfa82e](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/3fccfa82e1ec)

- Presto技术源码解析总结-一个SQL的奇幻之旅(下) [https://www.jianshu.com/p/d8a3d7488](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/d8a3d7488358)

- 数据库大神 Goetz Graefe与他的论文 [https://scholar.google.com/cita](https://link.zhihu.com/?target=https%3A//scholar.google.com/citations%3Fuser%3DpdDeRScAAAAJ%26hl%3Den)

- SQL 优化之火山模型 [https://zhuanlan.zhihu.com/p/21](https://zhuanlan.zhihu.com/p/219516250)

- Facebook Presto论文，Presto: SQL on Everything

- Volcano 执行模型以及如何用Code Generation与列式存储优化SparkSQL的执行效率 [https://databricks.com/blog/201](https://link.zhihu.com/?target=https%3A//databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html)

- 代码生成技术（二）查询编译执行 https://zhuanlan.zhihu.com/p/58249033

- SQL 查询的分布式执行与调度 https://zhuanlan.zhihu.com/p/100949808

- Volcano执行模型论文解读 [https://zhuanlan.zhihu.com/p/34](https://zhuanlan.zhihu.com/p/34220915)

- Vectorized and Compiled Queries — Part 1 [https://medium.com/@tilakpatidar](https://link.zhihu.com/?target=https%3A//medium.com/%40tilakpatidar/vectorized-and-compiled-queries-part-1-37794c3860cc)

- Vectorized and Compiled Queries — Part 2 [https://medium.com/@tilakpatida](https://link.zhihu.com/?target=https%3A//medium.com/%40tilakpatidar/vectorized-and-compiled-queries-part-2-cd0d91fa189f)

- Vectorized and Compiled Queries — Part 3 [https://medium.com/@tilakpatida](https://link.zhihu.com/?target=https%3A//medium.com/%40tilakpatidar/vectorized-and-compiled-queries-part-3-807d71ec31a5)

- Presto Core Data Structures: Slice, Block & Page [https://zhuanlan.zhihu.com/p/60](https://zhuanlan.zhihu.com/p/60813087)

- Presto 数据如何进行shuffle [https://zhuanlan.zhihu.com/p/61](https://zhuanlan.zhihu.com/p/61565957)

- Presto 由Stage到Task的旅程 https://zhuanlan.zhihu.com/p/55785284

  