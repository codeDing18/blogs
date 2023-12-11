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
                int waitTimeOut = ConnectContext.get() == null ? 300 : ConnectContext.get().getExecTimeout();
                MasterCatalogExecutor remoteExecutor = new MasterCatalogExecutor(waitTimeOut * 1000);
                try {
                    remoteExecutor.forward(id, -1);
                } catch (Exception e) {
                    Util.logAndThrowRuntimeException(LOG,
                            String.format("failed to forward init catalog %s operation to master.", name), e);
                }
                return;
            }
            //就是做了dbNameToId、dbNameToId初始化，用map保存了catalog下的数据库（需要请求hms）
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
                int waitTimeOut = ConnectContext.get() == null ? 300 : ConnectContext.get().getExecTimeout();
                MasterCatalogExecutor remoteExecutor = new MasterCatalogExecutor(waitTimeOut * 1000);
                try {
                    remoteExecutor.forward(extCatalog.getId(), id);
                } catch (Exception e) {
                    Util.logAndThrowRuntimeException(LOG,
                            String.format("failed to forward init external db %s operation to master", name), e);
                }
                return;
            }
            //总结下来就是对tableNameToId、idToTbl进行map初始化，保存所有表信息（需要请求hms）
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

对于第一次查询的外表，在analyze时，本地schemaCache现在并没有该表schema信息，所以需要请求hms获取

![image-20231206095136167](/Users/ding/Library/Application Support/typora-user-images/image-20231206095136167.png)

这里就是该表请求hms获得表的原始schema信息，并且在**下图的for循环字段中将每个字段类型转换成doris中的字段**

![image-20231206095005872](/Users/ding/Library/Application Support/typora-user-images/image-20231206095005872.png)

当获取到表的schema信息后，schemaCache就会进行本地保存，下次查询时就不会再请求hms

![image-20231206100333361](/Users/ding/Library/Application Support/typora-user-images/image-20231206100333361.png)







# iceberg

---

## 建立外表catalog

