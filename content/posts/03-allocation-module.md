+++
title = '03. Elasticsearch allocation模块分析'
date = 2024-03-18T23:17:44+08:00
draft = false
+++

## 节点发现

当一个节点启动时，首先会尝试连接到配置列表（`discovery.seed_hosts`）中指定的种子节点，获取集群中其他节点的信息。并从其他节点接收集群的状态信息，包括集群中的其他节点列表，集群配置以及主节点的信息等。并尝试加入一个现有的集群，如果找不到，可以选择自我选举成为主节点，或者根据配置等待加入一个集群。

## 选举主节点

假设有若干节点正在启动，集群启动的第一件事是从已知的活跃机器列表中选择一个作为主节点，选主之后的流程由主节点触发。

ES的选主算法是基于Bully算法的改进，主要思路是对节点ID排序，取ID值最大的节点作为Master，每个节点都运行这个流程。

基于节点ID排序的简单选举算法有三个附加约定条件：
1. 参选人数需要过半，达到 quorum（多数）后就选出了临时的主。
2. 得票数需过半。某节点被选为主节点，必须判断加入它的节点数过半，才确认Master身份。
3. 当探测到节点离开事件时，必须判断当前节点数是否过半。如果达不到 quorum，则放弃Master身份，重新加入集群。

## 选举集群元信息

Master的第一个任务是选举元信息，让各节点把各自存储的元信息发过来，根据版本号确定最新的元信息，然后把这个信息广播下去，这样集群的所有节点都有了最新的元信息。集群元信息的选举包括两个级别：集群级和索引级。

为了集群一致性，参与选举的元信息数量需要过半，Master发布集群状态成功的规则也是等待发布成功的节点数过半。在选举过程中，不接受新节点的加入请求。集群元信息选举完毕后，Master发布首次集群状态，然后开始选举shard级元信息。

## allocation过程

选举shard级元信息，构建内容路由表，是在allocation模块完成的。

在初始阶段，所有的shard都处于`ShardRoutingState.UNASSIGNED`（未分配）状态。ES中通过分配过程决定哪个分片位于哪个节点，重构内容路由表。

### 1. 选主分片

此时，Master不知道主分片在哪，它向集群的所有节点询问：大家把\[website]\[0]分片的元信息发给我。然后根据某种策略选一个分片作为主分片。ES 5.x以下的版本，通过对比shard级元信息的版本号来决定。

### 2. 选副本分片

主分片选举完成后，从上一个过程汇总的 shard信息中选择一个副本作为副本分片。

最后，allocation过程中允许新启动的节点加入集群。

## index recovery

为什么需要recovery？对于主分片来说，可能有一些数据没来得及刷盘；对于副分片来说，一是没刷盘，二是主分片写完了，副分片还没来得及写，主副分片数据不一致。

### 1. 主分片recovery

主分片通过将最后一次提交（Lucene的一次提交就是一次 fsync 刷盘的过程）之后的 translog中进行重放，建立Lucene索引，如此完成主分片的recovery。

### 2. 副本分片recovery

副本分片需要恢复成与主分片一致（类似Redis主从之间需要进行同步）。在6.0版本中，恢复分成两阶段执行。

· phase1：在主分片所在节点，获取translog保留锁，从获取保留锁开始，会保留translog不受其刷盘清空的影响。然后调用Lucene接口把shard做快照，这是已经刷磁盘中的分片数据。把这些shard数据复制到副本节点。在phase1完毕前，会向副分片节点发送告知对方启动engine，在phase2开始之前，副分片就可以正常处理写请求了。

· phase2：对translog做快照，这个快照里包含从phase1开始，到执行translog快照期间的新增索引。将这些translog发送到副分片所在节点进行重放。

## 参考

- [Elasticsearch源码解析与优化实战](https://book.douban.com/subject/30386800/)
