---
title: 利用Zookeeper实现 - 分布式锁
categories:
  - 大数据
tags:
  - Zookeeper
keywords: Zookeeper
date: 2019-01-28 11:03:44
---

在许多场景中，**数据一致性**是一个比较重要的话题，在单机环境中，我们可以通过Java提供的**并发API**来解决；而在分布式环境(会遇到网络故障、消息重复、消息丢失等各种问题)下要复杂得多，常见的解决方案是**分布式事务**、**分布式锁**等。

本文主要探讨如何利用Zookeeper来实现分布式锁。


### 关于分布式锁

分布式锁是控制分布式系统之间**同步访问共享资源**的一种方式。

在**实现**分布式锁的过程中需要注意的：

- 锁的可重入性(递归调用不应该被阻塞、避免死锁)
- 锁的超时(避免死锁、死循环等意外情况)
- 锁的阻塞(保证原子性等)
- 锁的特性支持(阻塞锁、可重入锁、公平锁、联锁、信号量、读写锁)

在**使用**分布式锁时需要注意：

- 分布式锁的开销(分布式锁一般能不用就不用，有些场景可以用乐观锁代替)
- 加锁的粒度(控制加锁的粒度，可以优化系统的性能)
- 加锁的方式

以下是几种常见的实现分布式锁的方案及其优缺点。

#### 基于数据库

**1. 基于数据库表**

最简单的方式可能就是直接创建一张锁表，当我们要锁住某个方法或资源时，我们就在该表中增加一条记录，想要释放锁的时候就删除这条记录。给某字段添加唯一性约束，如果有多个请求同时提交到数据库的话，**数据库会保证只有一个操作可以成功**，那么我们就可以认为操作成功的那个线程获得了该方法的锁，可以执行方法体内容。

会引入数据库单点、无失效时间、不阻塞、不可重入等问题。

**2. 基于数据库排他锁**

如果使用的是MySql的InnoDB引擎，在查询语句后面增加`for update`，数据库会在查询过程中(须通过唯一索引查询)给数据库表增加排他锁，我们可以认为获得排它锁的线程即可获得分布式锁，通过 connection.commit() 操作来释放锁。

会引入数据库单点、不可重入、无法保证一定使用行锁(部分情况下MySQL自动使用表锁而不是行锁)、排他锁长时间不提交导致占用数据库连接等问题。

**3. 数据库实现分布式锁总结**

优点：

- 直接借助数据库，容易理解。

缺点：

- 会引入更多的问题，使整个方案变得越来越复杂
- 操作数据库需要一定的开销，有一定的性能问题
- 使用数据库的行级锁并不一定靠谱，尤其是当我们的锁表并不大的时候

#### 基于缓存

相比较于基于数据库实现分布式锁的方案来说，基于缓存来实现在性能方面会表现的更好一点。目前有很多成熟的缓存产品，包括Redis、memcached、tair等。

这里以Redis为例举出几种实现方法：

**1. 基于 redis 的 setnx()、expire() 方法做分布式锁**

setnx 的含义就是 `SET if Not Exists`，其主要有两个参数 `setnx(key, value)`。该方法是原子的，如果 key 不存在，则设置当前 key 成功，返回 1；如果当前 key 已经存在，则设置当前 key 失败，返回 0。

expire 设置过期时间，要注意的是 setnx 命令不能设置 key 的超时时间，只能通过 expire() 来对 key 设置。


**2. 基于 redis 的 setnx()、get()、getset()方法做分布式锁**

getset 这个命令主要有两个参数 `getset(key，newValue)`，该方法是原子的，对 key 设置 newValue 这个值，并且返回 key 原来的旧值。

**3. 基于 Redlock 做分布式锁**

Redlock 是 Redis 的作者 antirez 给出的集群模式的 Redis 分布式锁，它基于 N 个完全独立的 Redis 节点（通常情况下 N 可以设置成 5）

**4. 基于 redisson 做分布式锁**

redisson 是 redis 官方的分布式锁组件，GitHub 地址：https://github.com/redisson/redisson

