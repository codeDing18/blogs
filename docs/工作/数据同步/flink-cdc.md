

# 准备工作

## 环境准备

mysql5.7.10（开启binlog）

Doris2.0.2

flink1.14

下载如下两个jar包放到flink的lib目录下

![image-20231127100845944](/Users/ding/Library/Application Support/typora-user-images/image-20231127100845944.png)



## 建表

mysql

```sql
CREATE TABLE employees_1 (
    emp_no      INT             NOT NULL,
    birth_date  DATE            NOT NULL,
    first_name  VARCHAR(14)     NOT NULL,
    last_name   VARCHAR(16)     NOT NULL,
    gender      ENUM ('M','F')  NOT NULL,    
    hire_date   DATE            NOT NULL,
    PRIMARY KEY (emp_no)
);

INSERT INTO `employees_1` VALUES (10001,'1953-09-02','Georgi','Facello','M','1986-06-26'),
(10002,'1964-06-02','Bezalel','Simmel','F','1985-11-21'),
(10003,'1959-12-03','Parto','Bamford','M','1986-08-28'),
(10004,'1954-05-01','Chirstian','Koblick','M','1986-12-01'),
(10005,'1955-01-21','Kyoichi','Maliniak','M','1989-09-12'),
(10006,'1953-04-20','Anneke','Preusig','F','1989-06-02'),
(10007,'1957-05-23','Tzvetan','Zielinski','F','1989-02-10'),
(10008,'1958-02-19','Saniya','Kalloufi','M','1994-09-15'),
(10009,'1952-04-19','Sumant','Peac','F','1985-02-18'),
(10010,'1963-06-01','Duangkaew','Piveteau','F','1989-08-24');

CREATE TABLE employees_2 (
    emp_no      INT             NOT NULL,
    birth_date  DATE            NOT NULL,
    first_name  VARCHAR(14)     NOT NULL,
    last_name   VARCHAR(16)     NOT NULL,
    gender      ENUM ('M','F')  NOT NULL,    
    hire_date   DATE            NOT NULL,
    PRIMARY KEY (emp_no)
);

INSERT INTO `employees_2` VALUES (10037,'1963-07-22','Pradeep','Makrucki','M','1990-12-05'),
(10038,'1960-07-20','Huan','Lortz','M','1989-09-20'),
(10039,'1959-10-01','Alejandro','Brender','M','1988-01-19'),
(10040,'1959-09-13','Weiyi','Meriste','F','1993-02-14'),
(10041,'1959-08-27','Uri','Lenart','F','1989-11-12'),
(10042,'1956-02-26','Magy','Stamatiou','F','1993-03-21'),
(10043,'1960-09-19','Yishay','Tzvieli','M','1990-10-20'),
(10044,'1961-09-21','Mingsen','Casley','F','1994-05-21'),
(10045,'1957-08-14','Moss','Shanbhogue','M','1989-09-02'),
(10046,'1960-07-23','Lucien','Rosenbaum','M','1992-06-20');
```

Doris

```sql
CREATE TABLE all_employees_info (
    emp_no       int NOT NULL,
    birth_date   date,
    first_name   varchar(20),
    last_name    varchar(20),
    gender       char(2),
    hire_date    date,
    database_name varchar(50),
    table_name    varchar(200)
)
UNIQUE KEY(`emp_no`, `birth_date`)
DISTRIBUTED BY HASH(`birth_date`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
```

Flink  sql

```sql
#先创建MySQL CDC 表
CREATE TABLE employees_source (
    database_name STRING METADATA VIRTUAL,
    table_name STRING METADATA VIRTUAL,
    emp_no int NOT NULL,
    birth_date date,
    first_name STRING,
    last_name STRING,
    gender STRING,
    hire_date date,
    PRIMARY KEY (`emp_no`) NOT ENFORCED
  ) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'localhost',
    'port' = '3306',
    'username' = 'root',
    'password' = 'MyNewPass4!',
    'database-name' = 'emp_[0-9]+',
    'table-name' = 'employees_[0-9]+'
  );
  
# 创建 Doris Sink 表
CREATE TABLE cdc_doris_sink (
    emp_no       int ,
    birth_date   STRING,
    first_name   STRING,
    last_name    STRING,
    gender       STRING,
    hire_date    STRING,
    database_name STRING,
    table_name    STRING
) 
WITH (
  'connector' = 'doris',
  'fenodes' = 'IP:8030',
  'table.identifier' = 'demo.all_employees_info',
  'username' = 'root',
  'password' = '',
  'sink.properties.two_phase_commit'='true',
  'sink.label-prefix'='doris_demo_emp_001'
);


#将数据同步到Doris里面
insert into cdc_doris_sink (emp_no,birth_date,first_name,last_name,gender,hire_date,database_name,table_name) 
select emp_no,cast(birth_date as string) as birth_date ,first_name,last_name,gender,cast(hire_date as string) as hire_date ,database_name,table_name from employees_source;
```

