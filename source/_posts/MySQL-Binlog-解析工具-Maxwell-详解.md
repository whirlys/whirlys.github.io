---
title: MySQL Binlog 解析工具 Maxwell 详解
categories:
  - 后端
tags:
  - mysql
keywords: mysql,maxwell
date: 2019-03-11 00:21:07
---

### maxwell 简介

Maxwell是一个能实时读取MySQL二进制日志binlog，并生成 JSON 格式的消息，作为生产者发送给 Kafka，Kinesis、RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序。它的常见应用场景有ETL、维护缓存、收集表级别的dml指标、增量到搜索引擎、数据分区迁移、切库binlog回滚方案等。官网(http://maxwells-daemon.io)、GitHub(https://github.com/zendesk/maxwell)

Maxwell主要提供了下列功能：

- 支持 `SELECT * FROM table` 的方式进行全量数据初始化
- 支持在主库发生failover后，自动恢复binlog位置(GTID)
- 可以对数据进行分区，解决数据倾斜问题，发送到kafka的数据支持database、table、column等级别的数据分区
- 工作方式是伪装为Slave，接收binlog events，然后根据schemas信息拼装，可以接受ddl、xid、row等各种event

除了Maxwell外，目前常用的MySQL Binlog解析工具主要有阿里的canal、mysql_streamer，三个工具对比如下：

![canal、maxwell、mysql_streamer对比](http://image.laijianfeng.org/20190310_163023.png)

canal 由Java开发，分为服务端和客户端，拥有众多的衍生应用，性能稳定，功能强大；canal 需要自己编写客户端来消费canal解析到的数据。

maxwell相对于canal的优势是使用简单，它直接将数据变更输出为json字符串，不需要再编写客户端。

#### 快速开始

首先MySQL需要先启用binlog，关于什么是MySQL binlog，可以参考文章《[MySQL Binlog 介绍](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483875&idx=1&sn=2cdc232fa3036da52a826964996506a8&chksm=e9c2edeedeb564f891b34ef1e47418bbe6b8cb6dcb7f48b5fa73b15cf1d63172df1a173c75d0&scene=0&xtrack=1&key=e3977f8a79490c6345befb88d0bbf74cbdc6b508a52e61ea076c830a5b64c552def6c6ad848d4bcc7a1d21e53e30eb5c1ead33acdb97df779d0e6fa8a0fbe4bda32c04077ea0d3511bc9f9490ad0b46c&ascene=1&uin=MjI4MTc0ODEwOQ%3D%3D&devicetype=Windows+7&version=62060719&lang=zh_CN&pass_ticket=h8jyrQ71hQc872LxydZS%2F3aU1JXFbp4raQ1KvY908BcKBeSBtXFgBY9IS9ZaLEDi)》

```
$ vi my.cnf

[mysqld]
server_id=1
log-bin=master
binlog_format=row
```

创建Maxwell用户，并赋予 maxwell 库的一些权限
```sql
CREATE USER 'maxwell'@'%' IDENTIFIED BY '123456';
GRANT ALL ON maxwell.* TO 'maxwell'@'%';
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE on *.* to 'maxwell'@'%'; 
```

使用 maxwell 之前需要先启动 kafka

```
wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.1.0/kafka_2.11-2.1.0.tgz
tar -xzf kafka_2.11-2.1.0.tgz
cd kafka_2.11-2.1.0
# 启动Zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties
```

单机启动 kafka 之前，需要修改一下配置文件，打开配置文件 `vi config/server.properties`，在文件最后加入 `advertised.host.name` 的配置，值为 kafka 所在机器的IP

```
advertised.host.name=10.100.97.246
```

不然后面通过 docker 启动 maxwell 将会报异常（其中的 hadoop2 是我的主机名）

```
17:45:21,446 DEBUG NetworkClient - [Producer clientId=producer-1] Error connecting to node hadoop2:9092 (id: 0 rack: null)
java.io.IOException: Can't resolve address: hadoop2:9092
        at org.apache.kafka.common.network.Selector.connect(Selector.java:217) ~[kafka-clients-1.0.0.jar:?]
        at org.apache.kafka.clients.NetworkClient.initiateConnect(NetworkClient.java:793) [kafka-clients-1.0.0.jar:?]
        at org.apache.kafka.clients.NetworkClient.ready(NetworkClient.java:230) [kafka-clients-1.0.0.jar:?]
        at org.apache.kafka.clients.producer.internals.Sender.sendProducerData(Sender.java:263) [kafka-clients-1.0.0.jar:?]
        at org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:238) [kafka-clients-1.0.0.jar:?]
        at org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:176) [kafka-clients-1.0.0.jar:?]
        at java.lang.Thread.run(Thread.java:748) [?:1.8.0_181]
Caused by: java.nio.channels.UnresolvedAddressException
        at sun.nio.ch.Net.checkAddress(Net.java:101) ~[?:1.8.0_181]
        at sun.nio.ch.SocketChannelImpl.connect(SocketChannelImpl.java:622) ~[?:1.8.0_181]
        at org.apache.kafka.common.network.Selector.connect(Selector.java:214) ~[kafka-clients-1.0.0.jar:?]
        ... 6 more
```

接着可以启动 kafka

```
bin/kafka-server-start.sh config/server.properties
```

测试 kafka

```
# 创建一个 topic
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

# 列出所有 topic
bin/kafka-topics.sh --list --zookeeper localhost:2181

# 启动一个生产者，然后随意发送一些消息
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
This is a message
This is another message

# 在另一个终端启动一下消费者，观察所消费的消息
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
This is a message
This is another message
```

通过 docker 快速安装并使用 Maxwell （当然之前需要自行安装 docker）

```
# 拉取镜像 
docker pull zendesk/maxwell

# 启动maxwell，并将解析出的binlog输出到控制台
docker run -ti --rm zendesk/maxwell bin/maxwell --user='maxwell' --password='123456' --host='10.100.97.246' --producer=stdout
```

测试Maxwell，首先创建一张简单的表，然后增改删数据

```sql
CREATE TABLE `test` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
 
insert into test values(1,22,"小旋锋");
update test set name='whirly' where id=1;
delete from test where id=1;
```

观察docker控制台的输出，从输出的日志中可以看出Maxwell解析出的binlog的JSON字符串的格式

```
{"database":"test","table":"test","type":"insert","ts":1552153502,"xid":832,"commit":true,"data":{"id":1,"age":22,"name":"小旋锋"}}
{"database":"test","table":"test","type":"update","ts":1552153502,"xid":833,"commit":true,"data":{"id":1,"age":22,"name":"whirly"},"old":{"name":"小旋锋"}}
{"database":"test","table":"test","type":"delete","ts":1552153502,"xid":834,"commit":true,"data":{"id":1,"age":22,"name":"whirly"}}
```

输出到 Kafka，关闭 docker，重新设置启动参数

```
docker run -it --rm zendesk/maxwell bin/maxwell --user='maxwell' \
    --password='123456' --host='10.100.97.246' --producer=kafka \
    --kafka.bootstrap.servers='10.100.97.246:9092' --kafka_topic=maxwell --log_level=debug
```

然后启动一个消费者来消费 maxwell topic的消息，观察其输出；再一次执行增改删数据的SQL，仍然可以得到相同的输出

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic maxwell
```

#### 输出JSON字符串的格式

- data 最新的数据，修改后的数据
- old 旧数据，修改前的数据
- type 操作类型，有insert, update, delete, database-create, database-alter, database-drop, table-create, table-alter, table-drop，bootstrap-insert，int(未知类型)
- xid 事务id
- commit 同一个xid代表同一个事务，事务的最后一条语句会有commit，可以利用这个重现事务
- server_id 
- thread_id 
- 运行程序时添加参数--output_ddl，可以捕捉到ddl语句
- datetime列会输出为"YYYY-MM-DD hh:mm:ss"，如果遇到"0000-00-00 00:00:00"会原样输出
- maxwell支持多种编码，但仅输出utf8编码
- maxwell的TIMESTAMP总是作为UTC处理，如果要调整为自己的时区，需要在后端逻辑上进行处理

与输出格式相关的配置如下

| 选项                     | 参数值  | 描述                                                         | 默认值 |
| ------------------------ | ------- | ------------------------------------------------------------ | ------ |
| `output_binlog_position` | BOOLEAN | 是否包含 binlog position                                     | false  |
| `output_gtid_position`   | BOOLEAN | 是否包含 gtid position                                       | false  |
| `output_commit_info`     | BOOLEAN | 是否包含 commit and xid                                      | true   |
| `output_xoffset`         | BOOLEAN | 是否包含 virtual tx-row offset                               | false  |
| `output_nulls`           | BOOLEAN | 是否包含值为NULL的字段                                       | true   |
| `output_server_id`       | BOOLEAN | 是否包含 server_id                                           | false  |
| `output_thread_id`       | BOOLEAN | 是否包含 thread_id                                           | false  |
| `output_schema_id`       | BOOLEAN | 是否包含 schema_id                                           | false  |
| `output_row_query`       | BOOLEAN | 是否包含 INSERT/UPDATE/DELETE 语句. Mysql需要开启 `binlog_rows_query_log_events` | false  |
| `output_ddl`             | BOOLEAN | 是否包含 DDL (table-alter, table-create, etc) events         | false  |
| `output_null_zerodates`  | BOOLEAN | 是否将 '0000-00-00' 转换为 null?                             | false  |

### 进阶使用

#### 基本的配置


| 选项                | 参数值                  | 描述                                  | 默认值 |
| ------------------- | ----------------------- | ------------------------------------- | ------ |
| `config`            |                         | 配置文件 `config.properties` 的路径   |        |
| `log_level`         | `debug,info,warn,error` | 日志级别                              | info   |
| `daemon`            |                         | 指定Maxwell实例作为守护进程到后台运行 |        |
| `env_config_prefix` | STRING                  | 匹配该前缀的环境变量将被视为配置值    |        |

可以把Maxwell的启动参数写到一个配置文件 `config.properties` 中，然后通过 config 选项指定，`bin/maxwell --config config.properties`

```
user=maxwell
password=123456
host=10.100.97.246
producer=kafka
kafka.bootstrap.servers=10.100.97.246:9092
kafka_topic=maxwell
```

#### mysql 配置选项


Maxwell 根据用途将 MySQL 划分为3种角色：

- `host`：主机，建maxwell库表，存储捕获到的schema等信息
    - 主要有六张表，bootstrap用于数据初始化，schemas记录所有的binlog文件信息，databases记录了所有的数据库信息，tables记录了所有的表信息，columns记录了所有的字段信息，positions记录了读取binlog的位移信息，heartbeats记录了心跳信息

- `replication_host`：复制主机，Event监听，读取该主机binlog
    - 将`host`和`replication_host`分开，可以避免 `replication_user` 往生产库里写数据

- `schema_host`：schema主机，捕获表结构schema的主机
    - binlog里面没有字段信息，所以maxwell需要从数据库查出schema，存起来。
    - `schema_host`一般用不到，但在`binlog-proxy`场景下就很实用。比如要将已经离线的binlog通过maxwell生成json流，于是自建一个mysql server里面没有结构，只用于发送binlog，此时表机构就可以制动从 schema_host 获取。

通常，这三个主机都是同一个，`schema_host` 只在有 `replication_host` 的时候使用。


与MySQL相关的有下列配置

| 选项                   | 参数值  | 描述                                                         | 默认值              |
| ---------------------- | ------- | ------------------------------------------------------------ | ------------------- |
| `host`                 | STRING  | mysql 地址                                                   | localhost           |
| `user`                 | STRING  | mysql 用户名                                                 |                     |
| `password`             | STRING  | mysql 密码                                                   | (no password)       |
| `port`                 | INT     | mysql 端口	3306                                           |                     |
| `jdbc_options`         | STRING  | mysql jdbc connection options                                | DEFAULT\_JDBC\_OPTS |
| `ssl`                  | SSL_OPT | SSL behavior for mysql cx                                    | DISABLED            |
| `schema_database`      | STRING  | Maxwell用于维护的schema和position将使用的数据库              | maxwell             |
| `client_id`            | STRING  | 用于标识Maxwell实例的唯一字符串                              | maxwell             |
| `replica_server_id`    | LONG    | 用于标识Maxwell实例的唯一数字                                | 6379 (see notes)    |
| `master_recovery`      | BOOLEAN | enable experimental master recovery code                     | false               |
| `gtid_mode`            | BOOLEAN | 是否开启基于GTID的复制                                       | false               |
| `recapture_schema`     | BOOLEAN | 重新捕获最新的表结构(schema)，不可在 config.properties中配置 | false               |
|                        |         |                                                              |                     |
| `replication_host`     | STRING  | server to replicate from. See split server roles             | schema-store host   |
| `replication_password` | STRING  | password on replication server                               | (none)              |
| `replication_port`     | INT     | port on replication server                                   | 3306                |
| `replication_user`     | STRING  | user on replication server                                   |                     |
| `replication_ssl`      | SSL_OPT | SSL behavior for replication cx cx                           | DISABLED            |
|                        |         |                                                              |                     |
| `schema_host`          | STRING  | server to capture schema from. See split server roles        | schema-store host   |
| `schema_password`      | STRING  | password on schema-capture server                            | (none)              |
| `schema_port`          | INT     | port on schema-capture server                                | 3306                |
| `schema_user`          | STRING  | user on schema-capture server                                |                     |
| `schema_ssl`           | SSL_OPT | SSL behavior for schema-capture server                       | DISABLED            |


#### 生产者的配置

仅介绍kafka，其他的生产者的配置详见官方文档。

kafka是maxwell支持最完善的一个生产者，并且内置了多个版本的kafka客户端(0.8.2.2, 0.9.0.1, 0.10.0.1, 0.10.2.1 or 0.11.0.1, 1.0.0.)，默认 kafka_version=1.0.0（当前Maxwell版本1.20.0）

Maxwell 会将消息投递到Kafka的Topic中，该Topic由 `kafka_topic` 选项指定，默认值为 `maxwell`，除了指定为静态的Topic，还可以指定为动态的，譬如 `namespace_%{database}_%{table}`，`%{database}` 和 `%{table}` 将被具体的消息的 database 和 table 替换。

Maxwell 读取配置时，如果配置项是以 `kafka.` 开头，那么该配置将设置到 Kafka Producer 客户端的连接参数中去，譬如

```
kafka.acks = 1
kafka.compression.type = snappy
kafka.retries=5
```

下面是Maxwell通用生产者和Kafka生产者的配置参数

| 选项                             | 参数值                  | 描述                                                         | 默认值        |
| -------------------------------- | ----------------------- | ------------------------------------------------------------ | ------------- |
| `producer`                       | PRODUCER_TYPE           | 生产者类型                                                   | stdout        |
| `custom_producer.factory`        | CLASS_NAME              | 自定义消费者的工厂类                                         |               |
| `producer_ack_timeout`           | PRODUCER\_ACK\_TIMEOUT  | 异步消费认为消息丢失的超时时间（毫秒ms）                     |               |
| `producer_partition_by`          | PARTITION_BY            | 输入到kafka/kinesis的分区函数                                | database      |
| `producer_partition_columns`     | STRING                  | 若按列分区，以逗号分隔的列名称                               |               |
| `producer_partition_by_fallback` | PARTITION\_BY\_FALLBACK | `producer_partition_by=column`时需要，当列不存在是使用       |               |
| `ignore_producer_error`          | BOOLEAN                 | 为false时，在kafka/kinesis发生错误时退出程序；为true时，仅记录日志 See also `dead_letter_topic` | true          |
| `kafka.bootstrap.servers`        | STRING                  | kafka 集群列表，`HOST:PORT[,HOST:PORT]`                      |               |
| `kafka_topic`                    | STRING                  | kafka topic                                                  | maxwell       |
| `dead_letter_topic`              | STRING                  | 详见官方文档                                                 |               |
| `kafka_version`                  | KAFKA_VERSION           | 指定maxwell的 kafka 生产者客户端版本，不可在config.properties中配置 | 0.11.0.1      |
| `kafka_partition_hash`           | `default,murmur3`       | 选择kafka分区时使用的hash方法                                | default       |
| `kafka_key_format`               | `array,hash`            | how maxwell outputs kafka keys, either a hash or an array of hashes | hash          |
| `ddl_kafka_topic`                | STRING                  | 当`output_ddl`为true时, 所有DDL的消息都将投递到该topic       | `kafka_topic` |


#### 过滤器配置

Maxwell 可以通过 `--filter` 配置项来指定过滤规则，通过 `exclude` 排除，通过 `include` 包含，值可以为具体的数据库、数据表、数据列，甚至用 Javascript 来定义复杂的过滤规则；可以用正则表达式描述，有几个来自官网的例子

```
# 仅匹配foodb数据库的tbl表和所有table_数字的表
--filter='exclude: foodb.*, include: foodb.tbl, include: foodb./table_\d+/'
# 排除所有库所有表，仅匹配db1数据库
--filter = 'exclude: *.*, include: db1.*'
# 排除含db.tbl.col列值为reject的所有更新
--filter = 'exclude: db.tbl.col = reject'
# 排除任何包含col_a列的更新
--filter = 'exclude: *.*.col_a = *'
# blacklist 黑名单，完全排除bad_db数据库，若要恢复，必须删除maxwell库
--filter = 'blacklist: bad_db.*' 
```

#### 数据初始化

Maxwell 启动后将从maxwell库中获取上一次停止时position，从该断点处开始读取binlog。如果binlog已经清除了，那么怎样可以通过maxwell把整张表都复制出来呢？也就是数据初始化该怎么做？

对整张表进行操作，人为地产生binlog？譬如找一个不影响业务的字段譬如update_time，然后加一秒，再减一秒？ 

```sql
update test set update_time = DATE_ADD(update_time,intever 1 second);
update test set update_time = DATE_ADD(update_time,intever -1 second);
```

这样明显存在几个大问题：

- 不存在一个不重要的字段怎么办？每个字段都很重要，不能随便地修改！
- 如果整张表很大，修改的过程耗时很长，影响了业务！
- 将产生大量非业务的binlog！

针对数据初始化的问题，Maxwell 提供了一个命令工具 `maxwell-bootstrap` 帮助我们完成数据初始化，`maxwell-bootstrap` 是基于 `SELECT * FROM table` 的方式进行全量数据初始化，不会产生多余的binlog！

这个工具有下面这些参数：


| 参数                    | 说明                                   |
| ----------------------- | -------------------------------------- |
| `--log_level LOG_LEVEL` | 日志级别（DEBUG, INFO, WARN or ERROR） |
| `--user USER`           | mysql 用户名                           |
| `--password PASSWORD`   | mysql 密码                             |
| `--host HOST`           | mysql 地址                             |
| `--port PORT`           | mysql 端口                             |
| `--database DATABASE`   | 要bootstrap的表所在的数据库            |
| `--table TABLE`         | 要引导的表                             |
| `--where WHERE_CLAUSE`  | 设置过滤条件                           |
| `--client_id CLIENT_ID` | 指定执行引导操作的Maxwell实例          |



实验一番，下面将引导 `test` 数据库中 `test` 表，首先是准备几条测试用的数据

```sql
INSERT INTO `test` VALUES (1, 1, '1');
INSERT INTO `test` VALUES (2, 2, '2');
INSERT INTO `test` VALUES (3, 3, '3');
INSERT INTO `test` VALUES (4, 4, '4');
```

然后 `reset master;` 清空binlog，删除 maxwell 库中的表。接着使用快速开始中的命令，启动Kafka、Maxwell和Kafka消费者，然后启动 `maxwell-bootstrap`

```
docker run -it --rm zendesk/maxwell bin/maxwell-bootstrap --user maxwell  \
    --password 123456 --host 10.100.97.246  --database test --table test --client_id maxwell
```

注意：`--bootstrapper=sync` 时，在处理bootstrap时，会阻塞正常的binlog解析；`--bootstrapper=async` 时，不会阻塞。

也可以执行下面的SQL，在 `maxwell.bootstrap` 表中插入记录，手动触发

```sql
insert into maxwell.bootstrap (database_name, table_name) values ('test', 'test');
```

就可以在 kafka 消费者端看见引导过来的数据了

```
{"database":"maxwell","table":"bootstrap","type":"insert","ts":1552199115,"xid":36738,"commit":true,"data":{"id":3,"database_name":"test","table_name":"test","where_clause":null,"is_complete":0,"inserted_rows":0,"total_rows":0,"created_at":null,"started_at":null,"completed_at":null,"binlog_file":null,"binlog_position":0,"client_id":"maxwell"}}
{"database":"test","table":"test","type":"bootstrap-start","ts":1552199115,"data":{}}
{"database":"test","table":"test","type":"bootstrap-insert","ts":1552199115,"data":{"id":1,"age":1,"name":"1"}}
{"database":"test","table":"test","type":"bootstrap-insert","ts":1552199115,"data":{"id":2,"age":2,"name":"2"}}
{"database":"test","table":"test","type":"bootstrap-insert","ts":1552199115,"data":{"id":3,"age":3,"name":"3"}}
{"database":"test","table":"test","type":"bootstrap-insert","ts":1552199115,"data":{"id":4,"age":4,"name":"4"}}
{"database":"maxwell","table":"bootstrap","type":"update","ts":1552199115,"xid":36756,"commit":true,"data":{"id":3,"database_name":"test","table_name":"test","where_clause":null,"is_complete":1,"inserted_rows":4,"total_rows":0,"created_at":null,"started_at":"2019-03-10 14:25:15","completed_at":"2019-03-10 14:25:15","binlog_file":"mysql-bin.000001","binlog_position":64446,"client_id":"maxwell"},"old":{"is_complete":0,"inserted_rows":1,"completed_at":null}}
{"database":"test","table":"test","type":"bootstrap-complete","ts":1552199115,"data":{}}
```

中间的4条便是 `test.test` 的binlog数据了，注意这里的 type 是 `bootstrap-insert`，而不是 `insert`。

然后再一次查看binlog，`show binlog events;`，会发现只有与 `maxwell` 相关的binlog，并没有 `test.test` 相关的binlog，所以 `maxwell-bootstrap` 命令并不会产生多余的 binlog，当数据表的数量很大时，这个好处会更加明显

Bootstrap 的过程是 `bootstrap-start -> bootstrap-insert -> bootstrap-complete`，其中，start和complete的data字段为空，不携带数据。

在进行bootstrap过程中，如果maxwell崩溃，重启时，bootstrap会完全重新开始，不管之前进行到多少，若不希望这样，可以到数据库中设置 `is_complete` 字段值为1(表示完成)，或者删除该行

#### Maxwell监控

Maxwell 提供了 `base logging mechanism, JMX, HTTP or by push to Datadog` 这四种监控方式，与监控相关的配置项有下列这些：

| 选项                       | 参数值                   | 描述                                                | 默认值         |
| -------------------------- | ------------------------ | --------------------------------------------------- | -------------- |
| `metrics_prefix`           | STRING                   | 指标的前缀                                          | MaxwellMetrics |
| `metrics_type`             | `slf4j,jmx,http,datadog` | 发布指标的方式                                      |                |
| `metrics_jvm`              | BOOLEAN                  | 是否收集JVM信息                                     | false          |
| `metrics_slf4j_interval`   | SECONDS                  | 将指标记录到日志的频率，`metrics_type`须配置为slf4j | 60             |
| `http_port`                | INT                      | `metrics_type`为http时，发布指标绑定的端口          | 8080           |
| `http_path_prefix`         | STRING                   | http的路径前缀                                      | /              |
| `http_bind_address`        | STRING                   | http发布指标绑定的地址                              | all addresses  |
| `http_diagnostic`          | BOOLEAN                  | http是否开启diagnostic后缀                          | false          |
| `http_diagnostic_timeout`  | MILLISECONDS             | http diagnostic 响应超时时间                        | 10000          |
| `metrics_datadog_type`     | `udp,http`               | `metrics_type`为datadog时发布指标的方式             | udp            |
| `metrics_datadog_tags`     | STRING                   | 提供给 datadog 的标签，如 tag1:value1,tag2:value2   |                |
| `metrics_datadog_interval` | INT                      | 推指标到datadog的频率，单位秒                       | 60             |
| `metrics_datadog_apikey`   | STRING                   | 当 `metrics_datadog_type=http` 时datadog用的api key |                |
| `metrics_datadog_host`     | STRING                   | 当`metrics_datadog_type=udp`时推指标的目标地址      | localhost      |
| `metrics_datadog_port`     | INT                      | 当`metrics_datadog_type=udp` 时推指标的端口         | 8125           |


具体可以得到哪些监控指标呢？有如下，注意所有指标都预先配置了指标前缀 `metrics_prefix`


| 指标                       | 类型     | 说明                                                         |
| -------------------------- | -------- | ------------------------------------------------------------ |
| `messages.succeeded`       | Counters | 成功发送到kafka的消息数量                                    |
| `messages.failed`          | Counters | 发送失败的消息数量                                           |
| `row.count`                | Counters | 已处理的binlog行数，注意并非所有binlog都发往kafka            |
| `messages.succeeded.meter` | Meters   | 消息成功发送到Kafka的速率                                    |
| `messages.failed.meter`    | Meters   | 消息发送失败到kafka的速率                                    |
| `row.meter`                | Meters   | 行(row)从binlog连接器到达maxwell的速率                       |
| `replication.lag`          | Gauges   | 从数据库事务提交到Maxwell处理该事务之间所用的时间（毫秒）    |
| `inflightmessages.count`   | Gauges   | 当前正在处理的消息数（等待来自目的地的确认，或在消息之前）   |
| `message.publish.time`     | Timers   | 向kafka发送record所用的时间（毫秒）                          |
| `message.publish.age`      | Timers   | 从数据库产生事件到发送到Kafka之间的时间（毫秒），精确度为+/-500ms |
| `replication.queue.time`   | Timers   | 将一个binlog事件送到处理队列所用的时间（毫秒）               |

上述有些指标为kafka特有的，并不支持所有的生产者。

实验一番，通过 http 方式获取监控指标

```
docker run -p 8080:8080 -it --rm zendesk/maxwell bin/maxwell --user='maxwell' \
    --password='123456' --host='10.100.97.246' --producer=kafka \
    --kafka.bootstrap.servers='10.100.97.246:9092' --kafka_topic=maxwell --log_level=debug \
    --metrics_type=http  --metrics_jvm=true --http_port=8080
```

上面的配置大部分与前面的相同，不同的有 `-p 8080:8080` docker端口映射，以及 `--metrics_type=http  --metrics_jvm=true --http_port=8080`，配置了通过http方式发布指标，启用收集JVM信息，端口为8080，之后可以通过 `http://10.100.97.246:8080/metrics` 便可获取所有的指标

![Maxwell监控](http://image.laijianfeng.org/20190310_234508.png)


http 方式有四种后缀，分别对应四种不同的格式


| endpoint       | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `/metrics`     | 所有指标以JSON格式返回                                       |
| `/prometheus`  | 所有指标以Prometheus格式返回（Prometheus是一套开源的监控&报警&时间序列数据库的组合） |
| `/healthcheck` | 返回Maxwell过去15分钟是否健康                                |
| `/ping`        | 简单的测试，返回 `pong`                                      |

如果是通过 JMX 的方式收集Maxwell监控指标，可以 `JAVA_OPTS` 环境变量配置JMX访问权限

```
export JAVA_OPTS="-Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.port=9010 \
-Dcom.sun.management.jmxremote.local.only=false \
-Dcom.sun.management.jmxremote.authenticate=false \
-Dcom.sun.management.jmxremote.ssl=false \
-Djava.rmi.server.hostname=10.100.97.246"
```

#### 多个Maxwell实例

在不同的配置下，Maxwell可以在同一个主服务器上运行多个实例。如果希望让生产者以不同的配置运行，例如将来自不同组的表(table)的事件投递到不同的Topic中，这将非常有用。Maxwell的每个实例都必须配置一个唯一的client_id，以便区分的binlog位置。

#### GTID 支持

Maxwell 从1.8.0版本开始支持基于GTID的复制([GTID-based replication](https://dev.mysql.com/doc/refman/5.6/en/replication-gtids.html))，在GTID模式下，Maxwell将在主机更改后透明地选择新的复制位置。

**什么是GTID Replication？**

GTID (Global Transaction ID) 是对于一个已提交事务的编号，并且是一个全局唯一的编号。

从 MySQL 5.6.5 开始新增了一种基于 GTID 的复制方式。通过 GTID 保证了每个在主库上提交的事务在集群中有一个唯一的ID。这种方式强化了数据库的主备一致性，故障恢复以及容错能力。

在原来基于二进制日志的复制中，从库需要告知主库要从哪个偏移量进行增量同步，如果指定错误会造成数据的遗漏，从而造成数据的不一致。借助GTID，在发生主备切换的情况下，MySQL的其它从库可以自动在新主库上找到正确的复制位置，这大大简化了复杂复制拓扑下集群的维护，也减少了人为设置复制位置发生误操作的风险。另外，基于GTID的复制可以忽略已经执行过的事务，减少了数据发生不一致的风险。

### 注意事项

#### timestamp column

maxwell对时间类型（datetime, timestamp, date）都是**当做字符串处理**的，这也是为了保证数据一致(比如`0000-00-00 00:00:00`这样的时间在timestamp里是非法的，但mysql却认，解析成java或者python类型就是null/None)。

如果MySQL表上的字段是 timestamp 类型，是有时区的概念，**binlog解析出来的是标准UTC时间**，但用户看到的是本地时间。比如 `f_create_time timestamp` 创建时间是北京时间 `2018-01-05 21:01:01`，那么mysql实际存储的是 `2018-01-05 13:01:01`，binlog里面也是这个时间字符串。如果不做消费者不做时区转换，会少8个小时。

与其每个客户端都要考虑这个问题，我觉得更合理的做法是提供时区参数，然后maxwell自动处理时区问题，否则要么客户端先需要知道哪些列是timestamp类型，或者连接上原库缓存上这些类型。

#### binary column

maxwell可以处理binary类型的列，如blob、varbinary，它的做法就是对二进制列使用 `base64_encode`，当做字符串输出到json。消费者拿到这个列数据后，不能直接拼装，需要 `base64_decode`。

#### 表结构不同步

如果是拿比较老的binlog，放到新的mysql server上去用maxwell拉去，有可能表结构已经发生了变化，比如binlog里面字段比 `schema_host` 里面的字段多一个。目前这种情况没有发现异常，比如阿里RDS默认会为 无主键无唯一索引的表，增加一个`__##alibaba_rds_rowid##__`，在 `show create table` 和 `schema` 里面都看不到这个隐藏主键，但binlog里面会有，同步到从库。

另外我们有通过git去管理结构版本，如果真有这种场景，也可以应对。

#### 大事务binlog

当一个事物产生的binlog量非常大的时候，比如迁移日表数据，maxwell为了控制内存使用，会自动将处理不过来的binlog放到文件系统

```
Using kafka version: 0.11.0.1
21:16:07,109 WARN  MaxwellMetrics - Metrics will not be exposed: metricsReportingType not configured.
21:16:07,380 INFO  SchemaStoreSchema - Creating maxwell database
21:16:07,540 INFO  Maxwell - Maxwell v?? is booting (RabbitmqProducer), starting at Position[BinlogPosition[mysql-bin.006235:24980714],
lastHeartbeat=0]
21:16:07,649 INFO  AbstractSchemaStore - Maxwell is capturing initial schema
21:16:08,267 INFO  BinlogConnectorReplicator - Setting initial binlog pos to: mysql-bin.006235:24980714
21:16:08,324 INFO  BinaryLogClient - Connected to rm-xxxxxxxxxxx.mysql.rds.aliyuncs.com:3306 at mysql-bin.006235/24980714 (sid:637
9, cid:9182598)
21:16:08,325 INFO  BinlogConnectorLifecycleListener - Binlog connected.
03:15:36,104 INFO  ListWithDiskBuffer - Overflowed in-memory buffer, spilling over into /tmp/maxwell7935334910787514257events
03:17:14,880 INFO  ListWithDiskBuffer - Overflowed in-memory buffer, spilling over into /tmp/maxwell3143086481692829045events
```

但是遇到另外一个问题，overflow随后就出现异常 `EventDataDeserializationException: Failed to deserialize data of EventHeaderV4`，当我另起一个maxwell指点之前的binlog postion开始解析，却有没有抛异常。事后的数据也表明并没有数据丢失。

问题产生的原因还不明，`Caused by: java.net.SocketException: Connection reset`，感觉像读取 binlog 流的时候还没读取到完整的event，异常关闭了连接。这个问题比较顽固，github上面类似问题都没有达到明确的解决。（这也从侧面告诉我们，大表数据迁移，也要批量进行，不要一个`insert into .. select` 搞定）

```
03:18:20,586 INFO  ListWithDiskBuffer - Overflowed in-memory buffer, spilling over into /tmp/maxwell5229190074667071141events
03:19:31,289 WARN  BinlogConnectorLifecycleListener - Communication failure.
com.github.shyiko.mysql.binlog.event.deserialization.EventDataDeserializationException: Failed to deserialize data of EventHeaderV4{time
stamp=1514920657000, eventType=WRITE_ROWS, serverId=2115082720, headerLength=19, dataLength=8155, nextPosition=520539918, flags=0}
        at com.github.shyiko.mysql.binlog.event.deserialization.EventDeserializer.deserializeEventData(EventDeserializer.java:216) ~[mys
ql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.EventDeserializer.nextEvent(EventDeserializer.java:184) ~[mysql-binlog-c
onnector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:890) [mysql-binlog-connector-java-0
.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.BinaryLogClient.connect(BinaryLogClient.java:559) [mysql-binlog-connector-java-0.13.0.jar:0.13
.0]
        at com.github.shyiko.mysql.binlog.BinaryLogClient$7.run(BinaryLogClient.java:793) [mysql-binlog-connector-java-0.13.0.jar:0.13.0
]
        at java.lang.Thread.run(Thread.java:745) [?:1.8.0_121]
Caused by: java.net.SocketException: Connection reset
        at java.net.SocketInputStream.read(SocketInputStream.java:210) ~[?:1.8.0_121]
        at java.net.SocketInputStream.read(SocketInputStream.java:141) ~[?:1.8.0_121]
        at com.github.shyiko.mysql.binlog.io.BufferedSocketInputStream.read(BufferedSocketInputStream.java:51) ~[mysql-binlog-connector-
java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.io.ByteArrayInputStream.readWithinBlockBoundaries(ByteArrayInputStream.java:202) ~[mysql-binlo
g-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.io.ByteArrayInputStream.read(ByteArrayInputStream.java:184) ~[mysql-binlog-connector-java-0.13
.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.io.ByteArrayInputStream.readInteger(ByteArrayInputStream.java:46) ~[mysql-binlog-connector-jav
a-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.AbstractRowsEventDataDeserializer.deserializeLong(AbstractRowsEventDataD
eserializer.java:212) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.AbstractRowsEventDataDeserializer.deserializeCell(AbstractRowsEventDataD
eserializer.java:150) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.AbstractRowsEventDataDeserializer.deserializeRow(AbstractRowsEventDataDeserializer.java:132) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.WriteRowsEventDataDeserializer.deserializeRows(WriteRowsEventDataDeserializer.java:64) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.WriteRowsEventDataDeserializer.deserialize(WriteRowsEventDataDeserializer.java:56) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.WriteRowsEventDataDeserializer.deserialize(WriteRowsEventDataDeserializer.java:32) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        at com.github.shyiko.mysql.binlog.event.deserialization.EventDeserializer.deserializeEventData(EventDeserializer.java:210) ~[mysql-binlog-connector-java-0.13.0.jar:0.13.0]
        ... 5 more
03:19:31,514 INFO  BinlogConnectorLifecycleListener - Binlog disconnected.
03:19:31,590 WARN  BinlogConnectorReplicator - replicator stopped at position: mysql-bin.006236:520531744 -- restarting
03:19:31,595 INFO  BinaryLogClient - Connected to rm-xxxxxx.mysql.rds.aliyuncs.com:3306 at mysql-bin.006236/520531744 (sid:6379, cid:9220521)
```

#### tableMapCache

前面讲过，如果我只想获取某几个表的binlog变更，需要用 include_tables 来过滤，但如果mysql server上现在删了一个表t1，但我的binlog是从昨天开始读取，被删的那个表t1在maxwell启动的时候是拉取不到表结构的。然后昨天的binlog里面有 t1 的变更，因为找不到表结构给来组装成json，会抛异常。

手动在 `maxwell.tables/columns` 里面插入记录是可行的。但这个问题的根本是，maxwell在binlog过滤的时候，只在处理row_event的时候，而对 tableMapCache 要求binlog里面的所有表都要有。

自己（seanlook）提交了一个commit，可以在做 tableMapCache 的时候也仅要求缓存 include_dbs/tables 这些表： https://github.com/seanlook/maxwell/commit/2618b70303078bf910a1981b69943cca75ee04fb


#### 提高消费性能

在用rabbitmq时，`routing_key` 是 `%db%.%table%`，但某些表产生的binlog增量非常大，就会导致各队列消息量很不平均，目前因为还没做到事务xid或者thread_id级别的并发回放，所以最小队列粒度也是表，尽量单独放一个队列，其它数据量小的合在一起。


#### binlog

Maxwell 在 maxwell 库中维护了 binlog 的位移等信息，由于一些原因譬如 `reset master;`，导致 maxwell 库中的记录与实际的binlog对不上，这时将报异常，这是可以手动修正binlog位移或者直接清空/删除 maxwell 库重建

```
com.github.shyiko.mysql.binlog.network.ServerException: Could not find first log file name in binary log index file
        at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:885)
        at com.github.shyiko.mysql.binlog.BinaryLogClient.connect(BinaryLogClient.java:564)
        at com.github.shyiko.mysql.binlog.BinaryLogClient$7.run(BinaryLogClient.java:796)
        at java.lang.Thread.run(Thread.java:748)
```
以及

```
com.github.shyiko.mysql.binlog.network.ServerException: A slave with the same server_uuid/server_id as this slave has connected to the master; the first event 'mysql-bin.000001' at 760357, the last event read from './mysql-bin.000001' at 1888540, the last byte read from './mysql-bin.000001' at 1888540.
        at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:885)
        at com.github.shyiko.mysql.binlog.BinaryLogClient.connect(BinaryLogClient.java:564)
        at com.github.shyiko.mysql.binlog.BinaryLogClient$7.run(BinaryLogClient.java:796)
        at java.lang.Thread.run(Thread.java:748)
```

#### 参考文档

- [Maxwell's Daemon Doc](http://maxwells-daemon.io/filtering/)
- [轻风博客.MySQL Binlog解析工具Maxwell 1.17.1 的安装和使用](http://pdf.us/2018/08/24/1750.html)
- [Sean.自建Binlog订阅服务 —— Maxwell](http://seanlook.com/2018/01/13/maxwell-binlog/)
- [MySQL 5.7 基于 GTID 的主从复制实践](https://www.hi-linux.com/posts/47176.html)




![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20190116_014816.png)