**基于缓存实现分布式锁总结**

优点：
- 性能好

缺点：

- 实现中需要考虑的因素太多
- 通过超时时间来控制锁的失效时间并不是十分的靠谱

#### 基于Zookeeper



**大致思想**为：每个客户端对某个方法加锁时，在 Zookeeper 上与该方法对应的指定节点的目录下，**生成一个唯一的临时有序节点**。 判断是否获取锁的方式很简单，只需要判断有序节点中**序号最小的一个**。 当释放锁的时候，只需将这个临时节点删除即可。同时，其可以避免服务宕机导致的锁无法释放，而产生的死锁问题

**Zookeeper实现分布式锁总结**

优点：

- 有效的解决单点问题，不可重入问题，非阻塞问题以及锁无法释放的问题
- 实现较为简单

缺点：

- 性能上不如使用缓存实现的分布式锁，因为每次在创建锁和释放锁的过程中，都要动态创建、销毁临时节点来实现锁功能
- 需要对Zookeeper的原理有所了解


### Zookeeper 如何实现分布式锁？

下面讲如何实现排他锁和共享锁，以及如何解决羊群效应。

#### 排他锁

排他锁，又称写锁或独占锁。如果事务T1对数据对象O1加上了排他锁，那么在整个加锁期间，只允许事务T1对O1进行读取或更新操作，其他任务事务都不能对这个数据对象进行任何操作，直到T1释放了排他锁。

排他锁核心是**保证当前有且仅有一个事务获得锁，并且锁释放之后，所有正在等待获取锁的事务都能够被通知到**。

Zookeeper 的强一致性特性，能够很好地保证在分布式高并发情况下**节点的创建一定能够保证全局唯一性**，即Zookeeper将会保证客户端无法重复创建一个已经存在的数据节点。可以利用Zookeeper这个特性，实现排他锁。

- **定义锁**：通过Zookeeper上的数据节点来表示一个锁
- **获取锁**：客户端通过调用 `create` 方法创建表示锁的临时节点，可以认为创建成功的客户端获得了锁，同时可以让没有获得锁的节点在该节点上注册Watcher监听，以便实时监听到lock节点的变更情况
- **释放锁**：以下两种情况都可以让锁释放
    - 当前获得锁的客户端发生宕机或异常，那么Zookeeper上这个临时节点就会被删除
    - 正常执行完业务逻辑，客户端主动删除自己创建的临时节点

基于Zookeeper实现排他锁流程：

![基于Zookeeper实现排他锁流程](http://image.laijianfeng.org/20190124_133414.jpg)


#### 共享锁

共享锁，又称读锁。如果事务T1对数据对象O1加上了共享锁，那么当前事务只能对O1进行**读取操作**，其他事务也只能对这个数据对象加共享锁，直到该数据对象上的所有共享锁都被释放。

共享锁与排他锁的区别在于，加了排他锁之后，数据对象只对当前事务可见，而加了共享锁之后，数据对象对所有事务都可见。

- **定义锁**：通过Zookeeper上的数据节点来表示一个锁，是一个类似于 `/lockpath/[hostname]-请求类型-序号` 的临时顺序节点
- **获取锁**：客户端通过调用 `create` 方法创建表示锁的临时顺序节点，如果是读请求，则创建 `/lockpath/[hostname]-R-序号` 节点，如果是写请求则创建 `/lockpath/[hostname]-W-序号` 节点
- **判断读写顺序**：大概分为4个步骤
    - 1)创建完节点后，获取 `/lockpath` 节点下的所有子节点，并对该节点注册子节点变更的Watcher监听
    - 2)确定自己的节点序号在所有子节点中的顺序
    - 3.1)对于读请求：1. 如果没有比自己序号更小的子节点，或者比自己序号小的子节点都是读请求，那么表明自己已经成功获取到了共享锁，同时开始执行读取逻辑 2. 如果有比自己序号小的子节点有写请求，那么等待 3. 
    - 3.2)对于写请求，如果自己不是序号最小的节点，那么等待
    - 4)接收到Watcher通知后，重复步骤1)
