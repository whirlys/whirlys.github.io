---
title: Elasticsearch 分布式特性
categories:
  - 大数据
tags:
  - elasticsearch
keywords: elasticsearch,elasticsearch分布式特性
date: 2018-08-25 18:29:15
---

### 前言
本文的主要内容：

- 分布式介绍及cerebro
- 构建集群
- 副本与分片
- 集群状态与故障转移
- 文档分布式存储
- 脑裂问题
- shard详解


### 分布式介绍及cerebro

ES支持集群模式，是一个分布式系统，其好处主要有两个：
- 增大系统容量，如内存、磁盘，使得ES集群可以支持PB级的数据
- 提高系统可用性，即使部分节点停止服务，整个集群依然可以正常服务

ES集群由多个ES实例组成
- 不同集群通过集群名称来区分，可通过cluster.name进行修改，名称默认为elasticsearch
- 每个ES实例本质上是一个JVM进程，且有自己的名字，通过node.name进行修改

#### cerebro

cerebro 是一个ES Web管理工具，项目地址 https://github.com/lmenezes/cerebro 

其配置文件为 conf/application.conf，启动 cerebro ，默认监听的地址为 0.0.0.0:9000

```
bin/cerebro
# 也可指定监听ip和端口号
bin/cerebro -Dhttp.port=1234 -Dhttp.address=127.0.0.1
```

访问 http://yourhost:9000 ，填写要监控的 ES 地址：http://eshost:9200 即可进入管理界面

