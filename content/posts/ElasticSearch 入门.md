---
title: "ElasticSearch 入门"
date: 2019-05-19T14:26:25+08:00
draft: false
---

# 基本概念

## Node与Cluster
ElasticSearch作为分布式的搜索引擎，支持多台服务器协同工作，每台服务器上也可以同时运行多个实例。
每个实例称为一个节点（node）。每个节点有`cluster.name`属性，相同的`cluster.name`的节点构成一个集群（cluster）。

## Index
ElasticSearch会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。
所以，Elastic 数据管理的顶层单位就叫做 Index（索引）。它是单个数据库的同义词。每个 Index （即数据库）的名字必须是小写。
下面的命令可以查看当前节点的所有 Index。
```bash
$ curl -X GET 'http://localhost:9200/_cat/indices?v'
```

## Document
Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。
Document 使用 JSON 格式表示，下面是一个例子。
```json
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}
```
同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

## Type
Document 可以分组，比如weather这个 Index 里面，可以按城市分组（北京和上海），也可以按气候分组（晴天和雨天）。这种分组就叫做 Type，它是虚拟的逻辑分组，用来过滤 Document。

不同的 Type 应该有相似的结构（schema），举例来说，id字段不能在这个组是字符串，在另一个组是数值。这是与关系型数据库的表的一个区别。性质完全不同的数据（比如products和logs）应该存成两个 Index，而不是一个 Index 里面的两个 Type（虽然可以做到）。

下面的命令可以列出每个 Index 所包含的 Type。
```bash
$ curl 'localhost:9200/_mapping?pretty=true'
```
根据规划，Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。

# 原理剖析

## es 写数据过程

