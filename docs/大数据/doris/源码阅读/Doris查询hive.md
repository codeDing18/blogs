# Hive

---

## 建立外表catalog

具体参考[doris数据湖](https://doris.apache.org/zh-CN/docs/lakehouse/multi-catalog/hive)

```sql
CREATE CATALOG hive PROPERTIES (
    'type'='hms',
    'hive.metastore.uris' = 'thrift://127.0.0.1:9083'
);


CREATE CATALOG hive PROPERTIES (
    'type'='hms',
    'hive.metastore.uris' = 'thrift://ip:9083',
    'hive.metastore.sasl.enabled' = 'true',
    'hive.metastore.kerberos.principal' = 'hive/_HOST@BIGDATA.CN',
    'dfs.nameservices'='yunns',
    'dfs.ha.namenodes.yunns'='nn1,nn2',
    'dfs.namenode.rpc-address.yunns.nn1'='ip:54310',
    'dfs.namenode.rpc-address.yunns.nn2'='ip:54310', 'dfs.client.failover.proxy.provider.yunns'='org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider',
    'hadoop.security.authentication' = 'kerberos',
    'hadoop.kerberos.keytab' = '/etc/security/keytabs/hdfs.keytab',   
    'hadoop.kerberos.principal' = 'hdfs/evm-ch9m33v1ccmlv0936f@BIGDATA.CN',
    'yarn.resourcemanager.principal' = 'yarn/evm-ch9m33v1ccmlv0936f@BIGDATA.CN'
);
```

## 查询外表

```bash
switch hive;
select * from test.test2;
```

当创建完hive catalog后，进行第一次查询时，在analyze阶段getTables时会出现下图的方法栈，最终会调用ExternalCatalog.makeSureInitialized()方法。在查询外表时，每次都会先进行初始化检查确保外表的catalog、database都进行了初始化。

![image-20231206103617817](/Users/ding/Library/Application Support/typora-user-images/image-20231206103617817.png)

![image-20231206105748646](/Users/ding/Library/Application Support/typora-user-images/image-20231206105748646.png)

接下来看下ExternalCatalog.makeSureInitialized()具体做了什么。

```java
//ExternalCatalog.makeSureInitialized()
public final synchronized void makeSureInitialized() {
  			//这个方法池化了访问hms的客户端
        //client = new PooledHiveMetaStoreClient(hiveConf,
        //Math.max(MIN_CLIENT_POOL_SIZE, Config.max_external_cache_loader_thread_pool_size));
        initLocalObjects();
        if (!initialized) {
            if (!Env.getCurrentEnv().isMaster()) {
                // Forward to master and wait the journal to replay.
               
            }
            //客户端请求hms,做了dbNameToId、dbNameToId初始化，用map保存了catalog下的数据库
            init();
            initialized = true;
        }
    }
```

看下ExternalDatabase.makeSureInitialized()方法

```java
public final synchronized void makeSureInitialized() {
    		//确保外部的catalog已经初始化
        extCatalog.makeSureInitialized();
        if (!initialized) {
            if (!Env.getCurrentEnv().isMaster()) {
                // Forward to master and wait the journal to replay.
                
            }
            //总结下来就是请求hms对tableNameToId、idToTbl进行map初始化，保存所有表信息
            init();
        }
    }
```

> 为什么每次都要进行初始化检查？
>
> 如下图所示，初始化都需要请求hms获取信息，用map结构保存数据后，下次可直接查询不用再请求hms

![image-20231206110242691](/Users/ding/Library/Application Support/typora-user-images/image-20231206110242691.png)

查询出来的库信息用map进行保存，如下图所示。

![image-20231205160318963](/Users/ding/Library/Application Support/typora-user-images/image-20231205160318963.png)

接下来在analyze阶段会走到下图所示，外部表第一次会请求hms给出表的信息，将该信息给自己表的属性中并判断该表是什么类型的表

![image-20231205171718319](/Users/ding/Library/Application Support/typora-user-images/image-20231205171718319.png)

```java
protected synchronized void makeSureInitialized() {
        super.makeSureInitialized();
        if (!objectCreated) {
            remoteTable = ((HMSExternalCatalog) catalog).getClient().getTable(dbName, name);
            if (remoteTable == null) {
                dlaType = DLAType.UNKNOWN;
            } else {
                if (supportedIcebergTable()) {
                    dlaType = DLAType.ICEBERG;
                } else if (supportedHoodieTable()) {
                    dlaType = DLAType.HUDI;
                } else if (supportedHiveTable()) {
                    dlaType = DLAType.HIVE;
                } else {
                    dlaType = DLAType.UNKNOWN;
                }
            }
            objectCreated = true;
        }
    }
```

> 疑问：在获取到remoteTable时已经知道cols信息，为什么这里不做schema缓存，而是要在下面图片重新请求hms获取schema信息?
>
> 这个remoteTable是org.apache.hadoop.hive.metastore.api.Table，只保存了表的基本信息（比如表名、数据库名），但如果是iceberg或者hudi，其表的元数据信息有自己的实现，在这里没法获取到。并且这里的schema信息还需要转换成Doris中的格式，所以源码将这一步放在下下张图片的initSchema中。此外还有IcebergExternalTable，都有自己的实现

对于第一次查询的外表，在analyze时，本地schemaCache现在并没有该表schema信息，所以需要请求hms获取

![image-20231206095136167](/Users/ding/Library/Application Support/typora-user-images/image-20231206095136167.png)

这里就是该表请求hms获得表的原始schema信息，并且在**下图的for循环字段中将每个字段类型转换成doris中的字段。**这里是hive表schema处理办法，如果是iceberg、hudi还要走它们各自的逻辑

![image-20231206095005872](/Users/ding/Library/Application Support/typora-user-images/image-20231206095005872.png)

> 在这一步initSchema时有个问题，为什么不先判断是iceberg类型还是hudi类型，然后再去请求对应的实现。这里先获取了schema信息已经请求了一次hms，后面再判断类型时又要请求一次hms，不是很浪费时间吗。传进去schema参数只是在new Column起作用，基本没啥作用

当获取到表的schema信息后，schemaCache就会进行本地保存，下次查询时就不会再请求hms

![image-20231206100333361](/Users/ding/Library/Application Support/typora-user-images/image-20231206100333361.png)



## 数据分片

hive表的数据split主要在HiveScanNode.getSplits

```java
//org.apache.doris.planner.external.HiveScanNode
protected List<Split> getSplits() throws UserException {
        long start = System.currentTimeMillis();
        try {
            HiveMetaStoreCache cache = Env.getCurrentEnv().getExtMetaCacheMgr()
                    .getMetaStoreCache((HMSExternalCatalog) hmsTable.getCatalog());
            boolean useSelfSplitter = hmsTable.getCatalog().useSelfSplitter();
            List<Split> allFiles = Lists.newArrayList();
          	//主要看下这个办法
            getFileSplitByPartitions(cache, getPartitions(), allFiles, useSelfSplitter);
            //..
            return allFiles;
        } catch (Throwable t) {
          //..
        }
    }


private void getFileSplitByPartitions(HiveMetaStoreCache cache, List<HivePartition> partitions,List<Split> allFiles, boolean useSelfSplitter) throws IOException {
        List<FileCacheValue> fileCaches;
        
        for (HiveMetaStoreCache.FileCacheValue fileCacheValue : fileCaches) {
            // This if branch is to support old splitter, will remove later.
            if (fileCacheValue.getSplits() != null) {
                allFiles.addAll(fileCacheValue.getSplits());
            }
            if (fileCacheValue.getFiles() != null) {
                boolean isSplittable = fileCacheValue.isSplittable();
                for (HiveMetaStoreCache.HiveFileStatus status : fileCacheValue.getFiles()) {
                  //真正的split在  splitFile方法中
                  allFiles.addAll(splitFile(status.getPath(), status.getBlockSize(),
                            status.getBlockLocations(), status.getLength(), status.getModificationTime(),isSplittable, fileCacheValue.getPartitionValues(),
                            new HiveSplitCreator(fileCacheValue.getAcidInfo())));
                }
            }
        }
    }


//org.apache.doris.planner.external.FileScanNode
protected List<Split> splitFile(Path path, long blockSize, BlockLocation[] blockLocations, long length, long modificationTime, boolean splittable, List<String> partitionValues, SplitCreator splitCreator) throws IOException {
        //..省略
        
        // Min split size is DEFAULT_SPLIT_SIZE(128MB).
        splitSize = Math.max(splitSize, DEFAULT_SPLIT_SIZE);
        long bytesRemaining;
  			//hive的split逻辑
        for (bytesRemaining = length; (double) bytesRemaining / (double) splitSize > 1.1D;
                bytesRemaining -= splitSize) {
            int location = getBlockIndex(blockLocations, length - bytesRemaining);
            String[] hosts = location == -1 ? null : blockLocations[location].getHosts();
            result.add(splitCreator.create(path, length - bytesRemaining, splitSize,
                    length, modificationTime, hosts, partitionValues));
        }
        if (bytesRemaining != 0L) {
            int location = getBlockIndex(blockLocations, length - bytesRemaining);
            String[] hosts = location == -1 ? null : blockLocations[location].getHosts();
            result.add(splitCreator.create(path, length - bytesRemaining, bytesRemaining,
                    length, modificationTime, hosts, partitionValues));
        }

        return result;
    }
```





