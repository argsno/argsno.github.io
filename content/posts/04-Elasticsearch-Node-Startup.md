+++
title = '04. Elasticsearch 节点启动'
date = 2024-03-20T23:26:40+08:00
draft = false
+++

## 启动流程核心工作
总体而言，节点启动流程的任务包括：
- 解析配置，包括配置文件和命令行参数。
- 检查外部环境和内部环境，例如，JVM版本、操作系统内核参数等。
- 初始化内部资源，创建内部模块，初始化探测器。
- 启动各个子模块和keepalive线程。

## 启动流程分析
启动入口：`org.elasticsearch.bootstrap.Elasticsearch#main(java.lang.String[])`

### 检测外部环境
在创建Node时覆盖`validateNodeBeforeAcceptingRequests`方法，在接收请求之前执行`BootstracpChecks.check`执行各项外部环境检查：
```java
node = new Node(environment) {  
    @Override  
    protected void validateNodeBeforeAcceptingRequests(  
        final BootstrapContext context,  
        final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {  
        BootstrapChecks.check(context, boundTransportAddress, checks);  
    }  
};
```

执行的检查包括：
```java
// the list of checks to execute  
static List<BootstrapCheck> checks() {  
    final List<BootstrapCheck> checks = new ArrayList<>();  
    checks.add(new HeapSizeCheck());  
    final FileDescriptorCheck fileDescriptorCheck  
        = Constants.MAC_OS_X ? new OsXFileDescriptorCheck() : new FileDescriptorCheck();  
    checks.add(fileDescriptorCheck);  
    checks.add(new MlockallCheck());  
    if (Constants.LINUX) {  
        checks.add(new MaxNumberOfThreadsCheck());  
    }  
    if (Constants.LINUX || Constants.MAC_OS_X) {  
        checks.add(new MaxSizeVirtualMemoryCheck());  
    }  
    if (Constants.LINUX || Constants.MAC_OS_X) {  
        checks.add(new MaxFileSizeCheck());  
    }  
    if (Constants.LINUX) {  
        checks.add(new MaxMapCountCheck());  
    }  
    checks.add(new ClientJvmCheck());  
    checks.add(new UseSerialGCCheck());  
    checks.add(new SystemCallFilterCheck());  
    checks.add(new OnErrorCheck());  
    checks.add(new OnOutOfMemoryErrorCheck());  
    checks.add(new EarlyAccessCheck());  
    checks.add(new G1GCCheck());  
    return Collections.unmodifiableList(checks);  
}
```
如果插件有额外的检查项也会加进去。

- HeapSizeCheck：堆大小检查
- FileDescriptorCheck：文件描述符检查
- MlockallCheck：内存锁定检查
- MaxNumberOfThreadsCheck：最大线程数检查
- MaxSizeVirtualMemoryCheck：最大虚拟内存检查
- MaxFileSizeCheck：最大文件大小检查
- MaxMapCountCheck：虚拟内存区域最大数量检查
- ClientJvmCheck：JVM Client模式检查
- UseSerialGCCheck：串行收集检查
- SystemCallFilterCheck：系统调用过滤器检查
- OnErrorCheck：OnError检查
- OnOutOfMemoryErrorCheck：OnOutOfMemory检查
- EarlyAccessCheck：early-access快照版本检查
- G1GCCheck：G1GC检查

### 启动内部模块
环境检查完毕，调用`Node.start()`开始启动节点各子模块。子模块在Node类中创建，启动它们时调用各自的start()方法，例如：
- discovery.start()；
- clusterService.start()；
- nodeConnectionsService.start()；
子模块的start方法基本就是初始化内部数据、创建线程池、启动线程池等操作。

### 启动keepalive线程
调用`keepAliveThread.start()`方法启动keepalive线程，线程本身不做具体的工作。主线程执行完启动流程后会退出，keepalive线程是唯一的用户线程，作用是保持进程运行。

## 节点关闭流程
设想当我们为 ES 集群更新配置、升级版本时，需要通过“kill”ES进程来关闭节点。
- 但是kill操作是否安全？
- 如果此时节点有正在执行的读写操作会有什么影响？
- 如果节点是Master该如何处理？
- 关闭流程是怎么实现的？
- kill节点都会带来哪些风险？

ES进程会捕获SIGTERM信号（kill命令默认信号）进行处理，调用各模块的stop方法，让它们有机会停止服务，安全退出。

进程重启期间，如果主节点被关闭，则集群会重新选主，在这期间，集群有一个短暂的无主状态。如果集群中的主节点是单独部署的，则新主当选后，可以跳过gateway和recovery流程，否则新主需要重新分配旧主所持有的分片：提升其他副本为主分片，以及分配新的副分片。

如果数据节点被关闭，则读写请求的TCP连接也会因此关闭，对客户端来说写操作执行失败。但写流程已经到达Engine环节的会正常写完，只是客户端无法感知结果。此时客户端重试，如果使用自动生成ID，则数据内容会重复。

综合来说，滚动升级产生的影响是中断当前写请求，以及主节点重启可能引起的分片分配过程。

## 关闭流程分析

在节点启动过程中，Bootstrap#setup 方法中添加了 shutdown hook，当进程收到系统SIGTERM（kill命令默认信号）或SIGINT信号时，调用Node#close方法，执行节点关闭流程。

每个模块的Service中都实现了doStop和doClose，用于处理这个模块的正常关闭流程。节点总的关闭流程位于Node#close，在close方法的实现中，先调用一遍各个模块的doStop，然后再次遍历各个模块执行doClose。

综合来看，关闭顺序大致如下：
- 关闭快照和HTTPServer，不再响应用户REST请求。
- 关闭集群拓扑管理，不再响应ping请求。
- 关闭网络模块，让节点离线。
- 执行各个插件的关闭流程。
- 关闭IndicesService。
- 最后才关闭IndicesService，是因为这期间需要等待释放的资源最多，时间最长。

## 分片读写过程中执行关闭

写入过程中关闭：线程在写入数据时，会对Engine加写锁。IndicesService的doStop方法对本节点上全部索引并行执行removeIndex，当执行到Engine的flushAndClose（先flush然后关闭Engine），也会对Engine加写锁。由于写入操作已经加了写锁，此时写锁会等待，直到写入执行完毕。因此数据写入过程不会被中断。但是由于网络模块被关闭，客户端的连接会被断开。客户端应当作为失败处理，虽然ES服务端的写流程还在继续。

读取过程中关闭：线程在读取数据时，会对Engine加读锁。flushAndClose时的写锁会等待读取过程执行完毕。但是由于连接被关闭，无法发送给客户端，导致客户端读失败。

节点关闭过程中，IndicesService的doStop对Engine设置了超时，如果flushAndClose一直等待，则CountDownLatch.await默认1天才会继续后面的流程。

## 主节点被关闭

主节点被关闭时，没有想象中的特殊处理，节点正常执行关闭流程，当TransportService模块被关闭后，集群重新选举新Master。因此，滚动重启期间会有一段时间处于无主状态。

## 参考

- [Elasticsearch源码解析与优化实战](https://book.douban.com/subject/30386800/)


