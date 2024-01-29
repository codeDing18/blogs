# 原地部署Doris集群

**注意事项**

- 部署Doris集群的时候，配置端口时先用`netstat -tunlp | grep 端口号`检查下Doris配置的端口是否被占用
- 配置BE的storage_root_path=/path/your/data_dir时，先用`df -h`查看磁盘使用情况，将Doris的存储路径设置在不同于starrocks存储的挂载盘上，并且确保该盘的磁盘空间足够
- Doris集群全部启动后，`SHOW BACKENDS\G`检查下所有节点情况，均正常后再进行数据同步。



# X2Doris工具

**注意事项**

- 根据[工具使用手册](https://justtmp-bj-1308700295.cos.ap-beijing.myqcloud.com/x2doris.pdf)进行配置时，在作业设置阶段，如果环境中有了spark集群，DeployMode设置为cluster。如果没有spark集群只是用工具包中的spark的话，DeployMode设置为client（spark的配置请根据自己环境进行设置）

- 选择整个库进行迁移，如果报下图所示的错误的话，则表示这个库中有视图，目前该工具只支持同步普通的表。

  > 解决方案：已向社区反馈，选择整库时，可以排除库中的视图来建表。将社区修改后的jar包放到lib下，重新启动服务即可。

  ![image-20231114143608595](/Users/ding/Library/Application Support/typora-user-images/image-20231114143608595.png)



- 在建表时，Starrocks表ddl语句中某些属性在Doris中不被识别，报如下图所示错误。手动将ddl中不被doris识别的PROPERTIES删除后，再选择其它表时，删除的还是会恢复。

> 解决方案：已向社区反应过，将上一个问题中的jar包替换在lib下重启服务即可

![image-20231114145922329](/Users/ding/Library/Application Support/typora-user-images/image-20231114145922329.png)

