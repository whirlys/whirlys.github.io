---
title: 利用Zookeeper实现 - Master选举
categories:
  - 大数据
tags:
  - Zookeeper
keywords: Zookeeper
date: 2019-01-23 00:07:38
---

Zookeeper 是一个高可用的分布式数据管理与协调框架，基于ZAB协议算法的实现，该框架能够很好的保证分布式环境中数据的一致性。Zookeeper的典型应用场景主要有：数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等。


本文主要介绍如何利用Zookeeper实现Master选举。

### Master选举

Master选举在分布式系统中是一个非常常见的场景。在分布式系统中，常常采用主从模式的方式避免单点故障，提高系统服务的可用性。正常情况下，Master节点用来协调集群中其他系统单元，维护系统状态信息，或者负责一些复杂的逻辑，再将处理结果同步给其他节点。当Master节点宕机，或者由于其他问题导致无法提供服务时，系统将发起一次Master选举，从候选节点中选出一个新的Master节点，以继续提供服务。

譬如在一些读写分离的应用中，Master节点负责客户端的写请求，处理完毕之后再将结果同步给从节点。

#### 选举算法？

著名的选举算法有 Paxos算法、Raft算法、Bully算法等，但在业务系统的开发中，实现选举算法并不是我们工作的重心。

Zookeeper有一个非常重要的特性即**强一致性**，能够很好地保证在分布式高并发情况下**节点的创建一定能够保证全局唯一性**，即Zookeeper将会保证客户端无法重复创建一个已经存在的数据节点。也就是说，如果同时有多个客户端请求创建同一个节点，那么最终一定只有一个客户端请求能够创建成功。利用这个特性，就能很容易地在分布式环境中进行Master选举了。

### 利用Zookeeper实现Master选举

Apache Curator是一个Zookeeper的开源客户端，它提供了Zookeeper各种应用场景（Recipe，如共享锁服务、master选举、分布式计数器等）的抽象封装，本文使用 Curator 提供的Recipe来实现Master选举。

Curator提供了两种选举方案：Leader Latch 和 Leader Election。下面分别介绍这两种选举方案。

#### Leader Latch

使用 Leader Latch 方案进行Master选举，系统将**随机从候选者中选出一台作为 `leader`，直到调用 `close()` 释放leadship，此时再重新随机选举 `leader`，否则其他的候选者无法成为 `leader`。**

下面的程序将启动 N 个线程用来模拟分布式系统中的节点，每个线程将创建一个Zookeeper客户端和一个 LeaderLatch 对象用于选举；每个线程有一个名称，名称中有一个编号用于区分；每个线程的存活时间为 `number * 10秒` ，存活时间结束后将关闭 LeaderLatch 对象和客户端，表示该 '节点' 宕机，如果该节点为 Master节点，这时系统将重新发起 Master选举。

```java
public class LeaderLatchTest {
    private static final String zkServerIps = "master:2181,hadoop2:2181";
    private static final String masterPath = "/testZK/leader_latch";

    public static void main(String[] args) {
        final int clientNums = 5;  // 客户端数量，用于模拟
        final CountDownLatch countDownLatch = new CountDownLatch(clientNums);
        List<LeaderLatch> latchList = new CopyOnWriteArrayList();
        List<CuratorFramework> clientList = new CopyOnWriteArrayList();
        AtomicInteger atomicInteger = new AtomicInteger(1);
        try {
            for (int i = 0; i < clientNums; i++) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        CuratorFramework client = getClient();  // 创建客户端
                        clientList.add(client);
                        int number = atomicInteger.getAndIncrement();
                        final LeaderLatch latch = new LeaderLatch(client, masterPath, "client#" + number);
                        System.out.println("创建客户端：" + latch.getId());
                        // LeaderLatch 添加监听事件
                        latch.addListener(new LeaderLatchListener() {
                            @Override
                            public void isLeader() {
                                System.out.println(latch.getId() + ": 我现在被选举为Leader！我开始工作了....");
                            }
                            @Override
                            public void notLeader() {
                                System.out.println(latch.getId() + ": 我遗憾地落选了，我到一旁休息去吧...");
                            }
                        });
                        latchList.add(latch);
                        try {
                            latch.start();
                            // 随机等待 number * 10秒，之后关闭客户端
                            Thread.sleep(number * 10000);
                        } catch (Exception e) {
                            System.out.println(e.getMessage());
                        } finally {
                            System.out.println("客户端 " + latch.getId() + " 关闭");
                            CloseableUtils.closeQuietly(latch);
                            CloseableUtils.closeQuietly(client);
                            countDownLatch.countDown();
                        }
                    }
                }).start();
            }
            countDownLatch.await(); // 等待，只有所有线程都退出
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static synchronized CuratorFramework getClient() {
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString(zkServerIps)
                .sessionTimeoutMs(6000).connectionTimeoutMs(3000) //.namespace("LeaderLatchTest")
                .retryPolicy(new ExponentialBackoffRetry(1000, 3)).build();
        client.start();
        return client;
    }
}
```

控制台输出的日志

```
创建客户端：client#1
创建客户端：client#2
创建客户端：client#3
创建客户端：client#4
创建客户端：client#5
client#2: 我现在被选举为Leader！我开始工作了....
客户端 client#1 关闭
客户端 client#2 关闭
client#4: 我现在被选举为Leader！我开始工作了....
客户端 client#3 关闭
客户端 client#4 关闭
client#5: 我现在被选举为Leader！我开始工作了....
客户端 client#5 关闭
```

