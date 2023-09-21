# Presto Sql一直卡在Finishing

### presto源码分析

当用户抛出问题说他们有Sql卡在Finishing状态好几个小时，我们立马对Presto的状态机进行分析,如下代码，当作业进入FINISHING，则之后就会进行commit操作，当commit操作完成之后，立马会调用回掉方法，成功或者失败，都会进行相应的状态。所以作业肯定卡在commit阶段。
```java
//io/prestosql/execution/QueryStateMachine.java:760
public boolean transitionToFinishing() {
  if (transactionId.isPresent() && transactionManager.transactionExists(transactionId.get()) && transactionManager.isAutoCommit(transactionId.get())) {
            ListenableFuture<?> commitFuture = transactionManager.asyncCommit(transactionId.get());
            Futures.addCallback(commitFuture, new FutureCallback<Object>()
            {
                @Override
                public void onSuccess(@Nullable Object result)
              {
                    transitionToFinished();
                }
                @Override
                public void onFailure(Throwable throwable)
                {
                    transitionToFailed(throwable);
                }
            }, directExecutor());
        }
        else {
            transitionToFinished();
        }
}
```
跟着commit代码链路，最总会走到如下代码,insert into sql会将hive_staging下的文件，一个一个的rename到分区路径中，而任务是交给线程池executor来完成，线程池默认大小20（hive.max-concurrent-file-renames=20），所以这边就怀疑应该是和rename操作导致的线程hang住，作业一直处在finishing状态。
```java
//io/prestosql/plugin/hive/metastore/SemiTransactionalHiveMetastore.java:1889
private static void asyncRename() {
  for (String fileName : fileNames) {
            Path source = new Path(currentPath, fileName);
            Path target = new Path(targetPath, fileName);
            Path targetParent = target.getParent();
            fileRenameFutures.add(CompletableFuture.runAsync(() -> {
                if (cancelled.get()) {
                    return;
                }
                try {
                    if (!fileSystem.exists(targetParent)) {
                        fileSystem.mkdirs(targetParent);
                    }
                    if (fileSystem.exists(target) || !fileSystem.rename(source, target)) {
                        throw new PrestoException(HIVE_FILESYSTEM_ERROR, format("Error moving data files from %s to final location %s", source, target));
                    }
                }
                catch (IOException e) {
                    throw new PrestoException(HIVE_FILESYSTEM_ERROR, format("Error moving data files from %s to final location %s", source, target), e);
                }
            }, executor));
  }
}
```

### jstack分析
接下来，对presto coordinator进行jstack分析验证，如下jstack日志,20个线程都卡在了Client.wait方法
```shell
"hive-hive-5648" #70895 daemon prio=5 os_prio=0 tid=0x00007fa7f039c000 nid=0x1169c in Object.wait() [0x00007f729df29000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        at java.lang.Object.wait(Object.java:502)
        at org.apache.hadoop.util.concurrent.AsyncGet$Util.wait(AsyncGet.java:59)
        at org.apache.hadoop.ipc.Client.getRpcResponse(Client.java:1499)
        - locked <0x00007f8218f389f0> (a org.apache.hadoop.ipc.Client$Call)
        at org.apache.hadoop.ipc.Client.call(Client.java:1457)
        at org.apache.hadoop.ipc.Client.call(Client.java:1367)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:228)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:116)
        at com.sun.proxy.$Proxy212.rename(Unknown Source)
        at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.rename(ClientNamenodeProtocolTranslatorPB.java:579)
        at sun.reflect.GeneratedMethodAccessor3516.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:422)
        at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeMethod(RetryInvocationHandler.java:165)
        at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invoke(RetryInvocationHandler.java:157)
        at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeOnce(RetryInvocationHandler.java:95)
        - locked <0x00007f8218f38a78> (a org.apache.hadoop.io.retry.RetryInvocationHandler$Call)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:359)
        at com.sun.proxy.$Proxy213.rename(Unknown Source)
        at org.apache.hadoop.hdfs.DFSClient.rename(DFSClient.java:1525)
        at org.apache.hadoop.hdfs.DistributedFileSystem.rename(DistributedFileSystem.java:872)
        at org.apache.hadoop.fs.viewfs.ViewFileSystem.rename(ViewFileSystem.java:508)
        at io.prestosql.plugin.hive.metastore.SemiTransactionalHiveMetastore.lambda$asyncRename$38(SemiTransactionalHiveMetastore.java:1925)
        at io.prestosql.plugin.hive.metastore.SemiTransactionalHiveMetastore$$Lambda$5691/830112668.run(Unknown Source)
        at java.util.concurrent.CompletableFuture$AsyncRun.run(CompletableFuture.java:1626)
        at io.airlift.concurrent.BoundedExecutor.drainQueue(BoundedExecutor.java:78)
        at io.airlift.concurrent.BoundedExecutor$$Lambda$4167/1623362620.run(Unknown Source)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)
```
```shell
root@jssz-ai-offline-presto-01:/data/log/dump# grep -i 'lambda$asyncRename' thread | wc
     20      40    2680
```
接下来看看hadoop RPC相关code，如下代码客户端线程在发送完RPC请求数据之后，通过调用wait方法，线程进行无限期wait
```java
//org/apache/hadoop/ipc/Client.java:1445
return getRpcResponse(call, connection, -1, null);
//org/apache/hadoop/ipc/Client.java:1482
private Writable getRpcResponse(final Call call, final Connection connection,
      final long timeout, final TimeUnit unit) throws IOException {
    while (!call.done) {
        try {
          AsyncGet.Util.wait(call, timeout, unit);
          if (timeout >= 0 && !call.done) {
            return null;
          }
        } catch (InterruptedException ie) {
          Thread.currentThread().interrupt();
          throw new InterruptedIOException("Call interrupted");
        }
   }
}
```

