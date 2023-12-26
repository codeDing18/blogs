

> 注：coordinator分发task给worker用的是TaskResource的接口

# Coordinator的接口

---

## QueuedStatementResource

POST /v1/statement

Cli第一次发送的query请求会到这，接收Query请求后，生成QueryId，将Query加入待执行队列，返回的Response中包含了nextURI

```java
public Response postStatement(
        String statement,
        @Context HttpServletRequest servletRequest,
        @Context HttpHeaders httpHeaders,
        @Context UriInfo uriInfo)
```



GET /v1/statement/queued/{queryId}/{slug}/{token}

Cli根据第一次请求中的返回结果，会接着发起第二次请求到这，在这里trino执行此请求时，会真正把Query提交到DispatchManager::createQueryInternal()执行（就是从这开始语法分析、语义分析、逻辑计划、执行计划）

```java
public void getStatus(
        @PathParam("queryId") QueryId queryId,
        @PathParam("slug") String slug,
        @PathParam("token") long token,
        @QueryParam("maxWait") Duration maxWait,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
```



DELETE /v1/statement/queued/{queryId}/{slug}/{token}

cli请求取消该query的运行

```java
public Response cancelQuery(
        @PathParam("queryId") QueryId queryId,
        @PathParam("slug") String slug,
        @PathParam("token") long token)
```



## NodeResource

GET /v1/node

获取当前节点的状态

```java
public Collection<HeartbeatFailureDetector.Stats> getNodeStats()
```



## QueryResource

GET /v1/query

获取所有的query信息

```java
public List<BasicQueryInfo> getAllQueryInfo(@QueryParam("state") String stateFilter, @Context HttpServletRequest servletRequest, @Context HttpHeaders httpHeaders)
```



GET /v1/query/{queryId}

获取具体的某个query的执行状态信息

```java
public Response getQueryInfo(@PathParam("queryId") QueryId queryId, @Context HttpServletRequest servletRequest, @Context HttpHeaders httpHeaders)
```



DELETE /v1/query/{queryId}

取消某个query的执行

```java
public void cancelQuery(@PathParam("queryId") QueryId queryId, @Context HttpServletRequest servletRequest, @Context HttpHeaders httpHeaders)
```



## QueryStateInfoResource

GET /v1/queryState

获取所有的query状态信息

```java
public List<QueryStateInfo> getQueryStateInfos(@QueryParam("user") String user, @Context HttpServletRequest servletRequest, @Context HttpHeaders httpHeaders)
```



GET /v1/queryState/{queryId}

获取指定的query状态信息

```java
public QueryStateInfo getQueryStateInfo(@PathParam("queryId") String queryId, @Context HttpServletRequest servletRequest, @Context HttpHeaders httpHeaders)
```



## ResourceGroupStateInfoResource

GET /v1/resourceGroupState/{resourceGroupId: .+}

获取资源组信息

```java
public ResourceGroupInfo getQueryStateInfos(@PathParam("resourceGroupId") String resourceGroupIdString)
```



## ExecutingStatementResource

GET /v1/statement/executing/{queryId}/{slug}/{token}

获取指定query的结果

```java
public void getQueryResults(
        @PathParam("queryId") QueryId queryId,
        @PathParam("slug") String slug,
        @PathParam("token") long token,
        @QueryParam("maxWait") Duration maxWait,
        @QueryParam("targetResultSize") DataSize targetResultSize,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
```



# coo与work都有的接口

---

## ServerInfoResource

GET /v1/info

获取trino服务信息

```java
public ServerInfo getInfo()
```



PUT /v1/info/state

更新节点状态

```java
public Response updateState(NodeState state)
```



GET /v1/info/state

获取节点状态

```java
public NodeState getServerState()
```



GET /v1/info/coordinator

coordinator存活返回空白页否则就是404

```java
public Response getServerCoordinator()
```

## StatusResource

GET /v1/status

获取节点状态信息，比如nodeId、nodeVersion、externalAddress等

```java
public NodeStatus getStatus()
```



## TaskExecutorResource

GET /v1/maxActiveSplits

遍历每个当前正在运行的split，筛选出大于stuckSplitsWarningThreshold的的split

```java
public String getMaxActiveSplit()
```

## ThreadResource

GET /v1/thread

获取线程信息

```java
public List<Info> getThreadInfo()
```



## MemoryResource

GET /v1/memory

获取当前work节点的内存信息

```java
public MemoryInfo getMemoryInfo()
```



## TaskResource

GET /v1/task

获取所有的task信息

```java
public List<TaskInfo> getAllTaskInfo(@Context UriInfo uriInfo)
```



POST /v1/task/{taskId}

创建一个新的task或者更新task的状态，**coordinator分发task给worker就是这个接口**

```java
public void createOrUpdateTask(
        @PathParam("taskId") TaskId taskId,
        TaskUpdateRequest taskUpdateRequest,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
```



GET /v1/task/{taskId}

获取指定的task的信息，比如task状况、最后一次心跳时间等

```java
public void getTaskInfo(
        @PathParam("taskId") TaskId taskId,
        @HeaderParam(TRINO_CURRENT_VERSION) Long currentVersion,
        @HeaderParam(TRINO_MAX_WAIT) Duration maxWait,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
```



GET /v1/task/{taskId}/status

获取指定task的状况，比如task状态、nodeId、queuedPartitionedDrivers、queuedPartitionedSplitsWeight、outputDataSize等

```java
public void getTaskStatus(
        @PathParam("taskId") TaskId taskId,
        @HeaderParam(TRINO_CURRENT_VERSION) Long currentVersion,
        @HeaderParam(TRINO_MAX_WAIT) Duration maxWait,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
```



DELETE  /v1/task/{taskId}

根据taskId删除task

```java
public TaskInfo deleteTask(
        @PathParam("taskId") TaskId taskId,
        @QueryParam("abort") @DefaultValue("true") boolean abort,
        @Context UriInfo uriInfo)
```



POST  /v1/task/{taskId}/fail

使指定的taskId失败

```java
public TaskInfo failTask(
        @PathParam("taskId") TaskId taskId,
        FailTaskRequest failTaskRequest,
        @Context UriInfo uriInfo)
```



GET   /v1/task/{taskId}/results/{bufferId}/{token}

获取指定的task输出给下游某个task任务（这个task由bufferId指定）的数据

```java
public void getResults(
        @PathParam("taskId") TaskId taskId,
        @PathParam("bufferId") PipelinedOutputBuffers.OutputBufferId bufferId,
        @PathParam("token") long token,
        @HeaderParam(TRINO_MAX_SIZE) DataSize maxSize,
        @Suspended AsyncResponse asyncResponse)
```



GET   /v1/task/{taskId}/results/{bufferId}/{token}/acknowledge

确认先前收到的结果

```java
public void acknowledgeResults(
        @PathParam("taskId") TaskId taskId,
        @PathParam("bufferId") PipelinedOutputBuffers.OutputBufferId bufferId,
        @PathParam("token") long token)
```



DELETE  /v1/task/{taskId}/results/{bufferId}

删除task输出给指定bufferId的数据

```java
public void destroyTaskResults(
        @PathParam("taskId") TaskId taskId,
        @PathParam("bufferId") PipelinedOutputBuffers.OutputBufferId bufferId,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
```



