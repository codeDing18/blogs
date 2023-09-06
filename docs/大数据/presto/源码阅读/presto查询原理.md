# presto查询流程的原理

---



> 注：本文参考知乎这篇文章，https://zhuanlan.zhihu.com/p/293775390
>
> 本文的trino版本是423

## 1.从架构上看SQL Query的执行流程

![Snipaste_2023-08-21_16-55-53.png](../../img/Snipaste_2023-08-21_16-55-53.png)

请参见上面的架构图，从用户开始写SQL开始到查询结果返回，我们划分出以下几个部分：

- SQL Client：用户可以在这里输入SQL，它负责提交SQL Query给Presto集群。SQL Client一般用Presto自带的Presto Client比较多，它可以处理分批返回的结果，并在终端展示给用户。
- External Storage System：由于Presto自身不存储数据，计算涉及到的数据及元数据都来自于外部存储系统，如HDFS，AWS S3等分布式系统。在企业实践经验中，经常使用HiveMetaStore来存存储元数据，使用HDFS来存储数据，通过Presto执行计算的方式来加快Hive表查询速度。
- Presto Coordinator：负责接收SQL Query，生成执行计划，拆分Stage和Task，调度分布式执行的任务到Presto Worker上。
- Presto Worker：负责执行收到的HttpRemoteTask，根据执行计划确定好都有哪些Operator以及它们的执行顺序，之后通过TaskExecutor和Driver完成所有Operator的计算。如果第一个要执行的Operator是SourceOperator，当前Task会先从External Storage System中拉取数据再进行后续的计算。如果最后一个执行的Operator是TaskOutputOperator，当前Task会将计算结果输出到OutputBuffer，等待依赖当前Stage的Stage来拉取结算结果。整个Query的所有Stage中的所有Task执行完后，将最终结果返回给SQL Client。

> 至此，有几点疑问：
>
> - 怎么生成AST的
> - 逻辑计划与执行计划是怎么生成的
> - stage是怎么划分的
> - presto生成task后是怎么调度到worker中的



## 2. 从代码中看SQL执行流程

先介绍两个概念：

- SQL是声明式的（declarative）：声明式指的是你描述的就是你想要的结果而不是计算的过程，如数据工程师用SQL完成数据计算时既是如此。与之相反的是过程式的，如你写一段Java代码，通篇都是在描述如何完成计算的过程，而只有到了结尾才return你想要的结果。声明式可以说是结果导向的，它不关心实现过程，所以普遍来说，写SQL比写Java代码更简单易懂，更容易上手，甚至可以面向没有编程经验的数据分析师。
- 执行计划（Execution Plan）：但是SQL背后的实现过程，总得有代码去实现吧，而且在很多情况下还要兼顾功能、性能、成本等因素，可以说是非常复杂的，这就是SQL的执行引擎需要去考虑和实现的。既然SQL是声明式的，不关心实现过程，那么SQL执行引擎如何才能知道具体的执行步骤和细节呢？这里就需要引入一个执行计划的概念，它描述的是SQL执行的详细步骤和细节，SQL执行引擎只要按照执行计划执行即可完成整个计算过程。当然在这之前，SQL执行引擎需要做的第一件事是先解析SQL，生成SQL对应的执行计划。

以上两个概念，套用到任意类SQL的执行系统都是适用的，例如MySQL，Hive，SparkSQL，Clickhouse。具体而言，对于Presto来说，它的SQL执行过程可以详细拆解为以下几个步骤：

- 第一步：【Coordinator】接收SQL Query请求
- 第二步：【Coordinator】词法与语法分析（生成AST）
- 第三步：【Coordinator】创建和启动QueryExecution
- 第四步：【Coordinator】语义分析(Analysis)、生成执行计划LogicalPlan
- 第五步：【Coordinator】优化执行计划，生成Optimized Logical Plan
- 第六步：【Coordinator】为逻辑执行计划分段(PlanFragment)
- 第七步：【Coordinator】创建SqlStageExecution（创建Stage）
- 第八步：【Coordinator】Stage调度-生成HttpRemoteTask并分发到Presto Worker
- 第九步：【Worker】在Presto Worker上执行任务，生成Query结果
- 第十步：【Coordinator】分批返回Query计算结果给SQL客户端

从第一步到第八步，主要描述的是Presto对一个传入的SQL语句如何进行解析并生成最终的执行计划，生成Query执行计划的主要流程如下图所示：

![11](../../img/Snipaste_2023-08-24_09-35-47.png)