### 想办法复现
在偶然情况下，我们发现了用户如下类似sql，当分区路径20201011是一个文件的时候，是稳定复现的。
```shell
insert into tmp_bdp.z_guojh_test select 'tt' as a, 1 as b, '20201011' as log_date;
```
上述sql在ai-offline能够稳定复现，但是在我线下维护的集群和bsk-adhoc集群都未能复现，这个问题之后再分析。
最后实在没办法，跑到了ai-offline coordinator容器里面进行了环境搭建，简单的hadoop fs -mv xxx yyy就复现了，客户端hang住，于是对客户端进行jstack,如下，客户端也是卡在了Client.wait方法，而Connection线程会启动DataInputStream去nnproxy拿数据，可以看到，该线程一直hang住，当结果拿到之后，就会唤醒Call线程。
```shell
"IPC Client (1561063579) connection to nnproxy-01.bilibili.co/10.70.137.25:56310 from ai@BILIBILI.CO" #27 daemon prio=5 os_prio=0 tid=0x00007f394551d000 nid=0x10620 runnable [0x00007f391ac41000]
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
        at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
        at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
        at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
        - locked <0x00000000ec2a3af8> (a sun.nio.ch.Util$3)
        - locked <0x00000000ec2a3a70> (a java.util.Collections$UnmodifiableSet)
        - locked <0x00000000ec2a36a8> (a sun.nio.ch.EPollSelectorImpl)
        at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
        at org.apache.hadoop.net.SocketIOWithTimeout$SelectorPool.select(SocketIOWithTimeout.java:335)
        at org.apache.hadoop.net.SocketIOWithTimeout.doIO(SocketIOWithTimeout.java:157)
        at org.apache.hadoop.net.SocketInputStream.read(SocketInputStream.java:161)
        at org.apache.hadoop.net.SocketInputStream.read(SocketInputStream.java:131)
        at java.io.FilterInputStream.read(FilterInputStream.java:133)
        at java.io.BufferedInputStream.read1(BufferedInputStream.java:284)
        at java.io.BufferedInputStream.read(BufferedInputStream.java:345)
        - locked <0x00000000ec2cc228> (a java.io.BufferedInputStream)
        at java.io.DataInputStream.read(DataInputStream.java:149)
        at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
        at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
        - locked <0x00000000ed033060> (a java.io.BufferedInputStream)
        at java.io.FilterInputStream.read(FilterInputStream.java:83)
        at java.io.FilterInputStream.read(FilterInputStream.java:83)
        at org.apache.hadoop.ipc.Client$Connection$PingInputStream.read(Client.java:554)
        at java.io.DataInputStream.readInt(DataInputStream.java:387)
        at org.apache.hadoop.ipc.Client$IpcStreams.readResponse(Client.java:1794)
        at org.apache.hadoop.ipc.Client$Connection.receiveRpcResponse(Client.java:1163)
        at org.apache.hadoop.ipc.Client$Connection.run(Client.java:1059)


"main" #1 prio=5 os_prio=0 tid=0x00007f3944013000 nid=0x105e3 in Object.wait() [0x00007f394bc8d000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000edf5b620> (a org.apache.hadoop.ipc.Client$Call)
        at java.lang.Object.wait(Object.java:502)
        at org.apache.hadoop.util.concurrent.AsyncGet$Util.wait(AsyncGet.java:59)
        at org.apache.hadoop.ipc.Client.getRpcResponse(Client.java:1477)
        - locked <0x00000000edf5b620> (a org.apache.hadoop.ipc.Client$Call)
        at org.apache.hadoop.ipc.Client.call(Client.java:1435)
        at org.apache.hadoop.ipc.Client.call(Client.java:1345)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:227)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:116)
        at com.sun.proxy.$Proxy10.rename(Unknown Source)
        at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.rename(ClientNamenodeProtocolTranslatorPB.java:510)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:409)
        at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeMethod(RetryInvocationHandler.java:163)
        at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invoke(RetryInvocationHandler.java:155)
        at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeOnce(RetryInvocationHandler.java:95)
        - locked <0x00000000edf416e0> (a org.apache.hadoop.io.retry.RetryInvocationHandler$Call)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:346)
        at com.sun.proxy.$Proxy11.rename(Unknown Source)
        at org.apache.hadoop.hdfs.DFSClient.rename(DFSClient.java:1515)
        at org.apache.hadoop.hdfs.DistributedFileSystem.rename(DistributedFileSystem.java:718)
        at org.apache.hadoop.fs.viewfs.ViewFileSystem.rename(ViewFileSystem.java:493)
        at org.apache.hadoop.fs.shell.MoveCommands$Rename.processPath(MoveCommands.java:114)
        at org.apache.hadoop.fs.shell.CommandWithDestination.processPath(CommandWithDestination.java:262)
        at org.apache.hadoop.fs.shell.Command.processPaths(Command.java:327)
        at org.apache.hadoop.fs.shell.Command.processPathArgument(Command.java:299)
        at org.apache.hadoop.fs.shell.CommandWithDestination.processPathArgument(CommandWithDestination.java:257)
        at org.apache.hadoop.fs.shell.Command.processArgument(Command.java:281)
        at org.apache.hadoop.fs.shell.Command.processArguments(Command.java:265)
        at org.apache.hadoop.fs.shell.CommandWithDestination.processArguments(CommandWithDestination.java:228)
        at org.apache.hadoop.fs.shell.FsCommand.processRawArguments(FsCommand.java:119)
        at org.apache.hadoop.fs.shell.Command.run(Command.java:175)
        at org.apache.hadoop.fs.FsShell.run(FsShell.java:317)
        at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:76)
        at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:90)
        at org.apache.hadoop.fs.FsShell.main(FsShell.java:380)
```
于是怀疑是不是后段有问题，根本没将数据传输回客户端，于是修改hdfs-site.xml指向我临时搭建的nnproxy，于是发现了问题。

