# COW、MOR

---

## Copy-On-Write Table

对于 Copy-On-Write Table，用户的 update 会重写数据所在的文件，所以是一个写放大很高，但是读放大为 0，适合写少读多的场景。对于这种 Table，提供了两种查询：

- Snapshot Query: 查询最近一次 snapshot 的数据，也就是最新的数据。
- Incrementabl Query:用户需要指定一个 commit time，然后 Hudi 会扫描文件中的记录，过滤出 commit_time > 用户指定的 commit time 的记录。

## Merge-On-Read Table

对于 Merge-On-Read Table，整体的结构有点像 LSM-Tree，用户的写入先写入到 delta data 中，这部分数据使用行存，这部分 delta data 可以手动 merge 到存量文件中，整理为 parquet 的列存结构。对于这类 Tabel，提供了三种查询：

- Snapshot Query: 查询最近一次 snapshot 的数据，也就是最新的数据。这里是一个行列数据混合的查询。
- Incrementabl Query:用户需要指定一个 commit time，然后 Hudi 会扫描文件中的记录，过滤出 commit_time > 用户指定的 commit time 的记录。这里是一个行列数据混合的查询。
- Read Optimized Query: 只查存量数据，不查增量数据，因为使用的都是列式文件格式，所以效率较高。



# 参考资料

- [HUDI原理及深入探究](https://blog.csdn.net/Dreamershi/article/details/123501987)