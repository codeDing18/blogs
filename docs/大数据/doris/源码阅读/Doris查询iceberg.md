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

---



## 查询外表meta

```sql
switch iceberg;
use iceberg;
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
            //与catalog交互获取iceberg表
            Schema schema = ((IcebergExternalCatalog) catalog).getIcebergTable(dbName, name).schema();
            List<Types.NestedField> columns = schema.columns();
            List<Column> tmpSchema = Lists.newArrayListWithCapacity(columns.size());
            for (Types.NestedField field : columns) {
                //iceberg字段类型转换成Doris字段类型
                tmpSchema.add(new Column(field.name(),
                        icebergTypeToDorisType(field.type()), true, null, true, field.doc(), 													true,schema.caseInsensitiveFindField(field.name()).fieldId()));
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
          			//这一步来获取iceberg表
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

下图表示**catalog.loadTable**返回的iceberg表，表中包含了iceberg的元数据位置信息等

![image-20231206161445893](/Users/ding/Library/Application Support/typora-user-images/image-20231206161445893.png)

接下来看下BaseMetastoreCatalog.loadTable具体的代码实现

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
protected void refreshFromMetadataLocation(String newLocation, Predicate<Exception> shouldRetry, int numRetries) {
  			//下面的方法metadataLoader.apply最终会调用到这个lambda
  			//其主要是读取表的meta.json文件并对其进行解析形成TableMetadata
        this.refreshFromMetadataLocation(newLocation, shouldRetry, numRetries, (metadataLocation) -> {
            return TableMetadataParser.read(this.io(), metadataLocation);
        });
    }


//BaseMetastoreTableOperations
protected void refreshFromMetadataLocation(String newLocation, Predicate<Exception> shouldRetry, int numRetries, Function<String, TableMetadata> metadataLoader) {
        if (!Objects.equal(this.currentMetadataLocation, newLocation)) {
            //。。。
            Tasks.foreach(new String[]{newLocation}).....run((metadataLocation) -> {
              	//调用到上面方法的lambda，返回的TableMetadata添加在newMetadata中
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

iceberg-core中的BaseMetastoreCatalog的实现类如下图所示，我们的是hivecatalog

![image-20231207103911905](/Users/ding/Library/Application Support/typora-user-images/image-20231207103911905.png)

TableOperations ops = this.newTableOps(identifier);返回的HiveTableOperations

![image-20231207104729936](/Users/ding/Library/Application Support/typora-user-images/image-20231207104729936.png)



## 数据分片

```java
//org.apache.doris.planner.external.iceberg.IcebergScanNode
private List<Split> doGetSplits() throws UserException {
		//scan是org.apache.iceberg.DataTableScan（继承自BaseTableScan）
    TableScan scan = icebergTable.newScan();

    // set snapshot
    Long snapshotId = getSpecifiedSnapshot();
    if (snapshotId != null) {
        scan = scan.useSnapshot(snapshotId);
    }

    // set filter
    
    // Min split size is DEFAULT_SPLIT_SIZE(128MB).
    long splitSize = Math.max(ConnectContext.get().getSessionVariable().getFileSplitSize(), DEFAULT_SPLIT_SIZE);
   
  	//对大文件进行拆分，由scan.planFiles()生成FileScanTask，再根据splitSize进行split
    CloseableIterable<FileScanTask> fileScanTasks = TableScanUtil.splitFiles(scan.planFiles(), splitSize);
  
   //把多个小文件合并成在一个 CombinedScanTask 中
  try (CloseableIterable<CombinedScanTask> combinedScanTasks =
            TableScanUtil.planTasks(fileScanTasks, splitSize, 1, 0)) {
    		//在forEach执行the given action for each element of the Iterable
        combinedScanTasks.forEach(taskGrp -> taskGrp.files().forEach(splitTask -> {
            String dataFilePath = normalizeLocation(splitTask.file().path().toString());

            // Counts the number of partitions read
          
            IcebergSplit split = new IcebergSplit(
                   //。。。);
          
            splits.add(split);
        }));
    } catch (IOException e) {
        //。。。。
    }
    //。。。。
    return splits;
}
```

这里看下scan.planFiles()

```java
//org.apache.iceberg.BaseTableScan
public CloseableIterable<FileScanTask> planFiles() {
        Snapshot snapshot = this.snapshot();
        if (snapshot != null) {
           //。。。
            //主要由doPlanFiles进行
            return CloseableIterable.whenComplete(this.doPlanFiles(), () -> {
                //。。。  
            });
        } else {
           //。。。。
        }
    }

//org.apache.iceberg.DataTableScan
public CloseableIterable<FileScanTask> doPlanFiles() {
  			//确认当前快照
        Snapshot snapshot = this.snapshot();
        FileIO io = this.table().io();
  			//读取快照中的清单文件，根据清单文件的"content":0确定是data（0）还是delete（1）文件
        List<ManifestFile> dataManifests = snapshot.dataManifests(io);
        List<ManifestFile> deleteManifests = snapshot.deleteManifests(io);
        
        ManifestGroup manifestGroup = (new ManifestGroup(io, dataManifests, deleteManifests)).caseSensitive(this.isCaseSensitive()).select(this.scanColumns()).filterData(this.filter()).specsById(this.table().specs()).scanMetrics(this.scanMetrics()).ignoreDeleted();
       
				//构造出一个 ManifestGroup ,让 ManifestGroup 根据 manifest file 来获取输入的文件并生成FileScanTask
        return manifestGroup.planFiles();
    }

//org.apache.iceberg.ManifestGroup
public CloseableIterable<FileScanTask> planFiles() {
  			//最终iterator会调用createFileScanTasks生成FileScanTask
        return this.plan(ManifestGroup::createFileScanTasks);
    }


//org.apache.iceberg.ManifestGroup
public <T extends ScanTask> CloseableIterable<T> plan(ManifestGroup.CreateTasksFunction<T> createTasksFunc) {
        //处理逻辑在this.entries
  			//下面的方法最终调用到这里的lambda
        Iterable<CloseableIterable<T>> tasks = this.entries((manifest, entries) -> {
            int specId = manifest.partitionSpecId();
            ManifestGroup.TaskContext taskContext = (ManifestGroup.TaskContext)taskContextCache.get(specId);
            return createTasksFunc.apply(entries, taskContext);
        });
  			//返回一个ParallelIterable（线程池，会处理entries中对manifest处理的逻辑）
        return (CloseableIterable)(this.executorService != null ? new ParallelIterable(tasks, this.executorService) : CloseableIterable.concat(tasks));
    }


//org.apache.iceberg.ManifestGroup
private <T> Iterable<CloseableIterable<T>> entries(BiFunction<ManifestFile, CloseableIterable<ManifestEntry<DataFile>>, CloseableIterable<T>> entryFn) {
       //省略部分是根据表达式对ManifestFile的过滤
  			
  			//Iterables.transform会遍历每个matchingManifest，然后对每个manifest调用下面的lambda方法
        return Iterables.transform(matchingManifests, (manifest) -> {
            return new CloseableIterable<T>() {
                private CloseableIterable iterable;

                public CloseableIterator<T> iterator() {
                    //读取 ManifestFile
                    ManifestReader reader = ManifestFiles.read(manifest, 。。。);
                    
                    CloseableIterable entries;
                    if (ManifestGroup.this.ignoreDeleted) {
                        entries = reader.liveEntries();
                    } else {
                        entries = reader.entries();
                    }

                  	//文件级别裁剪
                    if (evaluator != null) {
                        entries = CloseableIterable.filter(ManifestGroup.this.scanMetrics.skippedDataFiles(), entries, (entry) -> {
                            return evaluator.eval((GenericDataFile)entry.file());
                        });
                    }

                 		//调用到上面方法传过来的lambda方法
                    this.iterable = (CloseableIterable)entryFn.apply(manifest, entries);
                    return this.iterable.iterator();
                }
               
            };
        });
    }


//org.apache.iceberg.ManifestGroup
//ManifestEntry就是manifest文件的内容
private static CloseableIterable<FileScanTask> createFileScanTasks(CloseableIterable<ManifestEntry<DataFile>> entries, ManifestGroup.TaskContext ctx) {
        return CloseableIterable.transform(entries, (entry) -> {
            //entry表示manifest文件的一行内容（比如manifest保存了三个data文件，就会有3行）
            DataFile dataFile = (DataFile)((DataFile)entry.file()).copy(ctx.shouldKeepStats());
            DeleteFile[] deleteFiles = ctx.deletes().forEntry(entry);
            DeleteFile[] var4 = deleteFiles;
            //。。。。
            //根据dataFile返回BaseFileScanTask
            return new BaseFileScanTask(dataFile, deleteFiles, ctx.schemaAsString(), ctx.specAsString(), ctx.residuals());
        });
    }
```

看下 combinedScanTasks =TableScanUtil.planTasks(fileScanTasks, splitSize, 1, 0)

> 这一步只是将小文件放入到bin包里（默认大小是8MB），见下图可知

```java
//org.apache.iceberg.util.TableScanUtil
public static CloseableIterable<CombinedScanTask> planTasks(CloseableIterable<FileScanTask> splitFiles, long splitSize, int lookback, long openFileCost) {
        //....
        return CloseableIterable.transform(CloseableIterable.combine(new PackingIterable(splitFiles, splitSize, lookback, weightFunc, true), splitFiles), BaseCombinedScanTask::new);
    }

//这里的小文件合并的核心是BinPacking.PackingIterator
public List<T> next() {
            while(true) {
                if (this.items.hasNext()) {
                    T item = this.items.next();
                    //每个小文件的大小
                    long weight = (Long)this.weightFunc.apply(item);
                  	//findBin会判断当前bin（大小默认8MB）能不能装的下当前的小文件
                    //如果可以返回bin，不可以返回null重新建个bin
                    BinPacking.Bin<T> bin = this.findBin(weight);
                    if (bin != null) {
                        bin.add(item, weight);
                        continue;
                    }
										//return new BinPacking.Bin(this.targetWeight);
                    //这里的targetWeight就是传入的splitSize
                    bin = this.newBin();
                  	//将当前的小文件放入当前的bin中
                    bin.add(item, weight);
                    this.bins.addLast(bin);
                    //。。
                }

                return ImmutableList.copyOf(((BinPacking.Bin)this.bins.removeFirst()).items());
            }
        }
```

![image-20231217152336778](/Users/ding/Library/Application Support/typora-user-images/image-20231217152336778.png)





# 参考

- [iceberg读取meta流程](https://zhuanlan.zhihu.com/p/648195366)
- [元数据管理与创建表](https://zhuanlan.zhihu.com/p/454151753)
- [Iceberg 小文件合并原理及实践](https://www.iteblog.com/archives/9896.html)