### nnproxy bug 分析
如下代码，NPE，太明显了，于是我们跟着RPC Server端代码，发现Server在构建数据返回客户端的时候，在setErrorDetail拿到的errorCode为null，导致了线程推出，客户端拿不到数据，所以就一直hang住了。
```shell
20/12/09 19:22:01 [IPC Server handler 712 on 56310] INFO Server: IPC Server handler 712 on 56310 caught an exception
java.lang.NullPointerException
	at org.apache.hadoop.ipc.protobuf.RpcHeaderProtos$RpcResponseHeaderProto$Builder.setErrorDetail(RpcHeaderProtos.java:4945)
	at org.apache.hadoop.ipc.Server.setupResponse(Server.java:2981)
	at org.apache.hadoop.ipc.Server.access$200(Server.java:125)
	at org.apache.hadoop.ipc.Server$RpcCall.run(Server.java:908)
	at org.apache.hadoop.ipc.Server$RpcCall.run(Server.java:818)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1864)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2565)
```

### nnproxy修复bug
[查看PR](https://git.bilibili.co/datacenter/bilibili-nnproxy/-/merge_requests/5)

### 结论
- 1.当nnproxy作为客户端和Namenode通信，如果NN返回的IOException，nnproxy在异常处理上是没问题的，当NN返回的是其他的Exception（IllegalStageException）的时候，nnproxy处理的有点问题，丢失了Exception的ErrorCode，导致Server类封装数据时候抛NPE
- 2.为什bsk-adhoc集群没能复现，因为bsk-adhoc使用的hive.keytab,hive在hdfs是超级账号，NN返回给nnproxy的是ParentNotDirectoryException,该Exception为IOExcetion，所以没能复现。

# PageTransportTimeoutException

---

```bash
io.trino.operator.PageTransportTimeoutException: Encountered too many errors talking to a worker node. The node may have crashed or be under too much load. This is probably a transient issue, so please retry your query in a few minutes. (http://5.5.243.95:8080/v1/task/20230921_014654_00001_6zuir.0.0.0/results/0/0 - 57 failures, failure duration 301.56s, total failed request time 301.67s)
	at io.trino.operator.HttpPageBufferClient$1.onFailure(HttpPageBufferClient.java:504)
	at com.google.common.util.concurrent.Futures$CallbackListener.run(Futures.java:1127)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
	at java.base/java.lang.Thread.run(Thread.java:833)
Caused by: java.io.UncheckedIOException: Failed communicating with server: http://5.5.243.95:8080/v1/task/20230921_014654_00001_6zuir.0.0.0/results/0/0
	at io.airlift.http.client.ResponseHandlerUtils.propagate(ResponseHandlerUtils.java:22)
	at io.trino.operator.HttpPageBufferClient$PageResponseHandler.handleException(HttpPageBufferClient.java:663)
	at io.trino.operator.HttpPageBufferClient$PageResponseHandler.handleException(HttpPageBufferClient.java:650)
	at io.airlift.http.client.jetty.JettyResponseFuture.failed(JettyResponseFuture.java:152)
	at io.airlift.http.client.jetty.BufferingResponseListener.onComplete(BufferingResponseListener.java:84)
	at org.eclipse.jetty.client.ResponseNotifier.notifyComplete(ResponseNotifier.java:213)
	at org.eclipse.jetty.client.ResponseNotifier.notifyComplete(ResponseNotifier.java:205)
	at org.eclipse.jetty.client.HttpExchange.notifyFailureComplete(HttpExchange.java:285)
	at org.eclipse.jetty.client.HttpExchange.abort(HttpExchange.java:256)
	at org.eclipse.jetty.client.HttpConversation.abort(HttpConversation.java:159)
	at org.eclipse.jetty.client.HttpRequest.abort(HttpRequest.java:865)
	at org.eclipse.jetty.client.HttpDestination.abort(HttpDestination.java:530)
	at org.eclipse.jetty.client.HttpDestination.failed(HttpDestination.java:294)
	at org.eclipse.jetty.client.AbstractConnectionPool$FutureConnection.failed(AbstractConnectionPool.java:574)
	at org.eclipse.jetty.util.Promise$Wrapper.failed(Promise.java:169)
	at org.eclipse.jetty.client.HttpClient$1$1.failed(HttpClient.java:628)
	at org.eclipse.jetty.util.Promise$1.failed(Promise.java:94)
	at org.eclipse.jetty.io.ClientConnector.connectFailed(ClientConnector.java:539)
	at org.eclipse.jetty.io.ClientConnector$ClientSelectorManager.connectionFailed(ClientConnector.java:584)
	at org.eclipse.jetty.io.ManagedSelector$Connect.failed(ManagedSelector.java:964)
	at org.eclipse.jetty.io.ManagedSelector$Connect.run(ManagedSelector.java:953)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	... 3 more
Caused by: java.net.SocketTimeoutException: Connect Timeout
	... 7 more
```

> 事故背景：自己电脑上启动了trino服务，然后像show tables;这种最简单的查询也一直没返回，最后报了如上错误。

**可能的原因**

- High CPU

  Tune down the `task.concurrency` to, for example, 8

- High memory

  In the `jvm.config`, `-Xmx` should no more than 80% total memory. In the `config.properties`, `query.max-memory-per-node` should be no more than the half of `Xmx` number.

- Low open file limit

  Set in the `/etc/security/limits.conf` a larger number for the Presto process. The default is definitely way too low.

