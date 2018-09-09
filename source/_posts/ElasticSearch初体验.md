---
title: ElasticSearch初体验
categories:
  - 大数据
tags:
  - elasticsearch
keywords: elastic,elasticsearch,elk,elasticsearch教程
date: 2018-08-15 19:37:25
---

### 需要明白的问题

1. 什么是倒排索引？它的组成是什么？
2. 常见的相关性算分方法有哪些？
3. 为什么查询语句没有返回预期的文档？
4. 常用的数据类型有哪些？Text和Keyword的区别是什么？
5. 集群是如何搭建起来的？是如何实现故障转移的？
6. Shard具体是由什么组成的？


## Elastic Stack
构建在开源基础之上, Elastic Stack 让您能够安全可靠地获取任何来源、任何格式的数据，并且能够实时地对数据进行搜索、分析和可视化

**Elasticsearch** 是基于 JSON 的分布式搜索和分析引擎，专为实现水平扩展、高可用和管理便捷性而设计。

**Kibana** 能够以图表的形式呈现数据，并且具有可扩展的用户界面，供您全方位配置和管理 Elastic Stack。

**Logstash** 是动态数据收集管道，拥有可扩展的插件生态系统，能够与 Elasticsearch 产生强大的协同作用。

**Beats** 是轻量型采集器的平台，从边缘机器向 Logstash 和 Elasticsearch 发送数据。


### [基础概念](https://www.elastic.co/guide/en/elasticsearch/reference/current/_basic_concepts.html)

- 文档 Document ：用户存储在ES中的数据文档
- 索引 Index ：由具有一些相同字段的文档的集合
- 类型 Type :  允许将不同类型的文档存储在同一索引中，6.0开始官方不允许在一个index下建立多个type，统一type名称：doc
- 节点 Node ：一个Elasticsearch的运行实例，是集群的构成单元，存储部分或全部数据，并参与集群的索引和搜索功能
- 集群 Cluster ：由一个或多个节点组成的集合，共同保存所有的数据，对外提供服务（包括跨所有节点的联合索引和搜索功能等）
- 分片 Shards ：分片是为了解决存储大规模数据的问题，将数据切分分别存储到不同的分片中
- 副本 Replicas ：副本可以在分片或节点发生故障时提高可用性，而且由于可以在所有副本上进行并行搜索，所以也可以提高集群的吞吐量
- 近实时 Near Realtime(NRT)：从索引文档到可搜索文档的时间有一点延迟（通常为一秒）

> note:
> 1. 在创建索引的时候如果没有配置索引Mapping，一个索引默认有5个shard和1个副本，一个索引总共有10个shard（算上副本shard）
> 2. Elasticsearch 的shard实际上是一个Lucene索引，截止Lucene-5843，一个Lucene索引限制的最大文档数为2,147,483,519 (= Integer.MAX_VALUE - 128)


### 安装Elasticsearch & Kibana

ES和Kibana的安装很简单，前提需要先安装好Java8，然后执行以下命令即可

