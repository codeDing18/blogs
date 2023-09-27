# 准备

---

- 将hudi-spark3.2-bundle_2.12-1.0.0-SNAPSHOT.jar与spark-avro_2.13-3.3.3.jar放到spark目录下的jars目录中
- [排除低版本jetty](https://blog.csdn.net/u010520724/article/details/128126213?ops_request_misc=&request_id=&biz_id=102&utm_term=%E5%AE%89%E8%A3%85hudi&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-4-128126213.142^v94^insert_down1&spm=1018.2226.3001.4187)

```bash
#自建个spark-hudi启动脚本
spark-sql  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'  --driver-java-options -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```



# 使用

---

> hoodie.embed.timeline.server=false，用于创建分区表时
>
> 这个设置是因为这是因为我构建hudi 的时候，使用hadoop3编译的，
> Hadoop3提供的是Jetty 9.3; Hudi 依赖于 Jetty 9.4 ( SessionHandler.setHttpOnly() 在9.3版本中并不存在).
> 使用hudi 0.9.0版本内置的hadoop 2.7.3编译则不存在这个问题。

```sql
create table hudi_mor (
  id bigint,
  name string,
  ts bigint,
  dt string,
  hh string
) using hudi
options (
  type = 'mor',
  primaryKey = 'id',
  preCombineField = 'ts',
  hoodie.embed.timeline.server=false
 )
partitioned by (dt, hh);


insert into hudi_mor partition (dt, hh)
values(1,'a1',1000,'2021-12-09','10');

insert into hudi_mor partition (dt, hh)
select 1 as id, 'a1' as name, 1000 as ts, '2021-12-09' as dt, '10' as hh;

update hudi_mor_tbl set price = price * 2, ts = 1111 where id = 1;
```

>preCombineField表示预合并字段，就是当我们插入一条记录，这记录除了preCombineField字段不相同外，其它都相同，但是如果当前preCombineField字段值是111，要插入的preCombineField字段值是99，插入后的preCombineField字段还是111（取最大值设置）。