可以看到，自第一步至第九步，全部是在Presto Coodinator上完成，足见Coordinator的核心地位，阅读Presto源码时你也会发现它的代码是极其复杂的，我前前后后阅读了十几遍，debug了几十遍才有了今天的这份自信来将我的经验输出到互联网上帮助你更高效的掌握它；第十步是在Presto Worker上执行；第十一步是在Presto Coordinator上执行，并将查询结果分批返回给客户端(如Presto SQL Client或者其他JDBC客户端)。

接下来几个章节我们也会逐一详细介绍以上的每个步骤，这里先对每一步做一些概要介绍。



## 3. SQL执行分步拆解

> **为了能够让内容更具体生动，我们选了一个典型的SQL，TPC-DS的Query55，以它为例来介绍SQL执行过程。**

假设现在有一个Presto用户，在Presto-Cli里面输入SQL，然后提交执行：

```sql
use tpcds.sf1;

-- For a given year, month and store manager calculate the total store sales of any combination all brands.
select i_brand_id as brand_id, i_brand as brand,
        sum(ss_ext_sales_price) as ext_price
from date_dim, store_sales, item
where d_date_sk = ss_sold_date_sk
        and ss_item_sk = i_item_sk
        and i_manager_id=82
        and d_moy=8
        and d_year=1999
group by i_brand, i_brand_id
order by ext_price desc, i_brand_id
limit 10;
```

简介：在TPC-DS的数据模型中，store_sales是事实表，date_dim与item是维度表。上面SQL的业务含义是对商店销售数据的分品牌统计。

### **3.1 第一步：【Coordinator】接收SQL Query请求**

Presto-Cli提交的Query，会以HTTP POST的方式请求到Presto集群的Coordinator。请求体中携带了SQL以及其他信息。

```bash
## Request Method/URI:
POST /v1/statement

## Headers：
X-Presto-Catalog = tpcds
X-Presto-Schema = sf1

## Request Body:

select i_brand_id as brand_id, i_brand as brand,
        sum(ss_ext_sales_price) as ext_price
from date_dim, store_sales, item
where d_date_sk = ss_sold_date_sk
        and ss_item_sk = i_item_sk
        and i_manager_id=82
        and d_moy=8
        and d_year=1999
group by i_brand, i_brand_id
order by ext_price desc, i_brand_id
limit 10;
```



Presto-Cli发起第一次HTTP请求，在Coordinator上，QueuedStatementResource接收Presto-Cli发来的HTTP Query请求后，生成QueryId，将Query加入待执行队列，返回Response告知Presto-Cli。

```java
		//@Path("/v1/statement")
		//QueuedStatementResource
		@ResourceSecurity(AUTHENTICATED_USER)
    @POST
    @Produces(APPLICATION_JSON)
    public Response postStatement(
            String statement,
            @Context HttpServletRequest servletRequest,
            @Context HttpHeaders httpHeaders,
            @Context UriInfo uriInfo)
    {
        if (isNullOrEmpty(statement)) {
            throw badRequest(BAD_REQUEST, "SQL statement is empty");
        }
				//registerQuery中生成的queryID，并且将Query加入待执行队列
        Query query = registerQuery(statement, servletRequest, httpHeaders);
				//返回的Response中包含了nextURI，所以第二次请求直接get这个nextURI，见下图所示
        return createQueryResultsResponse(query.getQueryResults(query.getLastToken(), uriInfo));
    }


//QueuedStatementResource::registerQuery()
private Query registerQuery(String statement, HttpServletRequest servletRequest, HttpHeaders httpHeaders)
    {
        // ...
  			//生成的query中包含了queryID
        Query query = new Query(statement, sessionContext, dispatchManager, queryInfoUrlFactory, tracer);
  			//将query加入待执行队列
        queryManager.registerQuery(query);
				
  			// ...
        return query;
    }
```

![1](../../img/Snipaste_2023-08-24_11-04-39.png)



紧接着Presto-Cli发起第二次HTTP请求，Presto处理此请求时，会真正把Query提交到DispatchManager::createQueryInternal()执行

```java
		//QueuedStatementResource
		@ResourceSecurity(PUBLIC)
    @GET
    @Path("queued/{queryId}/{slug}/{token}")
    @Produces(APPLICATION_JSON)
    public void getStatus(
            @PathParam("queryId") QueryId queryId,
            @PathParam("slug") String slug,
            @PathParam("token") long token,
            @QueryParam("maxWait") Duration maxWait,
            @Context UriInfo uriInfo,
            @Suspended AsyncResponse asyncResponse)
    {
        Query query = getQuery(queryId, slug, token);
				//在getStatus中最终会调用到DispatchManager::createQueryInternal()执行
        ListenableFuture<Response> future = getStatus(query, token, maxWait, uriInfo);
        bindAsyncResponse(asyncResponse, future, responseExecutor);
    }
```

