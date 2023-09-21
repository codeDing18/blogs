# 创建表

> hoodie.embed.timeline.server=false 
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
select 1 as id, 'a1' as name, 1000 as ts, '2021-12-09' as dt, '10' as hh;

update hudi_mor_tbl set price = price * 2, ts = 1111 where id = 1;
```

>preCombineField表示预合并字段，就是当我们插入一条记录，这记录除了preCombineField字段不相同外，其它都相同，但是如果当前preCombineField字段值是111，要插入的preCombineField字段值是99，插入后的preCombineField字段还是111（取最大值设置）。



# 问题

```bash
java.lang.NoSuchMethodError: org.apache.hadoop.hdfs.client.HdfsDataInputStream.getReadStatistics()Lorg/apache/hadoop/hdfs/DFSInputStream$ReadStatistics;
```

> 解决：
>
> [方法找不到](https://juejin.cn/post/7114541251784867853)

