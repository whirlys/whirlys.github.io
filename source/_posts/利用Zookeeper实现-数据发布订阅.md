---
title: 利用Zookeeper实现 - 数据发布订阅
categories:
  - 大数据
tags:
  - Zookeeper
keywords: Zookeeper
date: 2019-01-24 02:13:43
---



### 数据发布/订阅

所谓的数据发布/订阅，意思是发布者将数据发布到Zookeeper上的一个或一系列节点上，通过watcher机制，客户端可以监听(订阅)这些数据节点，当这些节点发生变化时，Zookeeper及时地通知客户端，从而达到动态获取数据的目的。

一种常见的场景就是配置中心。随着应用越来越多，功能越来越复杂，机器也越来越多，对于一些公共的程序配置，譬如各种功能的开关、数据库的配置、服务器的地址等，如果每个应用每个机器仍然单独维护，当要修改配置时就得一个一个地修改，这样显然非常不方便。

这些公共的配置信息通常具备以下3个特性：

- 数据量通常比较小
- 数据内容在运行时发生动态变化
- 集群中各机器共享、配置一致

可以将这些配置抽取出来，交给配置中心统一管理起来。配置中心的架构一般是这样：

![配置中心结构图](http://image.laijianfeng.org/20190123_133414.jpg)

### 开源配置中心

开源的配置中心有很多，各有特点，这里只列出几个进行简单地介绍。

#### Ctrip Apollo

github地址：https://github.com/ctripcorp/apollo

介绍：Apollo（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

#### Disconf

github地址：https://github.com/knightliao/disconf

介绍：专注于各种「分布式系统配置管理」的「通用组件」和「通用平台」, 提供统一的「配置管理服务」。主要目标是部署极其简单、部署动态化、统一管理、一个jar包，到处运行。


#### Spring Cloud Config

github地址：https://github.com/spring-cloud/spring-cloud-config

介绍：Spring Cloud Config是一个基于http协议的远程配置实现方式，通过统一的配置管理服务器进行配置管理，客户端通过https协议主动的拉取服务的的配置信息，完成配置获取。

#### Nacos

github地址：https://github.com/alibaba/nacos

介绍：Nacos是阿里最近才开源的一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。Nacos 致力于帮助您发现、配置和管理微服务。Nacos提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。


#### 利用Zookeeper实现一个配置中心

开源的配置中心当然都很优秀，但是现在我们还是先利用Zookeeper来实现一个属于自己的配置中心。

我们的配置中心保存的配置信息十分简单，就是JDBC连接MySQL需要用的连接信息。这些连接信息将转化为JSON字符串，保存在Zookeeper上的一个节点中；应用程序(通过线程模拟的)从Zookeeper中读取这些配置信息，然后查询数据库；当修改数据库连接信息时(**切换数据库**)，应用程序能及时的拉取新的连接信息，使用新的连接查询数据库。

定义一个 MysqlConfig 类，方便使用 FastJSON 将配置信息在JSON字符串与对象之间做转换。

```java
@AllArgsConstructor
@Data
public class MysqlConfig {
    private String url;
    private String driver;
    private String username;
    private String password;
}
```

最开始，将Zookeeper上节点的配置信息初始化为 test 数据库的连接信息，然后启动 N 个线程(模拟应用程序)，读取连接信息并查询数据，同时设置监听节点；等待 10 秒钟之后，将配置切换为 test2 数据库的连接信息，这时应用程序将受到配置变更的通知，然后获取信息连接信息，重新查询数据库。

```java
// 工具类
public class ZKUtils {
    private static final String zkServerIps = "master:2181,hadoop2:2181";

    public static synchronized CuratorFramework getClient() {
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString(zkServerIps)
                .sessionTimeoutMs(6000).connectionTimeoutMs(3000) //.namespace("LeaderLatchTest")
                .retryPolicy(new ExponentialBackoffRetry(1000, 3)).build();
        return client;
    }
}

// 配置中心示例，模拟数据库切换
public class ConfigCenterTest {
    // test 数据库的 test1 表
    private static final MysqlConfig mysqlConfig_1 = new MysqlConfig("jdbc:mysql://master:3306/test?useUnicode=true&characterEncoding=utf-8&useSSL=false", "com.mysql.jdbc.Driver", "root", "123456");
    // test2 数据库的 test1 表
    private static final MysqlConfig mysqlConfig_2 = new MysqlConfig("jdbc:mysql://master:3306/test2?useUnicode=true&characterEncoding=utf-8&useSSL=false", "com.mysql.jdbc.Driver", "root", "123456");
    // 存储MySQL配置信息的节点路径
    private static final String configPath = "/testZK/jdbc/mysql";
    private static final Integer clientNums = 3;
    private static CountDownLatch countDownLatch = new CountDownLatch(clientNums);

    public static void main(String[] args) throws Exception {
        // 最开始时设置MySQL配置信息为 mysqlConfig_1
        setMysqlConfig(mysqlConfig_1);
        // 启动 clientNums 个线程，模拟分布式系统中的节点，
        // 从Zookeeper中获取MySQL的配置信息，查询数据
        for (int i = 0; i < clientNums; i++) {
            String clientName = "client#" + i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    CuratorFramework client = ZKUtils.getClient();
                    client.start();
                    try {
                        Stat stat = new Stat();
                        // 如果要监听多个子节点则应该使用 PathChildrenCache
                        final NodeCache cacheNode = new NodeCache(client, configPath, false);
                        cacheNode.start(true);  // true 表示启动时立即从Zookeeper上获取节点

                        byte[] nodeData = cacheNode.getCurrentData().getData();
                        MysqlConfig mysqlConfig = JSON.parseObject(new String(nodeData), MysqlConfig.class);
                        queryMysql(clientName, mysqlConfig);    // 查询数据

                        cacheNode.getListenable().addListener(new NodeCacheListener() {
                            @Override
                            public void nodeChanged() throws Exception {
                                byte[] newData = cacheNode.getCurrentData().getData();
                                MysqlConfig newMysqlConfig = JSON.parseObject(new String(newData), MysqlConfig.class);
                                queryMysql(clientName, newMysqlConfig);    // 查询数据
                            }
                        });
                        Thread.sleep(20 * 1000);
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        client.close();
                        countDownLatch.countDown();
                    }
                }
            }).start();
        }
        Thread.sleep(10 * 1000);
        System.out.println("\n---------10秒钟后将MySQL配置信息修改为 mysqlConfig_2---------\n");
        setMysqlConfig(mysqlConfig_2);
        countDownLatch.await();
    }

    /**
     * 初始化，最开始的时候的MySQL配置为 mysqlConfig_1
     */
    public static void setMysqlConfig(MysqlConfig config) throws Exception {
        CuratorFramework client = ZKUtils.getClient();
        client.start();
        String mysqlConfigStr = JSON.toJSONString(config);
        Stat s = client.checkExists().forPath(configPath);
        if (s != null) {
            Stat resultStat = client.setData().forPath(configPath, mysqlConfigStr.getBytes());
            System.out.println(String.format("节点 %s 已存在，更新数据为：%s", configPath, mysqlConfigStr));
        } else {
            client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(configPath, mysqlConfigStr.getBytes());
            System.out.println(String.format("创建节点：%s，初始化数据为：%s", configPath, mysqlConfigStr));
        }
        System.out.println();
        client.close();
    }

    /**
     * 通过配置信息，查询MySQL数据库
     */
    public static synchronized void queryMysql(String clientName, MysqlConfig mysqlConfig) throws ClassNotFoundException, SQLException {
        System.out.println(clientName + " 查询MySQL数据，使用的MySQL配置信息：" + mysqlConfig);
        Class.forName(mysqlConfig.getDriver());
        Connection connection = DriverManager.getConnection(mysqlConfig.getUrl(), mysqlConfig.getUsername(), mysqlConfig.getPassword());
        Statement statement = connection.createStatement();
        ResultSet resultSet = statement.executeQuery("select * from test1");
        while (resultSet.next()) {
            System.out.println(String.format("id=%s, name=%s, age=%s", resultSet.getString(1), resultSet.getString(2), resultSet.getString(3)));
        }
        System.out.println();
        resultSet.close();
        statement.close();
        connection.close();
    }
}
```


控制台打印日志

```
client#2 查询MySQL数据，使用的MySQL配置信息：MysqlConfig(url=jdbc:mysql://master:3306/test?useUnicode=true&characterEncoding=utf-8&useSSL=false, driver=com.mysql.jdbc.Driver, username=root, password=123456)
id=2, name=赖键锋, age=25
id=3, name=小旋锋, age=22000
id=4, name=test, age=100

client#1 查询MySQL数据，使用的MySQL配置信息：MysqlConfig(url=jdbc:mysql://master:3306/test?useUnicode=true&characterEncoding=utf-8&useSSL=false, driver=com.mysql.jdbc.Driver, username=root, password=123456)
id=2, name=赖键锋, age=25
id=3, name=小旋锋, age=22000
id=4, name=test, age=100

client#0 查询MySQL数据，使用的MySQL配置信息：MysqlConfig(url=jdbc:mysql://master:3306/test?useUnicode=true&characterEncoding=utf-8&useSSL=false, driver=com.mysql.jdbc.Driver, username=root, password=123456)
id=2, name=赖键锋, age=25
id=3, name=小旋锋, age=22000
id=4, name=test, age=100


---------10秒钟后将MySQL配置信息修改为 mysqlConfig_2---------

节点 /testZK/jdbc/mysql 已存在，更新数据为：{"driver":"com.mysql.jdbc.Driver","password":"123456","url":"jdbc:mysql://master:3306/test2?useUnicode=true&characterEncoding=utf-8&useSSL=false","username":"root"}

client#1 查询MySQL数据，使用的MySQL配置信息：MysqlConfig(url=jdbc:mysql://master:3306/test2?useUnicode=true&characterEncoding=utf-8&useSSL=false, driver=com.mysql.jdbc.Driver, username=root, password=123456)
id=2, name=赖键锋, age=23
id=3, name=小旋锋, age=22
id=4, name=whirly, age=24

client#2 查询MySQL数据，使用的MySQL配置信息：MysqlConfig(url=jdbc:mysql://master:3306/test2?useUnicode=true&characterEncoding=utf-8&useSSL=false, driver=com.mysql.jdbc.Driver, username=root, password=123456)
id=2, name=赖键锋, age=23
id=3, name=小旋锋, age=22
id=4, name=whirly, age=24

client#0 查询MySQL数据，使用的MySQL配置信息：MysqlConfig(url=jdbc:mysql://master:3306/test2?useUnicode=true&characterEncoding=utf-8&useSSL=false, driver=com.mysql.jdbc.Driver, username=root, password=123456)
id=2, name=赖键锋, age=23
id=3, name=小旋锋, age=22
id=4, name=whirly, age=24
```

上面采用的示例是通过 NodeCache 来监听单个节点，如果要监听多个子节点则须使用 PathChildrenCache，使用示例可以参考《[Zookeeper 分布式协调服务介绍](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483840&idx=1&sn=615f280b8006333e3b943135e2156ce7&chksm=e9c2edcddeb564dba4cfae6f1c7d125b374b3fb8caf36f0f5da3d48a7ca05d0190e7a98abff6&scene=0&xtrack=1#rd)》


#### 相关文章

- [Zookeeper 分布式协调服务介绍](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483840&idx=1&sn=615f280b8006333e3b943135e2156ce7&chksm=e9c2edcddeb564dba4cfae6f1c7d125b374b3fb8caf36f0f5da3d48a7ca05d0190e7a98abff6&scene=0&xtrack=1#rd)
- [利用Zookeeper实现 - Master选举](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483848&idx=1&sn=dea24723c9e578cf1deaf540dc8f00ed&chksm=e9c2edc5deb564d3f126329ce0940fadbdede07559c9d86a88a42b8038fcae8b1994ff6e45a9&scene=0#rd)

#### 后序

代码下载：http://t.cn/E5ncvDR

我的博客：laijianfeng.org


> 参考：  
> 《从Paxos到Zookeeper分布式一致性原理与实践》     

![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20190116_014816.png)

