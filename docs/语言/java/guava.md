ListeningExecutorService

ListenableFuture

Executor



```
java.util.concurrent.Executor
com.google.common.util.concurrent.DirectExecutor
io.airlift.concurrent.MoreFutures.addSuccessCallback
```





```java
ListeningExecutorService executor=new DecoratingListeningExecutorService(...)
ListenableFuture<QueryExecution> queryExecutionFuture = executor.submit(...)
```

