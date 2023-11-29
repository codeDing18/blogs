# tpch

---

## tpch数据导入到hive表中

```bash
create database tpch;
use tpch;
 
create table lineitem (
  l_orderkey int,
  l_partkey int,
  l_suppkey int,
  l_linenumber int,
  l_quantity double,
  l_extendedprice double,
  l_discount double,
  l_tax double,
  l_returnflag string,
  l_linestatus string,
  l_shipdate string,
  l_commitdate string,
  l_receiptdate string,
  l_shipinstruct string,
  l_shipmode string,
  l_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create external table nation (
  n_nationkey int,
  n_name string,
  n_regionkey int,
  n_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create external table region (
  r_regionkey int,
  r_name string,
  r_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create external table part (
  p_partkey int,
  p_name string,
  p_mfgr string,
  p_brand string,
  p_type string,
  p_size int,
  p_container string,
  p_retailprice double,
  p_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create external table supplier (
  s_suppkey int,
  s_name string,
  s_address string,
  s_nationkey int,
  s_phone string,
  s_acctbal double,
  s_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create external table partsupp (
  ps_partkey int,
  ps_suppkey int,
  ps_availqty int,
  ps_supplycost double,
  ps_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create table customer (
  c_custkey int,
  c_name string,
  c_address string,
  c_nationkey int,
  c_phone string,
  c_acctbal double,
  c_mktsegment string,
  c_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
create external table orders (
  o_orderkey int,
  o_custkey int,
  o_orderstatus string,
  o_totalprice double,
  o_orderdate date,
  o_orderpriority string,
  o_clerk string,
  o_shippriority int,
  o_comment string)
row format delimited
fields terminated by '|'
stored as textfile;
 
use tpch;
load data local inpath "/tpch/supplier.tbl" into table supplier;
load data local inpath "/tpch/region.tbl" into table region;
load data local inpath "/tpch/partsupp.tbl" into table partsupp;
load data local inpath "/tpch/part.tbl" into table part;
load data local inpath "/tpch/orders.tbl" into table orders;
load data local inpath "/tpch/nation.tbl" into table nation;
load data local inpath "/tpch/lineitem.tbl" into table lineitem;
load data local inpath "/tpch/customer.tbl" into table customer;
```