![](http://ww1.sinaimg.cn/large/006tNc79ly1g3ag1epv8cj30k009yq38.jpg)

1. 客户端选择一个node 发送请求过去，这个node 就是 coordinating node（协调节点）。
2. coordinating node 对document 进行路由，将请求转发给对应的node（有primary shard）。
3. 实际的node 上的 primary shard 处理请求，然后将数据同步到 replica node。
4. coordinating node 如果发现 primary node 和所有 replica node 都搞定之后，就返回响应结果给客户端。

## es 读数据过程

可以通过 doc id 来查询，会根据 doc id 进行hash，判断出来当时把 doc id 分配到了哪个shard 上面去，从那个shard 去查询。

1. 客户端发送请求到任意一个node，成为 coordinate node。
2. coordinate node 对 doc id 进行哈希路由，将请求转发到对应的node，此时会使用 round-robin 随机轮询算法，在 primary shard 以及其所有replica 中随机选择一个，让读请求负载均衡。
3. 接收请求的node 返回document 给 coordinate node。
4. coordinate node 返回document 给客户端。

## Elasticsearch搜索的过程

1. 客户端发送请求到一个 coordinate node。
2. 协调节点将搜索请求转发到所有的shard 对应的 primary shard 或 replica shard，都可以。
3. query phase：每个shard 将自己的搜索结果（其实就是一些 doc id）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果。
4. fetch phase：接着由协调节点根据 doc id 去各个节点上拉取实际的 document 数据，最终返回给客户端。

写请求是写入primary shard，然后同步给所有的replica shard；读请求可以从primary shard 或replica shard 读取，采用的是随机轮询算法。

## Elasticsearch更新和删除文档的过程

如果是删除操作，commit 的时候会生成一个 .del 文件，里面将某个doc 标识为 deleted 状态，那么搜索的时候根据 **.del 文件**就知道这个doc 是否被删除了。

如果是更新操作，就是将原来的doc 标识为 deleted 状态，然后新写入一条数据。

buffer 每refresh 一次，就会产生一个 segment file，所以默认情况下是1 秒钟一个 segment file，这样下来 segment file 会越来越多，此时会定期执行merge。**每次merge 的时候，会将多个 segment file 合并成一个，同时这里会将标识为 deleted 的doc 给物理删除掉，然后将新的 segment file 写入磁盘，这里会写一个 commit point，标识所有新的 segment file，然后打开 segment file 供搜索使用，同时删除旧的 segment file。**

## Elasticsearch索引文档的过程

![](http://ww4.sinaimg.cn/large/006tNc79ly1g3ag42wf3tj30k00db3z2.jpg)

先写入内存buffer，在buffer 里的时候数据是搜索不到的；同时将数据写入translog 日志文件。

如果buffer 快满了，或者到一定时间，就会将内存buffer 数据 refresh 到一个新的 segment file 中，但是此时数据不是直接进入 segment file 磁盘文件，而是先进入 os cache 。这个过程就是 refresh。

每隔1 秒钟，es 将buffer 中的数据写入一个新的 segment file，每秒钟会产生一个新的磁盘文件 segment file，这个 segment file 中就存储最近1 秒内buffer 中写入的数据。

但是如果buffer 里面此时没有数据，那当然不会执行refresh 操作，如果buffer 里面有数据，默认1 秒钟执行一次refresh 操作，刷入一个新的segment file 中。

操作系统里面，磁盘文件其实都有一个东西，叫做 os cache，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 os cache，先进入操作系统级别的一个内存缓存中去。只要 buffer中的数据被refresh 操作刷入 os cache中，这个数据就可以被搜索到了。

为什么叫es 是准实时的？ NRT，全称 near real-time。默认是每隔1 秒refresh 一次的，所以es 是准实时的，因为写入的数据1 秒之后才能被看到。可以通过es 的 restful api 或者 java api，手动执行一次refresh 操作，就是手动将buffer 中的数据刷入 os cache中，让数据立马就可以被搜索到。只要数据被输入 os cache 中，buffer 就会被清空了，因为不需要保留buffer 了，数据在translog 里面已经持久化到磁盘去一份了。

重复上面的步骤，新的数据不断进入buffer 和translog，不断将 buffer 数据写入一个又一个新的 segment file 中去，每次 refresh 完buffer 清空，translog 保留。随着这个过程推进，translog 会变得越来越大。当translog 达到一定长度的时候，就会触发 commit 操作。

commit 操作发生第一步，就是将buffer 中现有数据 refresh 到 os cache 中去，清空buffer。然后，将一个 commit point写入磁盘文件，里面标识着这个 commit point 对应的所有 segment file，同时强行将 os cache 中目前所有的数据都 fsync 到磁盘文件中去。最后清空 现有translog 日志文件，重启一个translog，此时commit 操作完成。

这个commit 操作叫做 flush。默认30 分钟自动执行一次 flush，但如果translog 过大，也会触发 flush。flush 操作就对应着commit 的全过程，我们可以通过es api，手动执行flush 操作，手动将os cache 中的数据fsync 强刷到磁盘上去。

translog 日志文件的作用是什么？你执行commit 操作之前，数据要么是停留在buffer 中，要么是停留在os cache 中，无论是buffer 还是os cache 都是内存，一旦这台机器死了，内存中的数据就全丢了。所以需要将数据对应的操作写入一个专门的日志文件 translog 中，一旦此时机器宕机，再次重启的时候，es 会自动读取translog 日志文件中的数据，恢复到内存buffer 和os cache 中去。

translog 其实也是先写入os cache 的，默认每隔5 秒刷一次到磁盘中去，所以默认情况下，可能有5 秒的数据会仅仅停留在buffer 或者translog 文件的os cache 中，如果此时机器挂了，会丢失 5 秒钟的数据。但是这样性能比较好，最多丢5 秒的数据。也可以将translog 设置成每次写操作必须是直接 fsync 到磁盘，但是性能会差很多。

实际上你在这里，如果面试官没有问你es 丢数据的问题，你可以在这里给面试官炫一把，你说，其实es 第一是准实时的，数据写入1 秒后可以搜索到；可能会丢失数据的。有5 秒的数据，停留在buffer、translog os cache、segment file os cache 中，而不在磁盘上，此时如果宕机，会导致5 秒的数据丢失。

总结一下，数据先写入内存buffer，然后每隔1s，将数据refresh 到os cache，到了os cache 数据就能被搜索到（所以我们才说es 从写入到能被搜索到，中间有1s 的延迟）。每隔5s，将数据写入translog 文件（这样如果机器宕机，内存数据全没，最多会有5s 的数据丢失），translog 大到一定程度，或者默认每隔30mins，会触发commit 操作，将缓冲区的数据都flush 到segment file 磁盘文件中。

数据写入segment file 之后，同时就建立好了倒排索引。

# 参考
- [全文搜索引擎 Elasticsearch 入门教程](http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)
- [Elasticsearch: 权威指南](https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/index.html)