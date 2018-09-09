---
title: Elasticsearch 6.x Mapping设置
categories:
  - 大数据
tags:
  - elasticsearch
keywords: Elasticsearch，Elasticsearch6.x
date: 2018-08-16 21:14:33
---

### [Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)
类似于数据库中的表结构定义，主要作用如下：
- 定义Index下字段名（Field Name）
- 定义字段的类型，比如数值型，字符串型、布尔型等
- 定义倒排索引的相关配置，比如是否索引、记录postion等   

需要注意的是，在索引中定义太多字段可能会导致索引膨胀，出现内存不足和难以恢复的情况，下面有几个设置：
- index.mapping.total_fields.limit：一个索引中能定义的字段的最大数量，默认是 1000
- index.mapping.depth.limit：字段的最大深度，以内部对象的数量来计算，默认是20
- index.mapping.nested_fields.limit：索引中嵌套字段的最大数量，默认是50

### [数据类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)

#### 核心数据类型
- 字符串 - text
    - 用于全文索引，该类型的字段将通过分词器进行分词，最终用于构建索引
- 字符串 - keyword
    - 不分词，只能搜索该字段的完整的值，只用于 filtering 
- 数值型 
    - long：有符号64-bit integer：-2^63 ~ 2^63 - 1 
    - integer：有符号32-bit integer，-2^31 ~ 2^31 - 1 
    - short：有符号16-bit integer，-32768 ~ 32767
    - byte： 有符号8-bit integer，-128 ~ 127
    - double：64-bit IEEE 754 浮点数
    - float：32-bit IEEE 754 浮点数
    - half\_float：16-bit IEEE 754 浮点数
    - scaled\_float
- 布尔 - boolean
    - 值：false, "false", true, "true"
- 日期 - date
    - 由于Json没有date类型，所以es通过识别字符串是否符合format定义的格式来判断是否为date类型
    - format默认为：`strict_date_optional_time||epoch_millis` [format](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html)
- 二进制 - binary
    - 该类型的字段把值当做经过 base64 编码的字符串，默认不存储，且不可搜索
- 范围类型
    - 范围类型表示值是一个范围，而不是一个具体的值
    - 譬如 age 的类型是 integer\_range，那么值可以是  {"gte" : 10, "lte" : 20}；搜索 "term" : {"age": 15} 可以搜索该值；搜索 "range": {"age": {"gte":11, "lte": 15}} 也可以搜索到
    - range参数 relation 设置匹配模式
        - INTERSECTS ：默认的匹配模式，只要搜索值与字段值有交集即可匹配到
        - WITHIN：字段值需要完全包含在搜索值之内，也就是字段值是搜索值的子集才能匹配
        - CONTAINS：与WITHIN相反，只搜索字段值包含搜索值的文档
    - integer\_range
    - float\_range
    - long\_range
    - double\_range
    - date\_range：64-bit 无符号整数，时间戳（单位：毫秒）
    - ip\_range：IPV4 或 IPV6 格式的字符串

    ​

```
# 创建range索引
PUT range_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "expected_attendees": {
          "type": "integer_range"
        },
        "time_frame": {
          "type": "date_range", 
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
        }
      }
    }
  }
}

# 插入一个文档
PUT range_index/_doc/1
{
  "expected_attendees" : { 
    "gte" : 10,
    "lte" : 20
  },
  "time_frame" : { 
    "gte" : "2015-10-31 12:00:00", 
    "lte" : "2015-11-05"
  }
}

# 12在 10~20的范围内，可以搜索到文档1
GET range_index/_search
{
  "query" : {
    "term" : {
      "expected_attendees" : {
        "value": 12
      }
    }
  }
}

# within可以搜索到文档
# 可以修改日期，然后分别对比CONTAINS，WITHIN，INTERSECTS的区别
GET range_index/_search
{
  "query" : {
    "range" : {
      "time_frame" : { 
        "gte" : "2015-11-02",
        "lte" : "2015-11-03",
        "relation" : "within" 
      }
    }
  }
}
```

#### 复杂数据类型
- 数组类型 Array
    - 字符串数组 [ "one", "two" ]
    - 整数数组 [ 1, 2 ]
    - 数组的数组  [ 1, [ 2, 3 ]]，相当于 [ 1, 2, 3 ]
    - Object对象数组 [ { "name": "Mary", "age": 12 }, { "name": "John", "age": 10 }]
    - 同一个数组只能存同类型的数据，不能混存，譬如 [ 10, "some string" ] 是错误的
    - 数组中的 null 值将被 null_value 属性设置的值代替或者被忽略
    - 空数组 [] 被当做 missing field 处理