- **释放锁**：与排他锁逻辑一致

![Zookeeper实现共享锁节点树](http://image.laijianfeng.org/2019-01-28_20-08-52.jpg)

基于Zookeeper实现共享锁流程：

![基于Zookeeper实现共享锁流程](http://image.laijianfeng.org/2019-01-28_20-08-50.jpg)

#### 羊群效应

在实现共享锁的 "判断读写顺序" 的第1个步骤是：创建完节点后，获取 `/lockpath` 节点下的所有子节点，并对该节点注册子节点变更的Watcher监听。这样的话，任何一次客户端移除共享锁之后，Zookeeper将会发送子节点变更的Watcher通知给所有机器，系统中将有大量的 "Watcher通知" 和 "子节点列表获取" 这个操作重复执行，然后所有节点再判断自己是否是序号最小的节点(写请求)或者判断比自己序号小的子节点是否都是读请求(读请求)，从而继续等待下一次通知。

然而，这些重复操作很多都是 "无用的"，实际上**每个锁竞争者只需要关注序号比自己小的那个节点是否存在即可**

当集群规模比较大时，这些 "无用的" 操作不仅会对Zookeeper造成巨大的性能影响和网络冲击，更为严重的是，如果同一时间有多个客户端释放了共享锁，Zookeeper服务器就会在短时间内向其余客户端发送大量的事件通知--这就是所谓的 "**羊群效应**"。

**改进后的分布式锁实现**：

具体实现如下：

- 1. 客户端调用 `create` 方法创建一个类似于 `/lockpath/[hostname]-请求类型-序号` 的临时顺序节点
- 2. 客户端调用 `getChildren` 方法获取所有已经创建的子节点列表(这里不注册任何Watcher)
- 3. 如果无法获取任何共享锁，那么调用 `exist` 来对比自己小的那个节点注册Watcher
    - 读请求：向比自己序号小的最后一个**写请求节点**注册Watcher监听
    - 写请求：向比自己序号小的最后一个**节点**注册Watcher监听
- 4. 等待Watcher监听，继续进入步骤2

Zookeeper羊群效应改进前后Watcher监听图

![Zookeeper羊群效应改进前后](http://image.laijianfeng.org/2019-01-28_20-08-53.jpg)

### 基于Curator客户端实现分布式锁

Apache Curator是一个Zookeeper的开源客户端，它提供了Zookeeper各种应用场景（Recipe，如共享锁服务、master选举、分布式计数器等）的抽象封装，接下来将利用Curator提供的类来实现分布式锁。

Curator提供的跟分布式锁相关的类有5个，分别是：

- Shared Reentrant Lock 可重入锁
- Shared Lock 共享不可重入锁
- Shared Reentrant Read Write Lock 可重入读写锁
- Shared Semaphore 信号量
- Multi Shared Lock 多锁

> 关于错误处理：还是强烈推荐使用ConnectionStateListener处理连接状态的改变。当连接LOST时你不再拥有锁。

#### 可重入锁

Shared Reentrant Lock，全局可重入锁，所有客户端都可以请求，同一个客户端在拥有锁的同时，可以多次获取，不会被阻塞。它是由类 `InterProcessMutex` 来实现，它的主要方法：

```java
// 构造方法
public InterProcessMutex(CuratorFramework client, String path)
public InterProcessMutex(CuratorFramework client, String path, LockInternalsDriver driver)
// 通过acquire获得锁,并提供超时机制：
public void acquire() throws Exception
public boolean acquire(long time, TimeUnit unit) throws Exception
// 撤销锁
public void makeRevocable(RevocationListener<InterProcessMutex> listener)
public void makeRevocable(final RevocationListener<InterProcessMutex> listener, Executor executor)
```

定义一个 FakeLimitedResource 类来模拟一个共享资源，该资源一次只能被一个线程使用，直到使用结束，下一个线程才能使用，否则会抛出异常

```java
public class FakeLimitedResource {
    private final AtomicBoolean inUse = new AtomicBoolean(false);

    // 模拟只能单线程操作的资源
    public void use() throws InterruptedException {
        if (!inUse.compareAndSet(false, true)) {
            // 在正确使用锁的情况下，此异常不可能抛出
            throw new IllegalStateException("Needs to be used by one client at a time");
        }
        try {
            Thread.sleep((long) (100 * Math.random()));
        } finally {
            inUse.set(false);
        }
    }
}
```


下面的代码将创建 N 个线程来模拟分布式系统中的节点，系统将通过 InterProcessMutex 来控制对资源的同步使用；每个节点都将发起10次请求，完成 `请求锁--访问资源--再次请求锁--释放锁--释放锁` 的过程；客户端通过 `acquire` 请求锁，通过 `release` 释放锁，获得几把锁就要释放几把锁；这个共享资源一次只能被一个线程使用，如果控制同步失败，将抛异常。


```java
public class SharedReentrantLockTest {
    private static final String lockPath = "/testZK/sharedreentrantlock";
    private static final Integer clientNums = 5;
    final static FakeLimitedResource resource = new FakeLimitedResource(); // 共享的资源
    private static CountDownLatch countDownLatch = new CountDownLatch(clientNums);

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < clientNums; i++) {
            String clientName = "client#" + i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    CuratorFramework client = ZKUtils.getClient();
                    client.start();
                    Random random = new Random();
                    try {
                        final InterProcessMutex lock = new InterProcessMutex(client, lockPath);
                        // 每个客户端请求10次共享资源
                        for (int j = 0; j < 10; j++) {
                            if (!lock.acquire(10, TimeUnit.SECONDS)) {
                                throw new IllegalStateException(j + ". " + clientName + " 不能得到互斥锁");
                            }
                            try {
                                System.out.println(j + ". " + clientName + " 已获取到互斥锁");
                                resource.use(); // 使用资源
                                if (!lock.acquire(10, TimeUnit.SECONDS)) {
                                    throw new IllegalStateException(j + ". " + clientName + " 不能再次得到互斥锁");
                                }
                                System.out.println(j + ". " + clientName + " 已再次获取到互斥锁");
                                lock.release(); // 申请几次锁就要释放几次锁
                            } finally {
                                System.out.println(j + ". " + clientName + " 释放互斥锁");
                                lock.release(); // 总是在finally中释放
                            }
                            Thread.sleep(random.nextInt(100));
                        }
                    } catch (Throwable e) {
                        System.out.println(e.getMessage());
                    } finally {
                        CloseableUtils.closeQuietly(client);
                        System.out.println(clientName + " 客户端关闭！");
                        countDownLatch.countDown();
                    }
                }
            }).start();
        }
        countDownLatch.await();
        System.out.println("结束！");
    }
}
```

控制台打印日志，可以看到对资源的同步访问控制成功，并且锁是可重入的

```
0. client#3 已获取到互斥锁
0. client#3 已再次获取到互斥锁
0. client#3 释放互斥锁
0. client#1 已获取到互斥锁
0. client#1 已再次获取到互斥锁
0. client#1 释放互斥锁
0. client#2 已获取到互斥锁
0. client#2 已再次获取到互斥锁
0. client#2 释放互斥锁
0. client#0 已获取到互斥锁
0. client#0 已再次获取到互斥锁
0. client#0 释放互斥锁
0. client#4 已获取到互斥锁
0. client#4 已再次获取到互斥锁
0. client#4 释放互斥锁
1. client#1 已获取到互斥锁
1. client#1 已再次获取到互斥锁
1. client#1 释放互斥锁
2. client#1 已获取到互斥锁
2. client#1 已再次获取到互斥锁
2. client#1 释放互斥锁
1. client#4 已获取到互斥锁
1. client#4 已再次获取到互斥锁
1. client#4 释放互斥锁
1. client#3 已获取到互斥锁
1. client#3 已再次获取到互斥锁
1. client#3 释放互斥锁
1. client#2 已获取到互斥锁
1. client#2 已再次获取到互斥锁
1. client#2 释放互斥锁
2. client#4 已获取到互斥锁
2. client#4 已再次获取到互斥锁
2. client#4 释放互斥锁
....
....
client#2 客户端关闭！
9. client#0 已获取到互斥锁
9. client#0 已再次获取到互斥锁
9. client#0 释放互斥锁
9. client#3 已获取到互斥锁
9. client#3 已再次获取到互斥锁
9. client#3 释放互斥锁
client#0 客户端关闭！
8. client#4 已获取到互斥锁
8. client#4 已再次获取到互斥锁
8. client#4 释放互斥锁
9. client#4 已获取到互斥锁
9. client#4 已再次获取到互斥锁
9. client#4 释放互斥锁
client#3 客户端关闭！
client#4 客户端关闭！
结束！
```

同时在程序运行期间查看Zookeeper节点树，可以发现每一次请求的锁实际上对应一个临时顺序节点

```
[zk: localhost:2181(CONNECTED) 42] ls /testZK/sharedreentrantlock
[leases, _c_208d461b-716d-43ea-ac94-1d2be1206db3-lock-0000001659, locks, _c_64b19dba-3efa-46a6-9344-19a52e9e424f-lock-0000001658, _c_cee02916-d7d5-4186-8867-f921210b8815-lock-0000001657]
```

#### 不可重入锁

Shared Lock 与 Shared Reentrant Lock 相似，但是**不可重入**。这个不可重入锁由类 InterProcessSemaphoreMutex 来实现，使用方法和上面的类类似。

将上面程序中的 InterProcessMutex 换成不可重入锁 InterProcessSemaphoreMutex，如果再运行上面的代码，结果就会发现线程被阻塞在第二个 `acquire` 上，直到超时，也就是此锁不是可重入的。

控制台输出日志

```
0. client#2 已获取到互斥锁
0. client#1 不能得到互斥锁
0. client#4 不能得到互斥锁
0. client#0 不能得到互斥锁
0. client#3 不能得到互斥锁
client#1 客户端关闭！
client#4 客户端关闭！
client#3 客户端关闭！
client#0 客户端关闭！
0. client#2 释放互斥锁
0. client#2 不能再次得到互斥锁
client#2 客户端关闭！
结束！
```

把第二个获取锁的代码注释，程序才能正常执行

```
0. client#1 已获取到互斥锁
0. client#1 释放互斥锁
0. client#2 已获取到互斥锁
0. client#2 释放互斥锁
0. client#0 已获取到互斥锁
0. client#0 释放互斥锁
0. client#4 已获取到互斥锁
0. client#4 释放互斥锁
0. client#3 已获取到互斥锁
0. client#3 释放互斥锁
1. client#1 已获取到互斥锁
1. client#1 释放互斥锁
1. client#2 已获取到互斥锁
1. client#2 释放互斥锁
....
....
9. client#4 已获取到互斥锁
9. client#4 释放互斥锁
9. client#0 已获取到互斥锁
client#2 客户端关闭！
9. client#0 释放互斥锁
9. client#1 已获取到互斥锁
client#0 客户端关闭！
client#4 客户端关闭！
9. client#1 释放互斥锁
9. client#3 已获取到互斥锁
client#1 客户端关闭！
9. client#3 释放互斥锁
client#3 客户端关闭！
结束！
```


#### 可重入读写锁

Shared Reentrant Read Write Lock，可重入读写锁，一个读写锁管理一对相关的锁，一个负责读操作，另外一个负责写操作；读操作在写锁没被使用时可同时由多个进程使用，而写锁在使用时不允许读(阻塞)；此锁是可重入的；一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁，这也意味着写锁可以降级成读锁， 比如 `请求写锁 --->读锁 ---->释放写锁`；从读锁升级成写锁是不行的。
    
可重入读写锁主要由两个类实现：InterProcessReadWriteLock、InterProcessMutex，使用时首先创建一个 InterProcessReadWriteLock 实例，然后再根据你的需求得到读锁或者写锁，读写锁的类型是 InterProcessMutex。


```java
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < clientNums; i++) {
            final String clientName = "client#" + i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    CuratorFramework client = ZKUtils.getClient();
                    client.start();
                    final InterProcessReadWriteLock lock = new InterProcessReadWriteLock(client, lockPath);
                    final InterProcessMutex readLock = lock.readLock();
                    final InterProcessMutex writeLock = lock.writeLock();

                    try {
                        // 注意只能先得到写锁再得到读锁，不能反过来！！！
                        if (!writeLock.acquire(10, TimeUnit.SECONDS)) {
                            throw new IllegalStateException(clientName + " 不能得到写锁");
                        }
                        System.out.println(clientName + " 已得到写锁");
                        if (!readLock.acquire(10, TimeUnit.SECONDS)) {
                            throw new IllegalStateException(clientName + " 不能得到读锁");
                        }
                        System.out.println(clientName + " 已得到读锁");
                        try {
                            resource.use(); // 使用资源
                        } finally {
                            System.out.println(clientName + " 释放读写锁");
                            readLock.release();
                            writeLock.release();
                        }
                    } catch (Exception e) {
                        System.out.println(e.getMessage());
                    } finally {
                        CloseableUtils.closeQuietly(client);
                        countDownLatch.countDown();
                    }
                }
            }).start();
        }
        countDownLatch.await();
        System.out.println("结束！");
    }
}
```

控制台打印日志

```
client#1 已得到写锁
client#1 已得到读锁
client#1 释放读写锁
client#2 已得到写锁
client#2 已得到读锁
client#2 释放读写锁
client#0 已得到写锁
client#0 已得到读锁
client#0 释放读写锁
client#4 已得到写锁
client#4 已得到读锁
client#4 释放读写锁
client#3 已得到写锁
client#3 已得到读锁
client#3 释放读写锁
结束！
```

####  信号量

Shared Semaphore，一个计数的信号量类似JDK的 Semaphore，JDK中 Semaphore 维护的一组许可(permits)，而Cubator中称之为租约(Lease)。有两种方式可以决定 semaphore 的最大租约数，第一种方式是由用户给定的 path 决定，第二种方式使用 SharedCountReader 类。如果不使用 SharedCountReader，没有内部代码检查进程是否假定有10个租约而进程B假定有20个租约。 所以所有的实例必须使用相同的 numberOfLeases 值.

信号量主要实现类有：
```
InterProcessSemaphoreV2 - 信号量实现类
Lease - 租约(单个信号)
SharedCountReader - 计数器，用于计算最大租约数量
```

调用 `acquire` 会返回一个租约对象，客户端必须在 finally 中 close 这些租约对象，否则这些租约会丢失掉。但是，如果客户端session由于某种原因比如crash丢掉，那么这些客户端持有的租约会自动close，这样其它客户端可以继续使用这些租约。租约还可以通过下面的方式返还：

```
public void returnLease(Lease lease)
public void returnAll(Collection<Lease> leases) 
```
注意一次你可以请求多个租约，如果 Semaphore 当前的租约不够，则请求线程会被阻塞。同时还提供了超时的重载方法。
```
public Lease acquire() throws Exception
public Collection<Lease> acquire(int qty) throws Exception
public Lease acquire(long time, TimeUnit unit) throws Exception
public Collection<Lease> acquire(int qty, long time, TimeUnit unit) throws Exception
```

一个Demo程序如下

```java
public class SharedSemaphoreTest {
    private static final int MAX_LEASE = 10;
    private static final String PATH = "/testZK/semaphore";
    private static final FakeLimitedResource resource = new FakeLimitedResource();

    public static void main(String[] args) throws Exception {
        CuratorFramework client = ZKUtils.getClient();
        client.start();
        InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, PATH, MAX_LEASE);
        Collection<Lease> leases = semaphore.acquire(5);
        System.out.println("获取租约数量：" + leases.size());
        Lease lease = semaphore.acquire();
        System.out.println("获取单个租约");
        resource.use(); // 使用资源
        // 再次申请获取5个leases，此时leases数量只剩4个，不够，将超时
        Collection<Lease> leases2 = semaphore.acquire(5, 10, TimeUnit.SECONDS);
        System.out.println("获取租约，如果超时将为null： " + leases2);
        System.out.println("释放租约");
        semaphore.returnLease(lease);
        // 再次申请获取5个，这次刚好够
        leases2 = semaphore.acquire(5, 10, TimeUnit.SECONDS);
        System.out.println("获取租约，如果超时将为null： " + leases2);
        System.out.println("释放集合中的所有租约");
        semaphore.returnAll(leases);
        semaphore.returnAll(leases2);
        client.close();
        System.out.println("结束!");
    }
}
```

控制台打印日志

```
获取租约数量：5
获取单个租约
获取租约，如果超时将为null： null
释放租约
获取租约，如果超时将为null： [org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2$3@3108bc, org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2$3@370736d9, org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2$3@5f9d02cb, org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2$3@63753b6d, org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2$3@6b09bb57]
释放集合中的所有租约
结束!
```

注意：上面所讲的4种锁都是公平锁(fair)。从ZooKeeper的角度看，每个客户端都按照请求的顺序获得锁，相当公平。

#### 多锁

Multi Shared Lock 是一个锁的容器。当调用 `acquire`，所有的锁都会被 `acquire`，如果请求失败，所有的锁都会被 `release`。同样调用 `release` 时所有的锁都被 `release`(失败被忽略)。基本上，它就是组锁的代表，在它上面的请求释放操作都会传递给它包含的所有的锁。

主要涉及两个类：
```
InterProcessMultiLock - 对所对象实现类
InterProcessLock - 分布式锁接口类
```
它的构造函数需要包含的锁的集合，或者一组 ZooKeeper 的 path，用法和 Shared Lock 相同
```
public InterProcessMultiLock(CuratorFramework client, List<String> paths)
public InterProcessMultiLock(List<InterProcessLock> locks)
```

一个Demo程序如下

```java
public class MultiSharedLockTest {
    private static final String lockPath1 = "/testZK/MSLock1";
    private static final String lockPath2 = "/testZK/MSLock2";
    private static final FakeLimitedResource resource = new FakeLimitedResource();

    public static void main(String[] args) throws Exception {
        CuratorFramework client = ZKUtils.getClient();
        client.start();

        InterProcessLock lock1 = new InterProcessMutex(client, lockPath1); // 可重入锁
        InterProcessLock lock2 = new InterProcessSemaphoreMutex(client, lockPath2); // 不可重入锁
        // 组锁，多锁
        InterProcessMultiLock lock = new InterProcessMultiLock(Arrays.asList(lock1, lock2));
        if (!lock.acquire(10, TimeUnit.SECONDS)) {
            throw new IllegalStateException("不能获取多锁");
        }
        System.out.println("已获取多锁");
        System.out.println("是否有第一个锁: " + lock1.isAcquiredInThisProcess());
        System.out.println("是否有第二个锁: " + lock2.isAcquiredInThisProcess());
        try {
            resource.use(); // 资源操作
        } finally {
            System.out.println("释放多个锁");
            lock.release(); // 释放多锁
        }
        System.out.println("是否有第一个锁: " + lock1.isAcquiredInThisProcess());
        System.out.println("是否有第二个锁: " + lock2.isAcquiredInThisProcess());
        client.close();
        System.out.println("结束!");
    }
}
```


> **代码下载地址**：http://t.cn/EtVc1s4

#### 参考资料

1、 《从Paxos到Zookeeper分布式一致性原理与实践》    
2、 [Apache Curator Recipes Docs](http://curator.apache.org/curator-recipes/index.html)    
3、 [分布式锁的几种实现方式~](http://www.hollischuang.com/archives/1716)     
4、 [技术专题讨论第四期：漫谈分布式锁](http://www.spring4all.com/question/158)    
5、 [分布式锁看这篇就够了](https://zhuanlan.zhihu.com/p/42056183)     
6、 [Curator分布式锁](https://www.cnblogs.com/LiZhiW/p/4931577.html)




![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20190116_014816.png)
