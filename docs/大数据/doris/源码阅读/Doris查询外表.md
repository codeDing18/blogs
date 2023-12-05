# 建立外表catalog

具体参考[doris数据湖](https://doris.apache.org/zh-CN/docs/lakehouse/multi-catalog/hive)

```sql
CREATE CATALOG hive PROPERTIES (
    'type'='hms',
    'hive.metastore.uris' = 'thrift://127.0.0.1:9083'
);
```











# 查询外表

```bash
switch hive;
select * from test.test1;
```

当创建完hive catalog后，进行第一次查询时，Doris会对进行一次hms客户端请求，**查询外表的相关元数据信息然后对其进行缓存。**下图展示的是查询hive catalog中所有的库信息

![image-20231205155850146](/Users/ding/Library/Application Support/typora-user-images/image-20231205155850146.png)

查询出来的库信息用map进行保存，如下图所示。

![image-20231205160318963](/Users/ding/Library/Application Support/typora-user-images/image-20231205160318963.png)

第一次查询完成后initialized变为true，则下次查询就不会再请求hms获取相关元数据信息，如下图所示。

![image-20231205161243817](/Users/ding/Library/Application Support/typora-user-images/image-20231205161243817.png)

在进行catalog中的库元数据查询后，接下来会根据sql需要用的table来请求hms，如下图所示

![image-20231205161916910](/Users/ding/Library/Application Support/typora-user-images/image-20231205161916910.png)

Hms客户端请求hms查询出该库中所有的表

![image-20231205162855005](/Users/ding/Library/Application Support/typora-user-images/image-20231205162855005.png)







![image-20231205162327359](/Users/ding/Library/Application Support/typora-user-images/image-20231205162327359.png)





![image-20231205171718319](/Users/ding/Library/Application Support/typora-user-images/image-20231205171718319.png)





![image-20231205183343223](/Users/ding/Library/Application Support/typora-user-images/image-20231205183343223.png)