具体参考[官网](https://doris.apache.org/zh-CN/docs/lakehouse/multi-catalog/iceberg)

```sql
CREATE CATALOG iceberg PROPERTIES (
    'type'='iceberg',
    'iceberg.catalog.type'='hms',
    'hive.metastore.uris' = 'thrift://127.0.0.1:9083'
);

CREATE CATALOG iceberg PROPERTIES (
    'type'='iceberg',
    'iceberg.catalog.type'='hms',
    'hive.metastore.uris' = 'thrift://ip:9083',
    'hive.metastore.sasl.enabled' = 'true',
    'hive.metastore.kerberos.principal' = 'hive/_HOST@BIGDATA.CN',
    'dfs.nameservices'='ctyunns',
    'dfs.ha.namenodes.ctyunns'='nn1,nn2',
    'dfs.namenode.rpc-address.ctyunns.nn1'='IP:54310',
    'dfs.namenode.rpc-address.ctyunns.nn2'='ip:54310', 'dfs.client.failover.proxy.provider.ctyunns'='org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider',
    'hadoop.security.authentication' = 'kerberos',
    'hadoop.kerberos.keytab' = '/etc/security/keytabs/hdfs.keytab',   
    'hadoop.kerberos.principal' = 'hdfs/evm-ch9m33v1ccmlv0936f@BIGDATA.CN',
    'yarn.resourcemanager.principal' = 'yarn/evm-ch9m33v1ccmlv0936f@BIGDATA.CN'
);
```



## 查询外表

```sql
switch iceberg;
select * from testiceberg;
```

到下面这与hive一样，但是在Iceberg获取schema时进入的是自己的逻辑

![image-20231206165847848](/Users/ding/Library/Application Support/typora-user-images/image-20231206165847848.png)

根据调用的方法栈，在table.get().initSchemaAndUpdateTime()会根据是什么表类型进行各自的获取schema方法

![image-20231206170018850](/Users/ding/Library/Application Support/typora-user-images/image-20231206170018850.png)

看下IcebergExternalTable.initSchema()

```java
public List<Column> initSchema() {
  			//这里需要schema信息，但是得先有iceberg表，再返回schema
        return HiveMetaStoreClientHelper.ugiDoAs(catalog.getConfiguration(), () -> {
            Schema schema = ((IcebergExternalCatalog) catalog).getIcebergTable(dbName, name).schema();
            List<Types.NestedField> columns = schema.columns();
            List<Column> tmpSchema = Lists.newArrayListWithCapacity(columns.size());
            for (Types.NestedField field : columns) {
                tmpSchema.add(new Column(field.name(),
                        icebergTypeToDorisType(field.type()), true, null, true, field.doc(), true,
                        schema.caseInsensitiveFindField(field.name()).fieldId()));
            }
            return tmpSchema;
        });
    }

public org.apache.iceberg.Table getIcebergTable(String dbName, String tblName) {
        makeSureInitialized();
        return Env.getCurrentEnv()
                .getExtMetaCacheMgr()
          			//获取IcebergMetadataCache
                .getIcebergMetadataCache()
                .getIcebergTable(catalog, id, dbName, tblName);
    }

//IcebergMetadataCache.getIcebergTable
public Table getIcebergTable(Catalog catalog, long catalogId, String dbName, String tbName) {
        IcebergMetadataCacheKey key = IcebergMetadataCacheKey.of(
                catalogId,
                dbName,
                tbName);
  			//如果有缓存则直接返回
        Table cacheTable = tableCache.getIfPresent(key);
        if (cacheTable != null) {
            return cacheTable;
        }
  			//iceberg要获取元信息，先与catalog进行交互。通过catalog.loadTable获取到表所有信息，如下图所示
  			//这里的catalog.loadTable见下面分析
        Table table = HiveMetaStoreClientHelper.ugiDoAs(catalogId,
                () -> catalog.loadTable(TableIdentifier.of(dbName, tbName)));
        initIcebergTableFileIO(table);
				//缓存表的信息，下次就不需要再与hms通信
        tableCache.put(key, table);

        return table;
    }
```

> Catalog 通常用来保存和查找表的元数据，比如 schema,属性信息等
>
> Iceberg 为了支持多种 Catalog ,所以也定义了自己的 Catalog 规范，见下图
>
> HiveCatalog extends BaseMetastoreCatalog          BaseMetastoreCatalog implements Catalog

![image-20231207150424492](/Users/ding/Library/Application Support/typora-user-images/image-20231207150424492.png)

下图表示**catalog.loadTable**返回的iceberg表，表中包含了iceberg的元数据信息

![image-20231206161445893](/Users/ding/Library/Application Support/typora-user-images/image-20231206161445893.png)

上图中的代码是iceberg-core中的

```java
//iceberg-core中的BaseMetastoreCatalog（抽象类,我们建的是hivecatalog）
public Table loadTable(TableIdentifier identifier) {
        Object result;
        if (this.isValidIdentifier(identifier)) {
            //返回HiveTableOperations
            TableOperations ops = this.newTableOps(identifier);
          	//ops.current()返回TableMetadata
            if (ops.current() == null) {
                result = this.loadMetadataTable(identifier);
            } else {
                result = new BaseTable(ops, fullTableName(this.name(), identifier));
            }
        } else {
            result = this.loadMetadataTable(identifier);
        }

        LOG.info("Table loaded by catalog: {}", result);
        return (Table)result;
    }



//BaseMetastoreTableOperations
//shouldRefresh默认为true
public TableMetadata current() {
        return this.shouldRefresh ? this.refresh() : this.currentMetadata;
    }
//BaseMetastoreTableOperations
public TableMetadata refresh() {
        boolean currentMetadataWasAvailable = this.currentMetadata != null;
  
        try {
          	//在这个方法获取table metadata
            this.doRefresh();
        } catch (NoSuchTableException var3) {
            //。。。。
        }

        return this.current();
    }

//HiveTableOperations（继承BaseMetastoreTableOperations）
protected void doRefresh() {
        String metadataLocation = null;

        try {
            //return client.getTable最终是执行到doris中HiveMetaStoreClient.getTable
            //这里的table就是下图中返回的t
            Table table = (Table)this.metaClients.run((client) -> {
                return client.getTable(this.database, this.tableName);
            });
            validateTableIsIceberg(table, this.fullName);
            //从下图中的parameters可见metadata_location
            metadataLocation = (String)table.getParameters().get("metadata_location");
        } catch (NoSuchObjectException var4) {
            //...
        }
				//根据metadata_location来刷新metadata
        this.refreshFromMetadataLocation(metadataLocation, this.metadataRefreshMaxRetries);
    }

//BaseMetastoreTableOperations
protected void refreshFromMetadataLocation(String newLocation, Predicate<Exception> shouldRetry, int numRetries, Function<String, TableMetadata> metadataLoader) {
        if (!Objects.equal(this.currentMetadataLocation, newLocation)) {
            //。。。
            Tasks.foreach(new String[]{newLocation}).....run((metadataLocation) -> {
                newMetadata.set((TableMetadata)metadataLoader.apply(metadataLocation));
            });            
						//将获取的最新的metadata填充到HiveTableOperations中
            this.currentMetadata = (TableMetadata)newMetadata.get();
            this.currentMetadataLocation = newLocation;
            this.version = parseVersion(newLocation);
        }

        this.shouldRefresh = false;
    }

//HiveMetaStoreClient
public Table getTable(String dbname, String name) throws TException {
    return getTable(getDefaultCatalog(conf), dbname, name);
  }

//HiveMetaStoreClient   到这里就是具体的客户端请求hms，见下图返回的t
public Table getTable(String catName, String dbName, String tableName) throws TException {
    Table t;
    if (hiveVersion == HiveVersion.V1_0 || hiveVersion == HiveVersion.V2_0) {
      t = client.get_table(dbName, tableName);
    } else if (hiveVersion == HiveVersion.V2_3) {
      GetTableRequest req = new GetTableRequest(dbName, tableName);
      req.setCapabilities(version);
      t = client.get_table_req(req).getTable();
    } else {
      GetTableRequest req = new GetTableRequest(dbName, tableName);
      req.setCatName(catName);
      req.setCapabilities(version);
      t = client.get_table_req(req).getTable();
    }
    return deepCopy(filterHook.filterTable(t));
  }
```

返回的t

![image-20231207194220705](/Users/ding/Library/Application Support/typora-user-images/image-20231207194220705.png)

iceberg-core中的BaseMetastoreCatalog的实现类如下图所示，我们建的是hivecatalog

![image-20231207103911905](/Users/ding/Library/Application Support/typora-user-images/image-20231207103911905.png)

TableOperations ops = this.newTableOps(identifier);返回的HiveTableOperations

![image-20231207104729936](/Users/ding/Library/Application Support/typora-user-images/image-20231207104729936.png)













# 参考

- [iceberg读取meta流程](https://zhuanlan.zhihu.com/p/648195366)
- [元数据管理与创建表](https://zhuanlan.zhihu.com/p/454151753)