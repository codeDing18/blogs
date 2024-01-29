# 使用

web启动命令`sh bin/seatunnel-backend-daemon.sh start`

seatunnel启动命令`./bin/seatunnel-cluster.sh -d`



# 部署seatunnel web

目前1.0.0web刚推出来，部署后发现还有很多错误并且还没有官方文档，遂放弃使用web





# 部署seatunnel

**下载的2.3.3版本**，按照[官方文档部署](https://seatunnel.apache.org/docs/2.3.3/start-v2/locally/deployment)，在执行[quick-start-seatunnel-engine文档](https://seatunnel.apache.org/docs/2.3.3/start-v2/locally/quick-start-seatunnel-engine)中`./bin/seatunnel.sh --config ./config/v2.batch.config.template -e local`

**如果报如下错误**

`Plugin PluginIdentifier{engineType='seatunnel', pluginType='source', pluginName='connector-fake'} not found`

> 解决方法：先检查下connectors/seatunnel中对应的jar包是否存在（如果sh bin/install-plugin.sh 2.3.3很慢的话，可以手动从[Apache Maven Repository](https://repo.maven.apache.org/maven2/org/apache/seatunnel/)下载），然后看下connectors/plugin-mapping.properties中是否存在对应的connector-fake

**如果报如下错误**

![image-20231116163614096](/Users/ding/Library/Application Support/typora-user-images/image-20231116163614096.png)

> 解决方法：下载[file](https://repo.maven.apache.org/maven2/org/apache/seatunnel/seatunnel-hadoop3-3.1.4-uber/2.3.2/seatunnel-hadoop3-3.1.4-uber-2.3.2-optional.jar)，然后将jar包放入到seatunnel/lib目录下



# hive2doris

在将hive数据通过seatunnel导入到 Doris时，在[maven](https://repo.maven.apache.org/maven2/org/apache/seatunnel/)下载seatunnel-hadoop3-3.1.4-uber.jar and hive-exec-2.3.9.jar放在`seatunnel/lib` 目录下。然后执行`bin/seatunnel.sh -c config/hive2doris.config -e local`

**如果报以下错误**

![image-20231117090817283](/Users/ding/Library/Application Support/typora-user-images/image-20231117090817283.png)

> 解决方法：将seatunnel-hadoop3-3.1.4-uber.jar and hive-exec-2.3.9.jar放在seatunnel/plugin/hive/lib下（hive/lib需自己手动创建）

**如果报以下错误**

![image-20231117091057813](/Users/ding/Library/Application Support/typora-user-images/image-20231117091057813.png)

> 解决方法：我的hive是3.1.3，所以先将hive-exec-2.3.9.jar替换成自己hive/lib下的hive-exec-3.1.2.jar。但还是报错，后面发现目前官方的connector-hive只与hive2兼容，所以hive只能选取对应版本。

**如果报以下错误**

`org.apache.seatunnel.engine.server.checkpoint.CheckpointException: Checkpoint expired before completing. Please increase checkpoint timeout in the seatunnel.yaml`

> 解决方法：[社区issue](https://github.com/apache/seatunnel/issues/5722)

**如果报以下错误**

![image-20231123194557696](/Users/ding/Library/Application Support/typora-user-images/image-20231123194557696.png)

> 解决办法：我这里是在config中填了具体的分区，然后报上述错误，这是2.3.3的bug，社区将在下一版本中解决这问题，[社区issue](https://github.com/apache/seatunnel/issues/5484)

**如果报以下错误**

![image-20231123200238407](/Users/ding/Library/Application Support/typora-user-images/image-20231123200238407.png)

> 解决办法：hive表如果是text格式，在config中必须指定delimiter（官网上没有这个参数，源码里才有）。如果是orc、parquet格式则不需要。源码见下图。

![image-20231123201232280](/Users/ding/Library/Application Support/typora-user-images/image-20231123201232280.png)

# Hdfs2doris

**如果报以下错误**

![image-20231123195732736](/Users/ding/Library/Application Support/typora-user-images/image-20231123195732736.png)

>解决办法：这个是hive的text格式的表格，所以在config中需要指定schema，但如果是orc、parquet格式的表格就不需要指定schema

**如果报以下错误**

![image-20231123181632867](/Users/ding/Library/Application Support/typora-user-images/image-20231123181632867.png)

> 解决办法：检查下hdfs路径下是否有些目录没有数据文件

# mysql2doris

**如果报以下错误**

![image-20231117141338083](/Users/ding/Library/Application Support/typora-user-images/image-20231117141338083.png)

> 解决方法：从[maven](https://repo.maven.apache.org/maven2/org/apache/seatunnel/)下载connector-jdbc.jar包，然后将jar包放入seatunnel/connectors/seatunnel/下，并且修改seatunnel/connectors/plugin-mapping.properties，根据自己需求是sink还是source添加connector-jdbc



# 原理

hive、hdfs当作数据源的话，其原理是读取hdfs路径中的文件（下图展示的是读取text格式，如果是orc、parquet格式也有对应的read方法），然后写入到doris中。如果是jdbc方式的话，需要写query。

![image-20231123201528387](file:///Users/ding/Library/Application%20Support/typora-user-images/image-20231123201528387.png?lastModify=1700741726)

下图展示的是读取hive或者hdfs数据源时hdfs路径下的文件，**如果路径下没有文件就会报错，并且源码展示了哪些文件名会被过滤掉**

![image-20231123232926931](/Users/ding/Library/Application Support/typora-user-images/image-20231123232926931.png)





# 测试结果

| 数据源                  | 表大小 | 数据行数 | 文件数 | 并行度 | 时间 |
| ----------------------- | ------ | -------- | ------ | ------ | ---- |
| hive:tpch.partsupp      | 1.2g   | 800万    | 1      | 1      | 18s  |
| hive:tpch.partsupp      | 1.2g   | 800万    | 1      | 4      | 19s  |
| hive:tpch.partsupp_clus | 0.3g   | 800万    | 19     | 1      | 29s  |
| hive:tpch.partsupp_clus | 0.3g   | 800万    | 19     | 4      | 28s  |
| hive:tpch.partsupp_clus | 0.3g   | 800万    | 19     | 8      | 20s  |
| hive:tpch.partsupp_clus | 0.3g   | 800万    | 19     | 16     | 15s  |
| hive:tpch.orders        | 1.7g   | 1500万   | 1      | 1      | 37s  |
| hive:tpch.orders1       | 0.5g   | 1500万   | 27     | 1      | 38s  |
| hive:tpch.orders1       | 0.5g   | 1500万   | 27     | 8      | 38s  |
| hive:tpch.orders1       | 0.5g   | 1500万   | 27     | 16     | 35s  |
| Hdfs:tpch.orders        | 1.7g   | 1500万   | 1      | 1      | 45s  |
| Hdfs:tpch.orders        | 1.7g   | 1500万   | 1      | 2      | 39s  |
| Hdfs:tpch.orders        | 1.7g   | 1500万   | 1      | 4      | 38s  |
| Hdfs:tpch.orders        | 1.7g   | 1500万   | 1      | 8      | 41s  |
| Hdfs:tpch.orders        | 1.7g   | 1500万   | 1      | 20     | 41s  |
| Hdfs:tpch.orders1       | 0.5g   | 1500万   | 27     | 1      | 36s  |
| Hdfs:tpch.orders1       | 0.5g   | 1500万   | 27     | 4      | 36s  |







# 测评

- hive的分区表导入到Doris时，如果指定具体的分区会有bug，社区下个版本完善
- 目前的web界面还无法使用，官网也没文档。写入到doris的ddl以及作业的config需要用户自己写，用户体验可能较差
- 如果表中有个文件很大，比如这个文件是7.4g并且这个文件数据行数超过1亿，同步这个表时会出错，社区下个版本完善
- 目前只支持hive2.3.9，hive3只支持以jdbc方式连接，需要写query（数据大的话在hive中查询会较慢）









![image-20231121205955156](/Users/ding/Library/Application Support/typora-user-images/image-20231121205955156.png)



![image-20231121221439900](/Users/ding/Library/Application Support/typora-user-images/image-20231121221439900.png)



![](/Users/ding/Library/Application Support/typora-user-images/image-20231121222517890.png)



![image-20231128091640527](/Users/ding/Library/Application Support/typora-user-images/image-20231128091640527.png)



![image-20231128091703078](/Users/ding/Library/Application Support/typora-user-images/image-20231128091703078.png)

# 参考

- https://codeantenna.com/a/5EW50kVYLx

- https://zhuanlan.zhihu.com/p/635910859

- https://github.com/apache/seatunnel/issues/5423

- [多表加载](https://zhuanlan.zhihu.com/p/660797458)

- https://blog.csdn.net/weixin_54625990/article/details/130818488

  