> 备注：Slug与Token只是Presto用来验证接收到的请求是否是合法，slug在生成后就不会变，token在生成nextUrl的逻辑中，每次会token+1，这些不是我们关注的重点，可以忽略。

**DispatchManager::createQueryInternal()的执行流程是：**

```java
private <C> void createQueryInternal(QueryId queryId, Span querySpan, Slug slug, SessionContext sessionContext, String query, ResourceGroupManager<C> resourceGroupManager)
    {
        		//...

            // 第一步：获取Query对应的Session
            session = sessionSupplier.createSession(queryId, querySpan, sessionContext);

            // 第二步：检查是否有权限执行Query
            accessControl.checkCanExecuteQuery(sessionContext.getIdentity());

            // 第三步（对应SQL执行流程第二步，后面小节详细介绍）：prepareQuery负责调用SqlParser完成SQL的						  //解析，生成抽象语法树（AST）
            preparedQuery = queryPreparer.prepareQuery(session, query);

            // 第四步：选择Query执行对应的ResourceGroup
            SelectionContext<C> selectionContext = resourceGroupManager.selectGroup(...);

  					// 第五步：生成DispatchQuery
            DispatchQuery dispatchQuery = dispatchQueryFactory.createDispatchQuery(
                    session,
                    sessionContext.getTransactionId(),
                    query,
                    preparedQuery,
                    slug,
                    selectionContext.getResourceGroupId());
  
						// 第六步：向ResourceGroupManager提交Query执行
            boolean queryAdded = queryCreated(dispatchQuery);
            if (queryAdded && !dispatchQuery.isDone()) {
                try {
                    resourceGroupManager.submit(dispatchQuery, selectionContext, dispatchExecutor);
                }
                catch (Throwable e) {
                    // dispatch query has already been registered, so just fail it directly
                    dispatchQuery.fail(e);
                }
            }
        }
    }
```



### **3.2 第二步：【Coordinator】词法与语法分析（生成AST）**

在这一步中，SqlParser拿到一个SQL字符串，通过词法语法解析后，生成抽象语法树（AST）。

SqlParser的实现，使用了Antlr4作为解析工具。它首先定义了Antlr4的SQL语法文件（即SqlBase.g4)，之后用它的代码生成工具自动生成了超过13K行SQL解析逻辑。SQL解析完成后会生成AST。

> 注：词法分析与语法分析都是由Antlr4生成，见下图所示

<img src="../../img/Snipaste_2023-08-24_14-30-26.png" alt="1" style="zoom:50%;" />

抽象语法树（AST，**是在生成语法分析树后，再对语法分析树进行visit生成的**）是用一种树形结构（即我们在大学数据结构与算法课程中学过的树）来表示SQL想要表述的语义，将一段SQL字符串结构化，以支持SQL执行引擎根据AST生成SQL执行计划。在Presto中，Node表示树的节点的抽象。根据语义不同，SQL AST中有多种不同类型的节点，它们继承自Node节点，如下所示：