![cerebro管理界面](http://image.laijianfeng.org/20180825_150701.png)

![cerebro 节点信息](http://image.laijianfeng.org/20180825_151133.png)

![cerebro 集群配置](http://image.laijianfeng.org/20180825_151323.png)

在cerebro管理界面中我们可以看到 ES节点、索引、shard的分布、集群参数配置等多种信息



### 构建集群

如果只有一台机器，可以执行下面的命令，每次指定相同的集群名称，不同的节点名称和端口，即可在同一台机器上启动多个ES节点

```
bin/elasticsearch -Ecluster.name=my_cluster -Enode.name=node1 -Ehttp.port=9200 -d
```

作者的是在 virtualbox 上安装Ubuntu虚拟机，在安装好开发环境，正常启动ES之后，采取复制虚拟机的做法，复制后需要修改虚拟机的UUID，做法可自行上网搜索。

作者复制了两个，准备构建一个拥有三个ES节点的集群。启动虚拟机后可以进行关闭防火墙，配置hosts以使相互之间能够通过主机名访问，配置ssh免密访问等操作

分别修改ES节点中的 `cluster.name` 为相同名称，`node.name` 为各自的主机名，`network.host` 为 `0.0.0.0`，`discovery.zen.ping.unicast.hosts` 列表中中加入各自的 `node.name`

在ES主目录下执行命令启动ES

```
bin/elasticsearch
```

查看日志可见集群搭建完毕


#### Cluster State 集群状态

与ES集群相关的数据称为cluster state，主要记录如下信息：
- 节点信息，比如节点名称、连接地址等
- 索引信息，比如索引名称，配置等
- 其他。。

#### Master Node 主节点
- 可以修改cluster state的节点成为master节点，一个集群**只能有一个**
- cluster state存储在每个节点上，master维护最新版本并**同步**给其他节点
- master节点是通过集群中所有节点**选举产生**的，可以**被选举**的节点成为master-eligible（候选）节点，相关配置如下：`node.master: true`

#### Coordinating Node
- **处理请求**的节点即为coordinating节点，该节点为所有节点的默认角色，不能取消
- 路由请求到正确的节点处理，比如创建索引的请求到master节点

#### Data Node 数据节点
- **存储数据**的节点即为Data节点，默认节点都是data类型，相关配置如下：`node.data: true`


### 副本与分片
#### 提高系统可用性
提高系统可用性可从两个方面考虑：服务可用性和数据可用性

服务可用性：
- 2个节点的情况下，允许其中1个节点停止服务

数据可用性
- 引入副本（Replication）解决
- 每个节点上都有完备的数据



#### 增大系统容量

如何将数据分布于所有节点上？
- 引入分片（shard）解决问题

分片是ES支持PB级数据的基石
- 分片存储了部分数据，可以分布于任意节点上
- 分片数在索引创建时指定且后续不允许再修改，默认为5个
- 分片有主分片和副本分片之分，以实现数据的高可用
- 副本分片的数据由主分片同步，可以有多个，从而提高读取的吞吐量


#### 分片的分布

下图演示的是 3 个节点的集群中test\_index的分片分布情况，创建时我们指定了3个分片和副本

```
PUT test_index
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 3
  }
}
```

![主副分片的分布](http://image.laijianfeng.org/20180825_164121.png)

大致是均匀分布，实验中如果由于磁盘空间不足导致有分片未分配，为了测试可以将集群设置 `cluster.routing.allocation.disk.threshold_enabled` 设置为 false


- **此时增加节点是否能提高索引的数据容量？**

不能，因为已经设置了分片数为 3 ，shard的数量已经确定，新增的节点无法利用，



- **此时增加副本数能否提高索引的读取吞吐量？**

不能，因为新增的副本分片也是分布在这 3 台节点上，利用了同样的资源（CPU，内存，IO等）。如果要增加吞吐量，同时还需要增加节点的数量


- **分片数的设定很重要，需要提前规划好**
    - 过小会导致后续无法通过增加节点实现水平扩容
    - 过大会导致一个节点上分布过多分片，造成资源浪费，同时会影响查询性能
    - shard的数量的确定：一般建议一个shard的数据量不要超过 `30G`，shard数量最小为 2



### Cluster Health 集群健康

通过如下API可以查看集群健康状况，状态status包括以下三种：

- green 健康状态，指所有主副分片都正常分配
- yellow 指所有主分片都正常分配，但有副本分片未正常分配
- red 有主分片未分配



```
GET _cluster/health

# 结果
{
  "cluster_name": "elasticsearch",
  "status": "yellow",
  "timed_out": false,
  "number_of_nodes": 1,
  "number_of_data_nodes": 1,
  "active_primary_shards": 115,
  "active_shards": 115,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 111,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 50.88495575221239
}
```


#### Failover 故障转移

集群由 3 个节点组成，名称分别为 master，Hadoop2，Hadoop3， 其中 master 为主节点，集群状态status为 green

![集群状态green](http://image.laijianfeng.org/20180825_174406.png)

**如果此时 master 所在机器宕机导致服务终止，此时集群如何处理？**

Hadoop2 和 Hadoop3 发现 master 无法响应一段时间后会发起 master 主节点选举，比如这里选择 Hadoop2 为 master 节点。由于此时主分片 P0 和 P2 下线，集群状态变为 Red

![节点master宕机](http://image.laijianfeng.org/20180825_174933.png)

node2 发现主分片 P0 和 P2 未分配，将 R0 和 R2 提升为主分片，此时由于所有主分片都正常分配，集群状态变为 yellow

![image](http://image.laijianfeng.org/20180825_175028.png)


Hadoop2 为 P0 和 P2 生成新的副本，集群状态变为绿色

![image](http://image.laijianfeng.org/20180825_175235.png)

最后看看 Hadoop2 打印的日志

![image](http://image.laijianfeng.org/20180825_175517.png)

### 文档分布式存储

文档最终会存储在分片上。文档选择分片需要文档到分片的**映射算法**，目的是使得文档均匀分布在所有分片上，以充分利用资源。

算法：
- 随机选择或者round-robin算法？不可取，因为需要维护文档到分片的映射关系，成本巨大
- **根据文档值实时计算对应的分片**

#### 文档到分片的映射算法

ES通过如下的公式计算文档对应的分片
- `shard = hash(routing) % number_of_primary_shards`
- hash算法保证可以将数据均匀地分散在分片中
- routing是一个关键参数，默认是文档id，也可以自行指定
- number\_of\_primary\_shards是主分片数

该算法与主分片数相关，这也是分片数一旦确定后便不能更改的原因


#### 文档创建流程
1. Client向node3发起创建文档的请求
2. node3通过routing计算该文档应该存储在shard1上，查询cluster state后确认主分片P1在node2上，然后转发创建文档的请求到node2
3. P1 接收并执行创建文档请求后，将同样的请求发送到副本分片R1
4. R1接收并执行创建文档请求后，通知P1成功的结果
5. P1接收副本分片结果后，通知node3创建成功
6. node3返回结果到Client

![文档创建流程](http://image.laijianfeng.org/20180825_181026.png)

#### 文档读取流程

1. Client向node3发起获取文档1的请求
2. node3通过routing计算该文档在shard1上，查询cluster state后获取shard1的主副分片列表，然后以轮询的机制获取一个shard，比如这里是R1，然后转发读取文档的请求到node1
3. R1接收并执行读取文档请求后，将结果返回node3
4. node3返回结果给client

![文档读取流程](http://image.laijianfeng.org/20180825_181511.png)

#### 文档批量创建的流程
1. client向node3发起批量创建文档的请求（bulk）
2. node3通过routing计算所有文档对应的shard，然后按照主shard分配对应执行的操作，同时发送请求到涉及的主shard，比如这里3个主shard都需要参与
3. 主shard接收并执行请求后，将同样的请求同步到对应的副本shard
4. 副本shard执行结果后返回到主shard，主shard再返回node3
5. node3整合结果后返回client

![文档批量创建的流程 bulk](http://image.laijianfeng.org/20180825_181725.png)

#### 文档批量读取的流程
1. client向node3发起批量获取所有文档的请求（mget）
2. node3通过routing计算所有文档对应的shard，然后通过轮询的机制获取要参与shard，按照shard投建mget请求，通过发送请求到涉及shard，比如这里有2个shard需要参与
3. R1，R2返回文档结果
4. node3返回结果给client

![文档批量读取的流程 mget](http://image.laijianfeng.org/20180825_182023.png)

### 脑裂问题
脑裂问题，英文为split-brain，是分布式系统中的经典网络问题，如下图所示：

3个节点组成的集群，突然node1的网络和其他两个节点中断
![image](http://image.laijianfeng.org/20180807_1234.png)

node2与node3会重新选举master，比如node2成为了新的master，此时会更新cluster state

node1自己组成集群后，也更新cluster state

同一个集群有两个master，而且维护不同的cluster state，网络恢复后无法选择正确的master

![image](http://image.laijianfeng.org/20180807_1235.png)

**解决方案**为仅在可选举master-eligible节点数大于等于quorum时才可以进行master选举

- `quorum = master-eligible节点数/2 + 1`，例如3个master-eligible节点时，quorum 为2 
- 设定 `discovery.zen.minimun_master_nodes` 为 `quorum` 即可避免脑裂问题

![image](http://image.laijianfeng.org/20180807_1236.png)


### 倒排索引的不可变更
倒排索引一旦生成，不能更改
其好处如下：
- 不用考虑并发写文件的问题，杜绝了锁机制带来的性能问题
- 由于文件不再更改，可以充分利用文件系统缓存，只需载入一次，只要内存足够，对该文件的读取都会从内存读取，性能高
- 利于生成缓存数据
- 利于对文件进行压缩存储，节省磁盘和内存存储空间

坏处为需要写入新文档时，必须重新构建倒排索引文件，然后替换老文件后，新文档才能被检索，导致文档实时性差


### 文档搜索实时性

**解决方案**是新文档直接生成新的倒排索引文件，查询的时候同时查询所有的倒排文件，然后做结果的汇总计算即可


Lucene便是采用了这种方案，它构建的单个倒排索引称为segment，合在一起称为index，与ES中的Index概念不同，ES中的一个shard对应一个Lucene Index

Lucene会有一个专门的文件来记录所有的segment信息，称为commit point
![image](http://image.laijianfeng.org/20180807_1237.png)


#### refresh
segment写入磁盘的过程依然很耗时，可以借助文件系统缓存的特性，现将segment在缓存中创建并开放查询来进一步提升实时性，该过程在ES中被称为`refresh`

在refresh之前文档会先存储在一个buffer中，refresh时将buffer中的所有文档清空并生成segment

ES默认每1秒执行一次refresh，因此文档的实时性被提高到1秒，这也是ES被称为 近实时(Near Real Time)的原因
![image](http://image.laijianfeng.org/20180807_1238.png)


#### translog

如果在内存中的segment还没有写入磁盘前发生了宕机，那么其中的文档就无法恢复了，如何解决这个问题呢？
- ES引入translog机制，写入文档到buffer时，同时将该操作写入translog
- translog文件会即时写入磁盘(fsync)，6.x默认每个请求都会落盘

![image](http://image.laijianfeng.org/20180807_1345.png)

#### flush
flush负责将内存中的segment写入磁盘，主要做成如下的工作：
- 将translog写入磁盘
- 将index buffer清空，其中的文档生成一个新的segment，相当于一个refresh操作
- 更新commit point并写入磁盘
- 执行fsync操作，将内存中的segment写入磁盘
- 删除旧的translog文件

![image](http://image.laijianfeng.org/20180807_1239.png)

flush发生的时机主要有如下几种情况：
- 间隔时间达到时，默认是30分钟，5.x之前可以通过`index.translog.flush_threshold_period`修改，之后无法修改
- translog占满时，其大小可以通过`index.translog.flush_threshold_size`控制，默认是512mb，每个index有自己的translog

#### refresh
refresh发生的时机主要有如下几种情况：
- 间隔时间达到时，通过`index.settings.refresh_interval`来设定，默认是1秒
- `index.buffer`占满时，其大小通过`indices.memory.index_buffer_size`设置，默认为JVM heap的10%，所有shard共享
- flush发生时也会发生refresh


#### 删除与更新文档
segment一旦生成就不能更改，那么如果你要删除文档该如何操作？
- Lucene专门维护一个`.del`文件，记录所有已经删除的文档，注意`.del`上记录的是文档在Lucene内部的id
- 在查询结果返回前会过滤掉`.del`中所有的文档

要更新文档如何进行呢？
- 首先删除文档，然后再创建新文档

#### 整体视角
ES Index与Lucene Index的术语对照如下所示：
![image](http://image.laijianfeng.org/20180807_1346.png)


#### Segment Merging
随着segment的增多，由于一次查询的segment数增多，查询速度会变慢
ES会定时在后台进行segment merge的操作，减少segment的数量
通过force\_merge api可以手动强制做segment merge的操作



-----

打开微信扫一扫，关注【小旋锋】微信公众号，及时接收博文推送

![小旋锋的微信公众号](http://image.laijianfeng.org/%E5%B0%8F%E6%97%8B%E9%94%8B%E7%9A%84%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%8F%B7_%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)