- 对象类型 Object
    - 对象类型可能有内部对象
    - 被索引的形式为：manager.name.first

    ​

```
# tags字符串数组，lists 对象数组
PUT my_index/_doc/1
{
  "message": "some arrays in this document...",
  "tags":  [ "elasticsearch", "wow" ], 
  "lists": [ 
    {
      "name": "prog_list",
      "description": "programming list"
    },
    {
      "name": "cool_list",
      "description": "cool stuff list"
    }
  ]
}

```

- 嵌套类型 Nested
    - nested 类型是一种对象类型的特殊版本，它允许索引对象数组，**独立地索引每个对象**

#### 嵌套类型与Object类型的区别

通过例子来说明:
1. 插入一个文档，不设置mapping，此时 user 字段被自动识别为**对象数组**


```
DELETE my_index

PUT my_index/_doc/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}

```

2. 查询 user.first为 Alice，user.last 为 Smith的文档，理想中应该找不到匹配的文档
3. 结果是查到了文档1，为什么呢？



```
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
```

4. 是由于Object对象类型在内部被转化成如下格式的文档：
```
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```
5. user.first 和 user.last 扁平化为多值字段，alice 和 white 的**关联关系丢失了**。导致这个文档错误地匹配对 alice 和 smith 的查询
6. 如果最开始就把user设置为 nested 嵌套对象呢？


```
DELETE my_index
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "user": {
          "type": "nested" 
        }
      }
    }
  }
}

PUT my_index/_doc/1
{
  "group": "fans",
  "user": [
    {
      "first": "John",
      "last": "Smith"
    },
    {
      "first": "Alice",
      "last": "White"
    }
  ]
}
```

7. 再来进行查询，可以发现以下第一个查不到文档，第二个查询到文档1，符合我们预期


```
GET my_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }} 
          ]
        }
      }
    }
  }
}

GET my_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "White" }} 
          ]
        }
      },
      "inner_hits": { 
        "highlight": {
          "fields": {
            "user.first": {}
          }
        }
      }
    }
  }
}
```

8. nested对象将数组中每个对象作为独立隐藏文档来索引，这意味着每个嵌套对象都可以独立被搜索

