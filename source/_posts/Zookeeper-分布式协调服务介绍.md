---
title: Zookeeper 分布式协调服务介绍
categories:
  - 大数据
tags:
  - Zookeeper
keywords: Zookeeper
date: 2019-01-21 21:24:09
---

#### 分布式系统

分布式系统的简单定义：分布式系统是一个硬件或软件组件分布在不同的网络计算机上，彼此之间仅仅通过消息传递进行通信和协调的系统。


分布式系统的特征：

- 分布性：系统中的计算机在空间上随意分布和随时变动
- 对等性：系统中的计算机是对等的，没有主从之分
- 并发性：并发性操作是非常常见的行为
- 缺乏全局时钟：系统中的计算机具有明显的分布性，且缺乏一个全局的时钟序列控制，所以很难比较两个事件的先后
- 故障总是会发生：任何在设计阶段考虑到的异常情况，一定会在系统实际运行中发生，并且还会遇到很多在设计时未考虑到的异常故障

随着分布式架构的出现，越来越多的分布式应用会面临数据一致性问题。

#### 选择Zookeeper

Zookeeper是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于它实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、master选举、分布式锁和分布式队列等功能。

Zookeeper致力于提供一个高性能、高可用，具有严格的顺序访问控制能力的分布式协调服务；其主要的设计目标是简单的数据模型、可以构建集群、顺序访问、高性能。Zookeeper已经成为很多大型分布式项目譬如Hadoop、HBase、Storm、Solr等中的核心组件，用于分布式协调。

Zookeeper可以保证如下**分布式一致性特性**：

- 顺序一致性：从同一个客户端发起的事务请求，最终将会严格地按照其发起顺序被应用到Zookeeper中去
- 原子性：所有事务请求的处理结果在整个集群中所有的机器上的应用情况是一致的
- 单一视图：无论客户端连接的是哪个Zookeeper服务器，其看到的服务器数据模型都是一致的
- 可靠性：一旦服务端成功地应用了一个事务，并完成对客户端的响应，那么该事务所引起的服务端状态变更将会被一直保留下来，除非有另一个事务又对其进行了变更
- 实时性：在一定的时间内，客户端最终一定能够从服务端上读取到最新的数据状态