> [antlr学习链接](https://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491631&idx=2&sn=d70dd5119f456986a9a7939914cc0f98&chksm=ebdb90bddcac19ab0523b46f6b82361eda6858063d42b4ada781bd449a08c9fba6b92c78d23e&scene=21#wechat_redirect)
>
> [antlr教程](https://wizardforcel.gitbooks.io/antlr4-short-course/content/calculator-visitor.html)

![1](../../img/Snipaste_2023-08-24_14-23-04.png)

对于第一步中接收到的SQL(TPC-DS Query55)，我们来看一下它对应的AST长什么样？（细节比较多，请一定要点击图片看大图）

![1](../../img/Snipaste_2023-08-24_16-55-37.png)

**这一步，我们可以拿到一个用户Statement来表示的根的AST树，在后面我们将会使用它来生成执行计划。**

**上面这张图就是通过visit模式遍历语法分析tree生成的，语法分析tree如下图所示**

![语法分析树](../../img/Snipaste_2023-08-24_17-04-16.png)

**继承Visitor类，对语法分析树进行visit来开发自己的业务逻辑代码，最终生成Statement来表示的根的AST。如下图所示，就是visit结束后返回的结果，等同于上面的AST**

![1](../../img/Snipaste_2023-08-24_16-51-54.png)



### 3.3 第三步：【Coordinator】创建QueryExecution并提交给ResourceGroupManager运行

在DispatchManager::createQueryInternal() 中的dispatchQueryFactory.createDispatchQuery()方法执行中，根据Statement的类型，生成QueryExecution。对于此文中我们举例的SQL，会生成SqlQueryExecution；对于Create Table这样的SQL，生成的是DataDefinitionExecution；

随后，这一步中生成的QueryExecution被包装到LocalDispatchQuery中，提交给ResourceGroupManager等待运行：

```java
// File: DispatchManager.java
// package: io.trino.dispatcher
// Method: createQueryInternal()
resourceGroupManager.submit(dispatchQuery, selectionContext, dispatchExecutor);
```

**之后在submit()中，经过一系列的异步操作（各种Future），代码辗转执行到了SqlQueryExecution::start()方法，此方法串起来了执行计划的生成以及调度。接下来看下怎么执行到SqlQueryExecution::start()方法中**

首先是通过线程池调用了如图所示方法栈:

![1](../../img/Snipaste_2023-09-03_12-30-16.png)

在waitForMinimumWorkers()方法中，通过调用`io.airlift.concurrent.MoreFutures.addSuccessCallback`方法异步执行lambda表达式（**addSuccessCallback方法的调用链具体可见上图所示**）。然后在lambda表达式中再通过addSuccessCallback方法执行` () -> startExecution(queryExecution), queryExecutor`，在startExecution(queryExecution)中最终会执行到SqlQueryExecution::start()方法，如下图所示：

![1](../../img/Snipaste_2023-09-03_12-53-00.png)

```java
// File: SqlQueryExecution.java
// package: io.trino.execution
// Method: start()

// 介绍：这里只节选了最核心的代码
public void start()
{
    ...
    // 生成logical plan，此方法进一步调用了doPlanQuery()
    PlanRoot plan = planQuery();
    // 生成数据源Connector的Connector，创建SqlStageExecution（Stage）、指定StageScheduler
  	registerDynamicFilteringQuery(plan);	 
    planDistribution(plan);
    ...
    QueryScheduler scheduler = queryScheduler.get();
    // Stage的调度，根据执行计划，将Task调度到Presto Worker上
    scheduler.start();
    ...
}
```



### **3.4 第四步：【Coordinator】语义分析(Analysis)、生成执行计划LogicalPlan**

















## 参考资料

- 本文参考知乎这篇文章，https://zhuanlan.zhihu.com/p/293775390

- 本文用于举例的SQL来自于TPC-DS Benchmark标准Query55，详见 [http://www.tpc.org/tpcds/](https://link.zhihu.com/?target=http%3A//www.tpc.org/tpcds/)

- Presto技术源码解析总结-一个SQL的奇幻之旅(上) [https://www.jianshu.com/p/3fccfa82e](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/3fccfa82e1ec)

- Presto技术源码解析总结-一个SQL的奇幻之旅(下) [https://www.jianshu.com/p/d8a3d7488](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/d8a3d7488358)

- 数据库大神 Goetz Graefe与他的论文 [https://scholar.google.com/cita](https://link.zhihu.com/?target=https%3A//scholar.google.com/citations%3Fuser%3DpdDeRScAAAAJ%26hl%3Den)

- SQL 优化之火山模型 [https://zhuanlan.zhihu.com/p/21](https://zhuanlan.zhihu.com/p/219516250)

- Facebook Presto论文，Presto: SQL on Everything

- Volcano 执行模型以及如何用Code Generation与列式存储优化SparkSQL的执行效率 [https://databricks.com/blog/201](https://link.zhihu.com/?target=https%3A//databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html)

- 代码生成技术（二）查询编译执行 https://zhuanlan.zhihu.com/p/58249033

- SQL 查询的分布式执行与调度 https://zhuanlan.zhihu.com/p/100949808

- Volcano执行模型论文解读 [https://zhuanlan.zhihu.com/p/34](https://zhuanlan.zhihu.com/p/34220915)

- Vectorized and Compiled Queries — Part 1 [https://medium.com/@tilakpatidar](https://link.zhihu.com/?target=https%3A//medium.com/%40tilakpatidar/vectorized-and-compiled-queries-part-1-37794c3860cc)

- Vectorized and Compiled Queries — Part 2 [https://medium.com/@tilakpatida](https://link.zhihu.com/?target=https%3A//medium.com/%40tilakpatidar/vectorized-and-compiled-queries-part-2-cd0d91fa189f)

- Vectorized and Compiled Queries — Part 3 [https://medium.com/@tilakpatida](https://link.zhihu.com/?target=https%3A//medium.com/%40tilakpatidar/vectorized-and-compiled-queries-part-3-807d71ec31a5)

- Presto Core Data Structures: Slice, Block & Page [https://zhuanlan.zhihu.com/p/60](https://zhuanlan.zhihu.com/p/60813087)

- Presto 数据如何进行shuffle [https://zhuanlan.zhihu.com/p/61](https://zhuanlan.zhihu.com/p/61565957)

- Presto 由Stage到Task的旅程 https://zhuanlan.zhihu.com/p/55785284

  