##### [elasticsearch单节点最简安装](https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html)
```
# 在Ubuntu16.04上安装，方式有很多种，选择二进制压缩包的方式安装
# 1. 在普通用户家目录下，下载压缩包
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.3.2.tar.gz
# 2. 解压
tar -xvf elasticsearch-6.3.2.tar.gz
# 3. 移动至/opt目录下
sudo mv elasticsearch-6.3.2 /opt
# 4. 修改配置文件elasticsearch.yml中的 network.host 值为 0.0.0.0，其他的配置参考官方文档
cd /opt/elasticsearch-6.3.2
vi config/elasticsearch.yml
# 5. 启动单节点，然后浏览器访问host:9200即可看到ES集群信息
bin/elasticsearch
```
![image](http://image.laijianfeng.org/20180815_153244.png)

##### [kibana最简安装](https://www.elastic.co/guide/en/kibana/current/targz.html)
```
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.3.2-linux-x86_64.tar.gz
shasum -a 512 kibana-6.3.2-linux-x86_64.tar.gz 
tar -xzf kibana-6.3.2-linux-x86_64.tar.gz
sudo mv kibana-6.3.2-linux-x86_64 /opt
cd /opt/kibana-6.3.2-linux-x86_64
# 修改 config/kibana.yml中 server.host: 0.0.0.0
# 启动Kibana，访问 host:5601即可进入kibana界面
```
![image](http://image.laijianfeng.org/20180815_153435.png)


### 交互方式 Rest API
Elasticsearch集群对外提供RESTful API

- Curl命令行
- Kibana Devtools
- Java API
- 其他各种API，如Python API等

> note:
> 我们后面主要使用 Kibana Devtools 这种交互方式

![image](http://image.laijianfeng.org/20180815_154136.png)

### [数据类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)
- 字符串： text（分词）, keyword（不分词）
- 数值型： long, integer, byte, double, float, half_float, scaled_float
- 布尔： boolean
- 日期： date
- 二进制： binary
- 范围类型： integer_range, float_range, long_range, double_range, date_range
- 复杂数据类型： Array, Object, Nested
- 地理： geo_point， geo_shape
- 专业： ip，completion， token_count， murmur3， Percolator， join
- 组合的


### 探索ES集群
Check your cluster, node, and index health, status, and statistics
Administer your cluster, node, and index data and metadata
Perform CRUD (Create, Read, Update, and Delete) and search operations against your indexes
Execute advanced search operations such as paging, sorting, filtering, scripting, aggregations, and many others

##### 使用\_cat API探索集群的健康情况
```
GET /_cat/health?v

# 结果
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1534319381 15:49:41  elasticsearch green           3         3    118  59    0    0        0             0                  -                100.0%
```
集群的健康状态(status)有三种:
- green：一切正常（集群功能齐全）
- yellow：所有数据都可用，但存在一些副本未分配（群集功能齐全）
- red：一些数据由于某种原因不可用（群集部分功能失效）


##### 查看节点信息
```
GET /_cat/nodes?v

# 结果（我的ES集群安装了三个节点）
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.100.97.207           30          96  13    0.15    0.08     0.08 mdi       *      master
10.100.97.246           68          96   3    0.00    0.00     0.00 mdi       -      hadoop2
10.100.98.22            15          97   2    0.00    0.02     0.04 mdi       -      hadoop3
```

##### 查看索引信息

```
GET /_cat/indices?v

# 结果
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   logstash-2015.05.20             4BjPjpq6RhOSCNUPMsY0MQ   5   1       4750            0     46.8mb         24.5mb
green  open   logstash-2015.05.18             mDkUKHSWR0a8UeZlKzts8Q   5   1       4631            0     45.6mb         23.8mb
green  open   hockey                          g1omiazvRSOE117w_uy_wA   5   1         11            0     45.3kb         22.6kb
green  open   .kibana                         AGdo8im_TxC04ARexUxqxw   1   1        143           10    665.6kb        332.8kb
green  open   shakespeare                     5009bDa7T16f5qTeyOdTlw   5   1     111396            0     43.9mb           22mb
green  open   logstash-2015.05.19             az4Jen4nT7-J9yRYpZ0A9A   5   1       4624            0     44.7mb         23.1mb
...
```

### 操作数据

##### 插入文档并查询
```
# 插入一个文档
PUT /customer/_doc/1?pretty
{
  "name": "John Doe"
}

# 结果
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 1,
  "_primary_term": 1
}

# 查询该文档
GET /customer/_doc/1

#结果
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "name": "John Doe"
  }
}
```
> note:
> 1. `customer` 为索引名，`_doc` 为type，1为文档\_id，需要注意的是：在es6.x建议索引的type值固定为`_doc`，在之后的版本将删除type了；文档id若不指定，es会自动分配一个\_id给文档
> 2. 插入文档后，查看索引信息` GET /_cat/indices?v `可以看到多了 customer 的索引信息
> 3. 文档结果，\_source字段是原始的json内容，其他的为文档元数据


##### [文档元数据](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-fields.html)
用于标注文档的元信息

- _index: 文档所在的索引名
- _type: 文档所在的类型名
- _id: 文档的唯一id
- \_uid: 组合id，由\_type和\_id组成（6.0开始\_type不再起作用，同\_id一样）
- _source: 文档的原始json数据，可以从这里获取每个字段的内容
- _all: 整合所有字段内容到该字段，默认禁用
- \_routing 默认值为 \_id，决定文档存储在哪个shard上：`shard_num = hash(_routing) % num_primary_shards` 


##### 删除索引
```
DELETE customer

#结果
{
  "acknowledged": true
}

GET /_cat/indices?v

# 再次查看索引信息，可以发现 customer 不存在，已被删除
```

##### 更新文档
```
PUT /customer/_doc/1?pretty
{
  "name": "John Doe"
}

POST /customer/_doc/1/_update
{
  "doc": { "name": "Jane Doe" }
}

POST /customer/_doc/1/_update
{
  "doc": { "name": "Jane Doe", "age": 20 }
}

# 可以看到 \_version的值一直在增加
```

##### 删除文档
```
DELETE /customer/_doc/2
```

##### 批量操作
es提供了[\_bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/docs-bulk.html)供批量操作，可以提高索引、更新、删除等操作的效率

\_bulk操作的类型有四种：
- index 索引：若已存在，则覆盖，文档不存在则创建
- create 创建：文档不存在则异常
- delete 删除
- update 更新

```
# _bulk 任务：
# 1. index创建 customer索引下id为3的文档
# 2. delete删除 customer索引下id为3的文档
# 3. create创建 customer索引下id为3的文档
# 4. update更新 customer索引下id为3的文档

POST _bulk
{"index":{"_index":"customer","_type":"_doc","_id":"3"}}
{"name":"whirly"}
{"delete":{"_index":"customer","_type":"_doc","_id":"3"}}
{"create":{"_index":"customer","_type":"_doc","_id":"3"}}
{"name":"whirly2"}
{"update":{"_index":"customer","_type":"_doc","_id":"3"}}
{"doc":{"name":"whirly3"}}
```
![image](http://image.laijianfeng.org/20180815_164226.png)

> note:
1. 批量查询用的是 [Multi Get API](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/docs-multi-get.html)


### [探索数据](https://www.elastic.co/guide/en/elasticsearch/reference/current/_exploring_your_data.html)

一个[简单的数据集](https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/src/test/resources/accounts.json)，数据结构如下：
```
{
    "account_number": 0,
    "balance": 16623,
    "firstname": "Bradshaw",
    "lastname": "Mckenzie",
    "age": 29,
    "gender": "F",
    "address": "244 Columbus Place",
    "employer": "Euron",
    "email": "bradshawmckenzie@euron.com",
    "city": "Hobucken",
    "state": "CO"
}
```

导入这个简单的数据集到es中
```
# 下载
wget https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/src/test/resources/accounts.json
# 导入
curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_doc/_bulk?pretty&refresh" --data-binary "@accounts.json"
```

上述命令是通过 \_bulk API 将 account.json 的内容插入 bank 索引中，type 为 \_doc
```
# account.json的内容:

{"index":{"_id":"1"}}
{"account_number":1,"balance":39225,"firstname":"Amber","lastname":"Duke","age":32,"gender":"M","address":"880 Holmes Lane","employer":"Pyrami","email":"amberduke@pyrami.com","city":"Brogan","state":"IL"}
...


# 导入完成后可以看到 bank 索引已存在 1000 条数据
GET bank/_search
```

##### 查询数据 API
任务：查询所有数据，根据 account_number 字段升序排序

1. [URI Search 方式](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/search-uri-request.html#search-uri-request)
```
GET /bank/_search?q=*&sort=account_number:asc&pretty
```

2. [Request Body Search](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/search-request-body.html) 方式
```
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

结果
```
{
  "took": 41,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1000,
    "max_score": null,
    "hits": [
      {
        "_index": "bank",
        "_type": "account",
        "_id": "0",
        "_score": null,
        "_source": {
          "account_number": 0,
          "balance": 16623,
          "firstname": "Bradshaw",
          "lastname": "Mckenzie",
          "age": 29,
          "gender": "F",
          "address": "244 Columbus Place",
          "employer": "Euron",
          "email": "bradshawmckenzie@euron.com",
          "city": "Hobucken",
          "state": "CO"
        },
        "sort": [
          0
        ]
      }...
    ]
  }
}
```

各个参数意思：
- took：本次查询耗费的时间（单位：毫秒）
- timed_out：是否超时
- _shards：本次查询搜索的 shard 的数量，包括成功的和失败的
- hits：查询结果
- hits.total：匹配的文档数量
- hits.hits：匹配的文档，默认返回10个文档
- hits.sort：排序的值
- _score：文档的得分
- hits.max_score：所有文档最高的得分


### 简要介绍 [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/query-dsl.html)
这个Elasticsearch提供的基于 json 的查询语言，我们通过一个小任务来了解一下

任务要求：
1. 查询 firstname 中为 "R" 开头，年龄在 20 到 30 岁之间的人物信息
2. 限制返回的字段为 firstname,city,address,email,balance
3. 根据年龄倒序排序，返回前十条数据
4. 对 firstname 字段进行高亮显示
5. 同时求所有匹配人物的 平均balance

```
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_phrase_prefix": {
            "firstname": "R"
          }
        }
      ],
      "filter": {
        "range": {
          "age": {
            "gte": 20,
            "lte": 30
          }
        }
      }
    }
  },
  "from": 0,
  "size": 10,
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ],
  "_source": [
    "firstname",
    "city",
    "address",
    "email",
    "balance"
  ],
  "highlight": {
    "fields": {
      "firstname": {}
    }
  },
  "aggs": {
    "avg_age": {
      "avg": {
        "field": "balance"
      }
    }
  }
}

