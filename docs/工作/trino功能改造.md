# trino任务

- **calcite物理计划交给谁执行**

- hdfs那debug，尝试下如果没有hadoop配置，启动trino会出问题吗(会读取环境变量，根据环境变量Hadoop home的目录，去读取etc/hadoop下的配置文件)，所以只需要看下trino是怎么读取hdfs中文件的

- use hive.default;describe test;debug看下这个hive表的string是怎么在trino中变为varchar的

- trino plugin加载机制（看看后面怎么做hive udf的动态加载）

- q72查不出来

- 物化视图

- 为什么过多的join会导致查询过慢

- 编译gluten

- presto的算子、资源管理和调度、查询优化、shuffle过程

- sql中的关系代数

- trino对hudi mor的优化(目前是非分区表在update后查询不到update的数据，分区表是spark没有在metadata中写入相关信息)

- connector加载机制

- trino中优化后的逻辑计划

- 对group by的优化，目前初步想法是参考https://github.com/trinodb/trino/issues/14237（hummingbird）中的向量化方法

- trino对hudi mor的优化(目前是非分区表在update后查询不到update的数据，分区表是spark没有在metadata中写入相关信息)

- trino 的connector与function机制的实现 hive udf的动态加载

  

  

  

# 字节

https://xie.infoq.cn/article/ef46f810f0d57fd14cd48b6e5



# 美团

[https://zhuanlan.zhihu.com/p/408957032](https://zhuanlan.zhihu.com/p/408957032)





# 滴滴



# 携程


链接：https://zhuanlan.zhihu.com/p/41538472

**性能方面**

- 根据Hive statistic信息，在执行查询之前分析hive扫描的数据，决定join查询是否采用Broadcast join还是map join。
- Presto Page在多节点网络传输中开启压缩，减少Network IO的损耗，提高分布计算的性能。
- 通过优化Datanode的存储方式，减少presto扫描Datanode时磁盘IO带来的性能影响。
- Presto自身参数方面的优化。

**安全方面**

- 启用Presto Kerberos模式，用户只能通过https安全协议访问Presto。
- 实现Hive Metastore Kerberos Impersonating 功能。
- 集成携程任务调度系统(宙斯)的授权规则。
- 实现Presto客户端Kerberos cache模式，简化Kerberos访问参数，同时减少和KDC交互。

**资源管控方面**

- 控制分区表最大查询分区数量限制。
- 控制单个查询生成split数量上限, 防止计算资源被恶意消耗。
- 自动发现并杀死长时间运行的查询。

**兼容性方面**

- 修复对Avro格式文件读取时丢失字段的情况。
- 兼容通过Hive创建 view，在Presto上可以对Hive view 做查询。(考虑到Presto和Hive语法的兼容性，目前能支持一些简单的view)。
- 去除Presto对于表字段类型和分区字段类型需要严格匹配的检测。
- 修复Alter table drop column xxx时出现ConcurrentModification问题。



# sqlscan工具

https://mp.weixin.qq.com/s/Sa1jI_-1fxNOLQxhi24BRg







# gateway智能选择查询引擎

Presto 接入了查询路由 Gateway，Gateway 会智能选择合适的引擎，用户查询优先请求 Presto，如果查询失败，会使用 Spark 查询，如果依然失败，最后会请求 Hive。在 Gateway 层，我们做了一些优化来区分大查询、中查询及小查询，对于查询时间小于 3 分钟的，我们即认为适合 Presto 查询，比如通过 HBO（基于历史的统计信息）及 JOIN 数量来区分查询大小，架构图如下：

https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247497505&idx=1&sn=6ae3547d253cf76afe4c813d850e999d&chksm=fd3eb1b4ca4938a253b7f0d54d35eaf110890f24d846a5c9ee85e48a2884bb4ad526ea0eccb1&scene=21#wechat_redirect





# 其它

Hive SQL 兼容

  隐式类型转换

  语义兼容

  语法兼容

  支持 Hive 视图

  Parquet HDFS 文件读取支持

  大量 UDF 支持

  其他

物理资源隔离（滴滴用的打标签）

直连Druid 的 Connector

多租户等