9. 需要注意的是：
- 使用 [nested 查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)来搜索
- 使用 nested 和 [reverse_nested](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-reverse-nested-aggregation.html) 聚合来分析
- 使用 [nested sorting](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-sort.html#nested-sorting) 来排序
- 使用 [nested inner hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-inner-hits.html#nested-inner-hits) 来检索和高亮

#### 地理位置数据类型
- geo\_point
    - 地理位置，其值可以有如下四中表现形式：
        -  object对象："location": {"lat": 41.12, "lon": -71.34}
        -  字符串："location": "41.12,-71.34"
        -  [geohash](http://geohash.gofreerange.com/)："location": "drm3btev3e86" 
        -  数组："location": [ -71.34, 41.12 ] 
    - 查询的时候通过 [Geo Bounding Box Query ](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-bounding-box-query.html) 进行查询
- geo\_shape

#### 专用数据类型
- 记录IP地址 ip
- 实现自动补全 completion
- 记录分词数 token_count
- 记录字符串hash值 murmur3
- Percolator



```
# ip类型，存储IP
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "ip_addr": {
          "type": "ip"
        }
      }
    }
  }
}

PUT my_index/_doc/1
{
  "ip_addr": "192.168.1.1"
}

GET my_index/_search
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}

```

#### 多字段特性 multi-fields
- 允许对同一个字段采用不同的配置，比如分词，常见例子如对人名实现拼音搜索，只需要在人名中新增一个**子字段**为 pinyin 即可
- 通过参数 fields 设置


#### 设置Mapping
![image](http://image.laijianfeng.org/20180804_024134.png)
```
GET my_index/_mapping

# 结果
{
  "my_index": {
    "mappings": {
      "doc": {
        "properties": {
          "age": {
            "type": "integer"
          },
          "created": {
            "type": "date"
          },
          "name": {
            "type": "text"
          },
          "title": {
            "type": "text"
          }
        }
      }
    }
  }
}
```

### [Mapping参数](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html)

#### [analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer.html)
- 分词器，默认为standard analyzer，当该字段被索引和搜索时对字段进行分词处理

#### [boost](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-boost.html)
- 字段权重，默认为1.0

#### [dynamic](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic.html)

- Mapping中的字段类型一旦设定后，禁止直接修改，原因是：Lucene实现的倒排索引生成后不允许修改
- 只能新建一个索引，然后reindex数据
- 默认允许新增字段
- 通过dynamic参数来控制字段的新增：
    - true（默认）允许自动新增字段
    - false 不允许自动新增字段，但是文档可以正常写入，但无法对新增字段进行查询等操作
    - strict 文档不能写入，报错

    ​

```
PUT my_index
{
  "mappings": {
    "_doc": {
      "dynamic": false, 
      "properties": {
        "user": { 
          "properties": {
            "name": {
              "type": "text"
            },
            "social_networks": { 
              "dynamic": true,
              "properties": {}
            }
          }
        }
      }
    }
  }
}
```
定义后my\_index这个索引下不能自动新增字段，但是在user.social\_networks下可以自动新增子字段


#### [copy_to](https://www.elastic.co/guide/en/elasticsearch/reference/current/copy-to.html)

- 将该字段复制到目标字段，实现类似\_all的作用
- 不会出现在\_source中，只用来搜索



```
DELETE my_index
PUT my_index
{
  "mappings": {
    "doc": {
      "properties": {
        "first_name": {
          "type": "text",
          "copy_to": "full_name" 
        },
        "last_name": {
          "type": "text",
          "copy_to": "full_name" 
        },
        "full_name": {
          "type": "text"
        }
      }
    }
  }
}

PUT my_index/doc/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET my_index/_search
{
  "query": {
    "match": {
      "full_name": { 
        "query": "John Smith",
        "operator": "and"
      }
    }
  }
}
```

#### [index](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-index.html)
- 控制当前字段是否索引，默认为true，即记录索引，false不记录，即不可搜索

#### [index\_options](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-options.html)

- index_options参数控制将哪些信息添加到倒排索引，以用于搜索和突出显示，可选的值有：docs，freqs，positions，offsets
- docs：只索引 doc id
- freqs：索引 doc id 和词频，平分时可能要用到词频
- positions：索引 doc id、词频、位置，做 proximity or phrase queries 时可能要用到位置信息
- offsets：索引doc id、词频、位置、开始偏移和结束偏移，高亮功能需要用到offsets


#### [fielddata](https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html)
- 是否预加载 fielddata，默认为false
- Elasticsearch第一次查询时完整加载这个字段所有 Segment 中的倒排索引到内存中
- 如果我们有一些 5 GB 的索引段，并希望加载 10 GB 的 fielddata 到内存中，这个过程可能会要数十秒
- 将 fielddate 设置为 true ,将载入 fielddata 的代价转移到索引刷新的时候，而不是查询时，从而大大提高了搜索体验
- 参考：[预加载 fielddata](https://www.elastic.co/guide/cn/elasticsearch/guide/current/preload-fielddata.html)

#### eager\_global\_ordinals
- 是否预构建全局序号，默认false
- 参考：[预构建全局序号（Eager global ordinals）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/preload-fielddata.html#global-ordinals)


#### [doc\_values](https://www.elastic.co/guide/en/elasticsearch/reference/current/doc-values.html)
- 参考：[Doc Values and Fielddata](https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues-and-fielddata.html)

#### fields
- 该参数的目的是为了实现 multi-fields
- 一个字段，多种数据类型
- 譬如：一个字段 city 的数据类型为 text ，用于全文索引，可以通过 fields 为该字段定义 keyword 类型，用于排序和聚合



```
# 设置 mapping
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "city": {
          "type": "text",
          "fields": {
            "raw": { 
              "type":  "keyword"
            }
          }
        }
      }
    }
  }
}

# 插入两条数据
PUT my_index/_doc/1
{
  "city": "New York"
}

PUT my_index/_doc/2
{
  "city": "York"
}

# 查询，city用于全文索引 match，city.raw用于排序和聚合
GET my_index/_search
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  "sort": {
    "city.raw": "asc" 
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}
```

#### [format](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html)
- 由于JSON没有date类型，Elasticsearch预先通过format参数定义时间格式，将匹配的字符串识别为date类型，转换为时间戳（单位：毫秒）
- format默认为：`strict_date_optional_time||epoch_millis`
- Elasticsearch内建的时间格式:


| 名称                                     | 格式                     |
| -------------------------------------- | ---------------------- |
| epoch_millis                           | 时间戳（单位：毫秒）             |
| epoch_second                           | 时间戳（单位：秒）              |
| date\_optional\_time                   |                        |
| basic_date                             | yyyyMMdd               |
| basic\_date\_time                      | yyyyMMdd'T'HHmmss.SSSZ |
| basic\_date\_time\_no\_millis          | yyyyMMdd'T'HHmmssZ     |
| basic\_ordinal\_date                   | yyyyDDD                |
| basic\_ordinal\_date\_time             | yyyyDDD'T'HHmmss.SSSZ  |
| basic\_ordinal\_date\_time\_no\_millis | yyyyDDD'T'HHmmssZ      |
| basic\_time                            | HHmmss.SSSZ            |
| basic\_time\_no\_millis                | HHmmssZ                |
| basic\_t\_time                         | 'T'HHmmss.SSSZ         |
| basic\_t\_time\_no\_millis             | 'T'HHmmssZ             |

- 上述名称加前缀 `strict_` 表示为严格格式
- 更多的查看文档



#### properties
- 用于_doc，object和nested类型的字段定义**子字段**



```
PUT my_index
{
  "mappings": {
    "_doc": { 
      "properties": {
        "manager": { 
          "properties": {
            "age":  { "type": "integer" },
            "name": { "type": "text"  }
          }
        },
        "employees": { 
          "type": "nested",
          "properties": {
            "age":  { "type": "integer" },
            "name": { "type": "text"  }
          }
        }
      }
    }
  }
}

PUT my_index/_doc/1 
{
  "region": "US",
  "manager": {
    "name": "Alice White",
    "age": 30
  },
  "employees": [
    {
      "name": "John Smith",
      "age": 34
    },
    {
      "name": "Peter Brown",
      "age": 26
    }
  ]
}

```

#### normalizer
- 与 analyzer 类似，只不过 analyzer 用于 text 类型字段，分词产生多个 token，而 normalizer 用于 keyword 类型，只产生一个 token（整个字段的值作为一个token，而不是分词拆分为多个token）
- 定义一个自定义 normalizer，使用大写uppercase过滤器


```
PUT test_index_4
{
  "settings": {
    "analysis": {
      "normalizer": {
        "my_normalizer": {
          "type": "custom",
          "char_filter": [],
          "filter": ["uppercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "_doc": {
      "properties": {
        "foo": {
          "type": "keyword",
          "normalizer": "my_normalizer"
        }
      }
    }
  }
}

# 插入数据
POST test_index_4/_doc/1
{
  "foo": "hello world"
}

POST test_index_4/_doc/2
{
  "foo": "Hello World"
}

POST test_index_4/_doc/3
{
  "foo": "hello elasticsearch"
}

# 搜索hello，结果为空，而不是3条！！ 
GET test_index_4/_search
{
  "query": {
    "match": {
      "foo": "hello"
    }
  }
}

# 搜索 hello world，结果2条，1 和 2
GET test_index_4/_search
{
  "query": {
    "match": {
      "foo": "hello world"
    }
  }
}
```

#### 其他字段
- coerce
    - 强制类型转换，把json中的值转为ES中字段的数据类型，譬如：把字符串"5"转为integer的5
    - coerce默认为 true
    - 如果coerce设置为 false，当json的值与es字段类型不匹配将会 rejected
    - 通过 "settings": { "index.mapping.coerce": false } 设置索引的 coerce
- enabled
    - 是否索引，默认为 true
    - 可以在_doc和字段两个粒度进行设置
- ignore_above
    - 设置能被索引的字段的长度
    - 超过这个长度，该字段将不被索引，所以无法搜索，但聚合的terms可以看到
- null\_value
    - 该字段定义遇到null值时的处理策略，默认为Null，即空值，此时ES会忽略该值
    - 通过设定该值可以设定字段为 null 时的默认值
- ignore_malformed
    - 当数据类型不匹配且 coerce 强制转换时,默认情况会抛出异常,并拒绝整个文档的插入
    - 若设置该参数为 true，则忽略该异常，并强制赋值，但是不会被索引，其他字段则照常
- norms
    - norms 存储各种标准化因子，为后续查询计算文档对该查询的匹配分数提供依据
    - norms 参数对**评分**很有用，但需要占用大量的磁盘空间
    - 如果不需要计算字段的评分，可以取消该字段 norms 的功能
- position\_increment\_gap
    - 与 proximity queries（近似查询）和 phrase queries（短语查询）有关
    - 默认值 100
- search_analyzer
    - 搜索分词器，查询时使用
    - 默认与 analyzer 一样
- similarity
    - 设置相关度算法，ES5.x 和 ES6.x 默认的算法为 BM25
    - 另外也可选择 classic 和 boolean
- store
    - store 的意思是：是否在 _source 之外在独立存储一份，默认值为 false
    - es在存储数据的时候把json对象存储到"\_source"字段里，"\_source"把所有字段保存为一份文档存储（读取需要1次IO），要取出某个字段则通过 source filtering 过滤
    - 当字段比较多或者内容比较多，并且不需要取出所有字段的时候，可以把特定字段的store设置为true单独存储（读取需要1次IO），同时在\_source设置exclude
    - 关于该字段的理解，参考： [es设置mapping store属性](https://blog.csdn.net/helllochun/article/details/52136954)
- term_vector
    - 与倒排索引相关


### [Dynamic Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html)

ES是依靠JSON文档的字段类型来实现自动识别字段类型，支持的类型如下：

| JSON 类型 | ES 类型                                    |
| ------- | ---------------------------------------- |
| null    | 忽略                                       |
| boolean | boolean                                  |
| 浮点类型    | float                                    |
| 整数      | long                                     |
| object  | object                                   |
| array   | 由第一个非 null 值的类型决定                        |
| string  | 匹配为日期则设为date类型（默认开启）；<br/>匹配为数字则设置为 float或long类型（默认关闭）；<br/>设为text类型，并附带keyword的子字段 |


举栗子
```
POST my_index/doc
{
  "username":"whirly",
  "age":22,
  "birthday":"1995-01-01"
}
GET my_index/_mapping

# 结果
{
  "my_index": {
    "mappings": {
      "doc": {
        "properties": {
          "age": {
            "type": "long"
          },
          "birthday": {
            "type": "date"
          },
          "username": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      }
    }
  }
}
```

#### 日期的自动识别
- dynamic\_date\_formats 参数为自动识别的日期格式，默认为 [ "strict\_date\_optional_time","yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z"]
- date\_detection可以关闭日期自动识别机制



```
# 自定义日期识别格式
PUT my_index
{
  "mappings": {
    "_doc": {
      "dynamic_date_formats": ["MM/dd/yyyy"]
    }
  }
}
# 关闭日期自动识别机制
PUT my_index
{
  "mappings": {
    "_doc": {
      "date_detection": false
    }
  }
}
```

#### 数字的自动识别
- 字符串是数字时，默认不会自动识别为整形，因为字符串中出现数字完全是合理的
- numeric_detection 参数可以开启字符串中数字的自动识别


### [Dynamic templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-templates.html)

允许根据ES自动识别的数据类型、字段名等来动态设定字段类型，可以实现如下效果：
- 所有字符串类型都设定为keyword类型，即不分词
- 所有以message开头的字段都设定为text类型，即分词
- 所有以long\_开头的字段都设定为long类型
- 所有自动匹配为double类型的都设定为float类型，以节省空间

#### Dynamic templates API

```
"dynamic_templates": [
    {
      "my_template_name": { 
        ...  match conditions ... 
        "mapping": { ... } 
      }
    },
    ...
]
```
匹配规则一般有如下几个参数：
- match\_mapping\_type 匹配ES自动识别的字段类型，如boolean，long，string等
- match, unmatch 匹配字段名
- match_pattern 匹配正则表达式
- path\_match, path\_unmatch 匹配路径




```
# double类型的字段设定为float以节省空间
PUT my_index
{
  "mappings": {
    "_doc": {
      "dynamic_templates": [
        {
          "integers": {
            "match_mapping_type": "double",
            "mapping": {
              "type": "float"
            }
          }
        }
      ]
    }
  }
}
```

##### 自定义Mapping的建议
1. 写入一条文档到ES的临时索引中，获取ES自动生成的Mapping
2. 修改步骤1得到的Mapping，自定义相关配置
3. 使用步骤2的Mapping创建实际所需索引


### [Index Template 索引模板](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/indices-templates.html)
- 索引模板，主要用于在新建索引时自动应用预先设定的配置，简化索引创建的操作步骤
    - 可以设定索引的setting和mapping
    - 可以有多个模板，根据order设置，order大的覆盖小的配置
- 索引模板API，endpoint为 \_template



```
# 创建索引模板，匹配 test-index-map 开头的索引
PUT _template/template_1
{
  "index_patterns": ["test-index-map*"],
  "order": 2,
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "doc": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "name": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date",
          "format": "YYYY/MM/dd HH:mm:ss"
        }
      }
    }
  }
}

# 插入一个文档
POST test-index-map_1/doc
{
  "name" : "小旋锋",
  "created_at": "2018/08/16 20:11:11"
}

# 获取该索引的信息，可以发现 settings 和 mappings 和索引模板里设置的一样
GET test-index-map_1

# 删除
DELETE /_template/template_1

# 查询
GET /_template/template_1
```



> 参考文档： 
> 1. elasticsearch 官方文档
> 2. [慕课网 Elastic Stack从入门到实践](https://coding.imooc.com/class/181.html)