系统运行过程中查看 masterPath 可以看见客户端注册的临时节点，当客户端关闭时，临时节点也会被删除

![LeaderLatch选举时的ZK节点](http://image.laijianfeng.org/20190122_233414.png)

#### Leader Election

通过 Leader Election 选举方案进行 Master选举，需添加 LeaderSelectorListener 监听器对领导权进行控制，**当节点被选为leader之后，将调用 `takeLeadership` 方法进行业务逻辑处理，处理完成会立即释放 leadship，重新进行Master选举**，这样每个节点都有可能成为 leader。`autoRequeue()` 方法的调用确保此实例在释放领导权后还可能获得领导权。


```java
public class LeaderSelectorTest {
    private static final String zkServerIps = "master:2181,hadoop2:2181";
    private static final String masterPath = "/testZK/leader_selector";

    public static void main(String[] args) {
        final int clientNums = 5;  // 客户端数量，用于模拟
        final CountDownLatch countDownLatch = new CountDownLatch(clientNums);
        List<LeaderSelector> selectorList = new CopyOnWriteArrayList();
        List<CuratorFramework> clientList = new CopyOnWriteArrayList();
        AtomicInteger atomicInteger = new AtomicInteger(1);
        try {
            for (int i = 0; i < clientNums; i++) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        CuratorFramework client = getClient();  // 创建客户端
                        clientList.add(client);
                        int number = atomicInteger.getAndIncrement();
                        final String name = "client#" + number;
                        final LeaderSelector selector = new LeaderSelector(client, masterPath, new LeaderSelectorListener() {
                            @Override
                            public void takeLeadership(CuratorFramework client) throws Exception {
                                System.out.println(name + ": 我现在被选举为Leader！我开始工作了....");
                                Thread.sleep(3000);
                            }
                            @Override
                            public void stateChanged(CuratorFramework curatorFramework, ConnectionState connectionState) {
                            }
                        });
                        System.out.println("创建客户端：" + name);
                        try {
                            selector.autoRequeue();
                            selector.start();
                            selectorList.add(selector);
                            // 随机等待 number * 10秒，之后关闭客户端
                            Thread.sleep(number * 10000);
                        } catch (Exception e) {
                            System.out.println(e.getMessage());
                        } finally {
                            countDownLatch.countDown();
                            System.out.println("客户端 " + name + " 关闭");
                            CloseableUtils.closeQuietly(selector);
                            if (!client.getState().equals(CuratorFrameworkState.STOPPED)) {
                                CloseableUtils.closeQuietly(client);
                            }
                        }
                    }
                }).start();
            }
            countDownLatch.await(); // 等待，只有所有线程都退出
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static synchronized CuratorFramework getClient() {
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString(zkServerIps)
                .sessionTimeoutMs(6000).connectionTimeoutMs(3000) //.namespace("LeaderLatchTest")
                .retryPolicy(new ExponentialBackoffRetry(1000, 3)).build();
        client.start();
        return client;
    }
}
```

控制台输出的日志信息

```
创建客户端：client#2
创建客户端：client#1
创建客户端：client#3
创建客户端：client#5
创建客户端：client#4
client#5: 我现在被选举为Leader！我开始工作了....
client#3: 我现在被选举为Leader！我开始工作了....
client#2: 我现在被选举为Leader！我开始工作了....
client#4: 我现在被选举为Leader！我开始工作了....
客户端 client#1 关闭
client#5: 我现在被选举为Leader！我开始工作了....
client#3: 我现在被选举为Leader！我开始工作了....
client#2: 我现在被选举为Leader！我开始工作了....
客户端 client#2 关闭
client#4: 我现在被选举为Leader！我开始工作了....
client#5: 我现在被选举为Leader！我开始工作了....
client#3: 我现在被选举为Leader！我开始工作了....
client#4: 我现在被选举为Leader！我开始工作了....
客户端 client#3 关闭
client#5: 我现在被选举为Leader！我开始工作了....
client#4: 我现在被选举为Leader！我开始工作了....
client#5: 我现在被选举为Leader！我开始工作了....
客户端 client#4 关闭
client#5: 我现在被选举为Leader！我开始工作了....
client#5: 我现在被选举为Leader！我开始工作了....
client#5: 我现在被选举为Leader！我开始工作了....
客户端 client#5 关闭

```

LeaderSelectorListener类继承了ConnectionStateListener。一旦LeaderSelector启动，它会向curator客户端添加监听器。使用LeaderSelector必须时刻注意连接的变化。一旦出现连接问题如 `SUSPENDED`，curator实例必须确保它不再是leader，直至它重新收到 `RECONNECTED`。如果 `LOST` 出现，curator实例不再是 leader 并且其 `takeLeadership()` 应该直接退出。

推荐的做法是，如果发生 `SUSPENDED` 或者 `LOST` 连接问题，最好直接抛CancelLeadershipException，此时，leaderSelector实例会尝试中断并且取消正在执行 `takeLeadership()` 方法的线程。 建议扩展LeaderSelectorListenerAdapter，LeaderSelectorListenerAdapter中已经提供了推荐的处理方式 。


> 参考：  
> 《从Paxos到Zookeeper分布式一致性原理与实践》     
> [Zookeeper开源客户端Curator之Master/Leader选举](https://blog.csdn.net/wo541075754/article/details/70216046)

![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20190116_014816.png)