![Zookeeper服务架构图](http://image.laijianfeng.org/zkservice.jpg)

### Zookeeper 基本概念

- 集群角色
    - Leader：客户端提供读和写服务
    - Follower：提供读服务，所有写服务都需要转交给Leader角色，参与选举
    - Observer：提供读服务，不参与选举过程，一般是为了增强Zookeeper集群的读请求并发能力

- 会话 (session)

    - Zk的客户端与zk的服务端之间的连接
    - 通过心跳检测保持客户端连接的存活
    - 接收来自服务端的watch事件通知
    - 可以设置超时时间


#### ZNode 节点

ZNode 是Zookeeper中数据的最小单元，每个ZNode上可以保存数据(byte[]类型)，同时可以挂在子节点，因此构成了一个层次化的命名空间，我们称之为树

![Zookeeper数据模型](http://image.laijianfeng.org/20190121_141804.jpg)


- 节点是有生命周期的，生命周期由**节点类型**决定：
    - 持久节点(PERSISTENT)：节点创建后就一直存在于Zookeeper服务器上，直到有删除操作主动将其删除
    - 持久顺序节点(PERSISTENT_SEQUENTIAL)：基本特性与持久节点一致，额外的特性在于Zookeeper会记录其子节点创建的先后顺序
    - 临时节点(EPHEMERAL)：声明周期与客户端的会话绑定，客户端会话失效时节点将被自动清除
    - 临时顺序节点(EPHEMERAL_SEQUENTIAL)：基本特性与临时节点一致，但添加了顺序的特性

- 每个节点都有**状态信息**，抽象为 Stat 对象，状态属性如下：
    - czxid：节点被创建时的事务ID
    - mzxid：节点最后一个被更新时的事务ID
    - ctime：节点创建时间
    - mtime：节点最后一个被更新时间
    - version：节点版本号
    - cversion：子节点版本号
    - aversion：节点的ACL版本号
    - ephemeralOwner：创建该临时节点的会话的sessionID，若为持久节点则为0
    - dataLength：数据内容长度
    - numChildren：子节点数量

- 权限控制ACL (Access Control Lists) 
    - CREATE：创建子节点的权限
    - READ：获取节点数据和子节点列表的权限
    - WRITE：更新节点数据的权限
    - DELETE：删除子节点的权限
    - ADMIN：设置节点ACL的权限

#### watcher机制

Zookeeper 引入watcher机制来实现发布/订阅功能，能够让多个订阅者同时监听某一个节点对象，当这个节点对象状态发生变化时，会通知所有订阅者。

Zookeeper的watcher机制主要包括客户端线程、客户端WatchManager、Zookeeper服务器三个部分。其工作流程简单来说：客户端在向Zookeeper服务器注册Watcher的同时，会将Watcher对象存储在客户端的WatchManager中；当Zookeeper服务器端触发Watcher事件后，会向客户端发送通知，客户端线程从WatchManager中取出对应的Watcher对象来执行回调逻辑

![Zookeeper Watcher机制概述](http://image.laijianfeng.org/20190121_174457.png)

可以设置的两种 Watcher

- NodeCache
    - 监听数据节点的内容变更
    - 监听节点的创建，即如果指定的节点不存在，则节点创建后，会触发这个监听

- PathChildrenCache
    - 监听指定节点的子节点变化情况
    - 包括新增子节点、子节点数据变更和子节点删除

### 客户端命令

Zookeeper的安装可参考官方文档

```shell
# 启动客户端，默认为 localhost:2181
bin/zkCli.sh     
# 启动客户端，指定连接的Zookeeper地址
bin/zkCli.sh -server ip:port 
# create 创建一个节点，路径为 /test，内容为 some test data 
create /test "some test data"
# ls 列出指定节点下的所有子节点 
ls /
# get 获取指定节点的数据内容和属性
get /test
# set 更新指定节点的数据内容
set /test "new test data"
# delete 删除节点
delete /test
# rmr 删除非空节点
rmr /test
# stat 输出节点的状态信息
stat /test
```

#### 四字命令
四字命令可以查看Zookeeper服务器的一些信息，可以通过 telnet 和 nc 等方式执行四字命令，以执行 conf 命令为例

```
# telnet 方式 执行Zookeeper的 conf 命令
telnet localhost 2181
conf
# nc 方式 执行Zookeeper的 conf 命令
echo conf | nc localhost 2181
```

**四字命令介绍**：

- conf 命令用于输出Zookeeper服务器运行时的基本配置信息
- cons 命令用于输出这台服务器上所有客户端连接的详细信息
- crst 命令用于重置所有客户端的连接统计信息
- dump 命令用于输出当前集群的所有会话信息
- envi 命令用于输出Zookeeper所在服务器的运行时信息
- ruok 命令用于输出当前Zookeeper服务器是否正在运行
- stat 命令用于获取Zookeeper服务器的运行状态信息
- srvr 命令与stat命令功能一致，但仅输出服务器自身的信息
- srst 命令用于重置所有服务器的统计信息
- wchs 命令用于输出当前服务器上管理的 watcher 的概要信息
- wchc 命令用于输出当前服务器上管理的 watcher 的详细信息
- wchp 命令与wchc功能非常相似，但输出信息以节点路径为单位进行归组
- mntr 命令用于输出比stat命令更为详细的服务器统计信息

### Curator 客户端代码实例


Curator 是 Apache 基金会的顶级项目之一，解决了很多Zookeeper客户端非常底层的细节开发工作，包括会话超时重连、反复注册Watcher、NodeExistsException异常等，提供了一套易用性和可读性更强的Fluent风格的客户端API框架，除此之外，curator还提供了Zookeeper各种应用场景（Recipe，如共享锁服务、master选举、分布式计数器等）的抽象封装。

Zookeeper 的核心提交者 Patrick Hunt 对 Curator 的高度评价：

> Guava is to Java what Curator is to Zookeeper


#### 示例 - 增删查改

```java
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.imps.CuratorFrameworkState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.Stat;

public class CuratorCrud {
    // 集群模式则是多个ip
    private static final String zkServerIps = "master:2181,hadoop2:2181";

    public static void main(String[] args) throws Exception {
        // 创建节点
        String nodePath = "/testZK"; // 节点路径
        byte[] data = "this is a test data".getBytes(); // 节点数据
        byte[] newData = "new test data".getBytes(); // 节点数据

        // 设置重连策略ExponentialBackoffRetry, baseSleepTimeMs：初始sleep的时间,maxRetries：最大重试次数,maxSleepMs：最大重试时间
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(10000, 5);
        //（推荐）curator链接zookeeper的策略:RetryNTimes n：重试的次数 sleepMsBetweenRetries：每次重试间隔的时间
        // RetryPolicy retryPolicy = new RetryNTimes(3, 5000);
        // （不推荐） curator链接zookeeper的策略:RetryOneTime sleepMsBetweenRetry:每次重试间隔的时间,这个策略只会重试一次
        // RetryPolicy retryPolicy2 = new RetryOneTime(3000);
        // 永远重试，不推荐使用
        // RetryPolicy retryPolicy3 = new RetryForever(retryIntervalMs)
        // curator链接zookeeper的策略:RetryUntilElapsed maxElapsedTimeMs:最大重试时间 sleepMsBetweenRetries:每次重试间隔 重试时间超过maxElapsedTimeMs后，就不再重试
        // RetryPolicy retryPolicy4 = new RetryUntilElapsed(2000, 3000);

        // Curator客户端
        CuratorFramework client = null;
        // 实例化Curator客户端，Curator的编程风格可以让我们使用方法链的形式完成客户端的实例化
        client = CuratorFrameworkFactory.builder()  // 使用工厂类来建造客户端的实例对象
                .connectString(zkServerIps) // 放入zookeeper服务器ip
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)  // 设定会话时间以及重连策略
                // .namespace("testApp")    // 隔离的命名空间
                .build(); // 建立连接通道
        // 启动Curator客户端
        client.start();

        boolean isZkCuratorStarted = client.getState().equals(CuratorFrameworkState.STARTED);
        System.out.println("当前客户端的状态：" + (isZkCuratorStarted ? "连接中..." : "已关闭..."));
        try {
            // 检查节点是否存在
            Stat s = client.checkExists().forPath(nodePath);
            if (s == null) {
                System.out.println("节点不存在，创建节点");
                // 创建节点
                String result = client.create().creatingParentsIfNeeded()    // 创建父节点，也就是会递归创建
                        .withMode(CreateMode.PERSISTENT) // 节点类型
                        .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE) // 节点的ACL权限
                        .forPath(nodePath, data);
                System.out.println(result + "节点，创建成功...");
            } else {
                System.out.println("节点已存在，" + s);
            }
            getData(client, nodePath);  // 输出节点信息

            // 更新指定节点的数据
            int version = s == null ? 0 : s.getVersion();  // 版本不一致时的异常：KeeperErrorCode = BadVersion
            Stat resultStat = client.setData().withVersion(version)   // 指定数据版本
                    .forPath(nodePath, newData);    // 需要修改的节点路径以及新数据
            System.out.println("更新节点数据成功");
            getData(client, nodePath);  // 输出节点信息

            // 删除节点
            client.delete().guaranteed()    // 如果删除失败，那么在后端还是会继续删除，直到成功
                    .deletingChildrenIfNeeded() // 子节点也一并删除，也就是会递归删除
                    .withVersion(resultStat.getVersion())
                    .forPath(nodePath);
            System.out.println("删除节点：" + nodePath);
            Thread.sleep(1000);
        } finally {
            // 关闭客户端
            if (!client.getState().equals(CuratorFrameworkState.STOPPED)) {
                System.out.println("关闭客户端.....");
                client.close();
            }
            isZkCuratorStarted = client.getState().equals(CuratorFrameworkState.STARTED);
            System.out.println("当前客户端的状态：" + (isZkCuratorStarted ? "连接中..." : "已关闭..."));
        }
    }

    /**
     * 读取节点的数据
     */
    private static byte[] getData(CuratorFramework client, String nodePath) throws Exception {
        Stat stat = new Stat();
        byte[] nodeData = client.getData().storingStatIn(stat).forPath(nodePath);
        System.out.println("节点 " + nodePath + " 的数据为：" + new String(nodeData));
        System.out.println("该节点的数据版本号为：" + stat.getVersion() + "\n");
        return nodeData;
    }
}

// 输出
当前客户端的状态：连接中...
节点不存在，创建节点
/testZK节点，创建成功...
节点 /testZK 的数据为：this is a test data
该节点的数据版本号为：0
更新节点数据成功
节点 /testZK 的数据为：new test data
该节点的数据版本号为：1
删除节点：/testZK
关闭客户端.....
当前客户端的状态：已关闭...
```


#### 示例 - 异步接口

```java
public class CuratorBackGround {
    private static final String zkServerIps = "master:2181,hadoop2:2181";

    public static void main(String[] args) throws Exception {
        CountDownLatch samphore = new CountDownLatch(2);
        ExecutorService tp = Executors.newFixedThreadPool(2);   // 线程池
        String nodePath = "/testZK";
        byte[] data = "this is a test data".getBytes();
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(10000, 5);
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString(zkServerIps)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy).build();
        client.start();

        // 异步创建节点，传入 ExecutorService，这样比较复杂的就会放到线程池中执行
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL)
                .inBackground(new BackgroundCallback() {
                    @Override
                    public void processResult(CuratorFramework curatorFramework, CuratorEvent curatorEvent) throws Exception {
                        System.out.println("event[code: " + curatorEvent.getResultCode() + ", type: " + curatorEvent.getType() + "]");
                        System.out.println("当前线程：" + Thread.currentThread().getName());
                        samphore.countDown();
                    }
                }, tp).forPath(nodePath, data); // 此处传入 ExecutorService tp

        // 异步创建节点
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL)
                .inBackground(new BackgroundCallback() {
                    @Override
                    public void processResult(CuratorFramework curatorFramework, CuratorEvent curatorEvent) throws Exception {
                        System.out.println("event[code: " + curatorEvent.getResultCode() + ", type: " + curatorEvent.getType() + "]");
                        System.out.println("当前线程：" + Thread.currentThread().getName());
                        samphore.countDown();
                    }
                }).forPath(nodePath, data); // 此处没有传入 ExecutorService tp

        samphore.await();
        tp.shutdown();
    }
}

// 输出
event[code: -110, type: CREATE]
当前线程：main-EventThread
event[code: 0, type: CREATE]
当前线程：pool-1-thread-1
```

#### 示例 - watcher 事件监听 - NodeCache

```java
public class CuratorWatcher {
    private static final String zkServerIps = "master:2181,hadoop2:2181";

    public static void main(String[] args) throws Exception {
        final String nodePath = "/testZK";
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(10000, 5);
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString(zkServerIps)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy).build();
        try {
            client.start();
            client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(nodePath, "this is a test data".getBytes());

            final NodeCache cacheNode = new NodeCache(client, nodePath, false);
            cacheNode.start(true);  // true 表示启动时立即从Zookeeper上获取节点
            cacheNode.getListenable().addListener(new NodeCacheListener() {
                @Override
                public void nodeChanged() throws Exception {
                    System.out.println("节点数据更新，新的内容是： " + new String(cacheNode.getCurrentData().getData()));
                }
            });
            for (int i = 0; i < 5; i++) {
                client.setData().forPath(nodePath, ("new test data " + i).getBytes());
                Thread.sleep(1000);
            }
            Thread.sleep(10000); // 等待100秒，手动在 zkCli 客户端操作节点，触发事件
        } finally {
            client.delete().deletingChildrenIfNeeded().forPath(nodePath);
            client.close();
            System.out.println("客户端关闭......");
        }
    }
}
```

#### 示例 - watcher 事件监听 - PathChildrenCache

```
public class CuratorPCWatcher {
    private static final String zkServerIps = "master:2181,hadoop2:2181";

    public static void main(String[] args) throws Exception {
        final String nodePath = "/testZK";
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(10000, 5);
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString(zkServerIps)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy).build();
        client.start();
        try {
            // 为子节点添加watcher，PathChildrenCache: 监听数据节点的增删改，可以设置触发的事件
            final PathChildrenCache childrenCache = new PathChildrenCache(client, nodePath, true);

            /**
             * StartMode: 初始化方式
             *  - POST_INITIALIZED_EVENT：异步初始化，初始化之后会触发事件
             *  - NORMAL：异步初始化
             *  - BUILD_INITIAL_CACHE：同步初始化
             */
            childrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);

            // 列出子节点数据列表，需要使用BUILD_INITIAL_CACHE同步初始化模式才能获得，异步是获取不到的
            List<ChildData> childDataList = childrenCache.getCurrentData();
            System.out.println("当前节点的子节点详细数据列表：");
            for (ChildData childData : childDataList) {
                System.out.println("\t* 子节点路径：" + new String(childData.getPath()) + "，该节点的数据为：" + new String(childData.getData()));
            }

            // 添加事件监听器
            childrenCache.getListenable().addListener(new PathChildrenCacheListener() {
                @Override
                public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent event) throws Exception {
                    // 通过判断event type的方式来实现不同事件的触发
                    if (event.getType().equals(PathChildrenCacheEvent.Type.INITIALIZED)) {  // 子节点初始化时触发
                        System.out.println("子节点初始化成功");
                    } else if (event.getType().equals(PathChildrenCacheEvent.Type.CHILD_ADDED)) {  // 添加子节点时触发
                        System.out.print("子节点：" + event.getData().getPath() + " 添加成功，");
                        System.out.println("该子节点的数据为：" + new String(event.getData().getData()));
                    } else if (event.getType().equals(PathChildrenCacheEvent.Type.CHILD_REMOVED)) {  // 删除子节点时触发
                        System.out.println("子节点：" + event.getData().getPath() + " 删除成功");
                    } else if (event.getType().equals(PathChildrenCacheEvent.Type.CHILD_UPDATED)) {  // 修改子节点数据时触发
                        System.out.print("子节点：" + event.getData().getPath() + " 数据更新成功，");
                        System.out.println("子节点：" + event.getData().getPath() + " 新的数据为：" + new String(event.getData().getData()));
                    }
                }
            });
            Thread.sleep(100000); // sleep 100秒，在 zkCli.sh 操作子节点，注意查看控制台的输出
        } finally {
            client.close();
        }
    }
}

// 输出
当前节点的子节点详细数据列表：
	* 子节点路径：/testZK/node1，该节点的数据为：hello world
子节点：/testZK/node2 添加成功，该子节点的数据为：hello node2
子节点：/testZK/node2 数据更新成功，子节点：/testZK/node2 新的数据为：hello zookeeper
子节点：/testZK/node2 删除成功
```

### Zookeeper的典型应用场景

下一篇文章将使用 Curator 客户端来实现 Zookeeper 的典型应用场景的示例，这里简单概括一下Zookeeper的典型应用场景：

- 数据发布/订阅，即所谓的配置中心
- 负载均衡
- 命名服务
- 分布式协调/通知
- 集群管理
- master 选举
- 分布式锁
- 分布式队列


> 参考：   
> 《从Paxos到Zookeeper分布式一致性原理与实践》

![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20190116_014816.png)