```

其中：
- query 部分可以写各种查询条件
- from, size 设置要返回的文档的起始序号
- sort 设置排序规则
- \_source 设置要返回的文档的字段
- highlight 设置高亮的字段
- aggs 为设置聚合统计规则




##### 更多查询示例
- match\_all 查询 bank 索引所有文档


```
GET /bank/_search
{
  "query": {
    "match_all": {}
  },
  "size": 2
}
```

- match 全文搜索，查询 address 字段值为 mill lane 的所有文档


```
GET /bank/_search
{
  "query": {
    "match": {
      "address": "mill lane"
    }
  }
}
```

- match\_phrase 短语匹配


```
GET /bank/_search
{
  "query": {
    "match_phrase": {
      "address": "mill lane"
    }
  }
}
```
> note: 
> match 和 match\_phrase 的区别：
> - match 中会分词，将 mill lane 拆分为 mill 和 lane， 实际查询 address 中有 mill **或者** lane 的文档
> - match\_phrase：将 mill lane 作为一个整体查询，实际查询 address 中有 mill lane 的文档

- 布尔查询（多条件查询）


```
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}

```


- 布尔查询-过滤
  查询 bank 索引中 balance 值在 20000 到 30000 之间的文档


```
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```


- 聚合查询
  对所有文档进行聚合，state 值相同的分到同一个桶里，分桶结果命名为 group\_by\_state ，再对每个桶里的文档的 balance 字段求平均值，结果命名为 average\_balance，通过设置 size 的值为0，不返回任何文档内容


```
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

分别计算 age 值在 20~30 ，30~40，40~50 三个年龄段的男和女的平均存款balance



```
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
```

> 参考文档：
> 1. [elasticsearch 官方文档 Getting Started](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html)
> 2. 慕课网 [Elastic Stack从入门到实践](https://coding.imooc.com/class/181.html)