在flink web ui界面可以看到作业信息

![image-20231127092212459](/Users/ding/Library/Application Support/typora-user-images/image-20231127092212459.png)



# 功能测试结果

当上述作业启动后，mysql数据库中的数据已经同步到doris中，doris查询如下：

![image-20231127092556083](/Users/ding/Library/Application Support/typora-user-images/image-20231127092556083.png)

## 

当在mysql中进行update操作时

![image-20231127093521243](/Users/ding/Library/Application Support/typora-user-images/image-20231127093521243.png)

Doris也可同步更新，结果如下：

![image-20231127093600965](/Users/ding/Library/Application Support/typora-user-images/image-20231127093600965.png)



在mysql中执行insert操作，mysql结果如下：

![image-20231127093813014](/Users/ding/Library/Application Support/typora-user-images/image-20231127093813014.png)

Doris同步结果如下：

![image-20231127093902982](/Users/ding/Library/Application Support/typora-user-images/image-20231127093902982.png)



mysql执行delete操作，结果如下：

![image-20231127094044688](/Users/ding/Library/Application Support/typora-user-images/image-20231127094044688.png)

doris结果如下

![image-20231127094126793](/Users/ding/Library/Application Support/typora-user-images/image-20231127094126793.png)





# 原理

MySQL CDC Source 的启动模式默认是先全量读取再增量读取。全量读取是通过表的主键将表划分成多个块（chunk）， 然后 MySQL CDC Source 将多个块分配给多个 reader 以并行读取表的数据，所以可以在flinksql client中设置读取的并行度`SET 'parallelism.default' = 8;`（当数据行数达到亿行时可设置到8，**如果mysql数据量很大，Doris写入压力会很大，需要调大Doris的be中streaming_load_max_mb大小，其默认10GB，但我们安装的时候调到了40GB基本够用**），在增量读取阶段只有一个reader进行读取binlog。





# 注意事项

- mysql要开启binlog功能
- 在flink sql中一定要开启`SET 'execution.checkpointing.interval' = '10s’;`（生产环境可以设置为5-10分钟）
- 如果想增加列或者删除列只能自己写个flink程序，flink sql目前还不支持

- **注意这里如果要想Mysql表里的数据修改，Doris里的数据也同样修改，Doris数据表的模型要是Unique key模型，其他数据模型（Aggregate Key 和 Duplicate Key）不能进行数据的更新操作。**







# flink代码

```java
public static void main(String[] args) throws Exception {


    //1.创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    //2.flinkcdc 做断点续传,需要将flinkcdc读取binlog的位置信息以状态方式保存在checkpoint中即可.

    //(1)开启checkpoint 每隔5s 执行一次ck 指定ck的一致性语义

   env.enableCheckpointing(5000L);
 env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

    //3.设置任务关闭后,保存最后后一次ck数据.
   env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

    env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3,2000L));

   env.setStateBackend(new FsStateBackend("hdfs://192.168.1.204:9000/flinkCDC"));
    //4.设置访问HDFS的用户名
    System.setProperty("HADOOP_USER_NAME","root");

    //5.创建Sources数据源
    Properties prop = new Properties();
    prop.setProperty("scan.startup.mode","initial");  //"scan.startup.mode","initial" 三种要补充解释下

    DebeziumSourceFunction<String> mysqlSource = MySQLSource.<String>builder()
            .hostname("192.168.1.205")
            .port(3306)
            .username("root")
            .password("Root@123")
            .tableList("ssm.order") //这里的表名称,书写格式:db.table  不然会报错
            .databaseList("ssm")
            .debeziumProperties(prop)
            .deserializer(new StringDebeziumDeserializationSchema())
            .build();
    
    //6.添加数据源
    DataStreamSource<String> source = env.addSource(mysqlSource);
     //7.打印数据
    source.print();

    //8.执行任务

    env.execute();

}
```







# 参考

- [mysql-cdc flink doris](https://developer.aliyun.com/article/1005812)
- https://zhuanlan.zhihu.com/p/425938765
- [finkcdc程序](https://blog.csdn.net/zhang5324496/article/details/120212622?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-120212622-blog-121071915.235%5Ev38%5Epc_relevant_sort_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-120212622-blog-121071915.235%5Ev38%5Epc_relevant_sort_base2&utm_relevant_index=2)