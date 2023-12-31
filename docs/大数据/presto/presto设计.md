纵观 Presto 的实现：

**从数据源连接器实现的角度看，**Presto 为了方便管理各种数据源的连接，对数据源连接器进行了统一的抽象，定义了连接器实现的一系列标准：

- 连接器如何创建；
- 事务处理方式；
- 库表等元数据信息的获取；
- 任务划分；
- 节点选择；
- 数据获取；
- 资源评估。

每个数据源的连接器都需要按照此标准实现，运行时通过 Java SPI 服务发现提供机制加载预先配置的连接器，这样 Presto 借助于各种连接器，就有了整合数据的能力。

**从集群启动的角度看**，每个节点首先加载本地配置标识自己的身份信息，向 coordinator 发送 http 请求在 header 中附带标识信息，经过一定时间 coordinator 会收集到所有节点发送的标识信息，进行汇总后，同步一份给所有节点，这样每个节点都了解了彼此，集群才真正意义上建立起来。

而对于上面的实现，Presto 不是从 0 开始，**相反借助了很多开源项目：**

- 像开箱即用快速部署 RESTful 服务的 airlift；
- 依赖注入框架 Guice；
- 字节码工具 asm 和 net.bytebuddy；
- 快速生成 JMX MBean 的 org.weakref.jmxutils；
- 缓存 guava；
- http 客户端 okhttp；
- Java 集合扩展的 fastutil。

如果我们能够理解这些开源项目原理作用，Presto 又是如何使用，可以加深我们对 Presto 实现的理解，同时也可以将有价值的开源项目应用到实际项目中。

接下来我们一起从原理、具体用法和 Presto 集成的角度，分别对上面主要的开源项目进行分析。

> [参考链接](https://xie.infoq.cn/article/2e43d26cb9c75c26816901b78)

