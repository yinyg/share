# 前言

本文主要基于Elasticsearch 7.x及之后的版本进行介绍。

# 一、ES介绍
简单来说，**Elasticsearch (ES)** 是一个基于 Lucene 构建的开源、分布式、RESTful 风格的**搜索和数据分析引擎**。

## （一）常见的应用场景

1. **全文搜索：** 电商网站的商品搜索、GitHub 的代码搜索、新闻网站的检索。
2. **日志分析（ELK Stack）：** 结合 Logstash 和 Kibana，实时收集、分析服务器日志（这是 ES 目前最火的用法）。
3. **监控与度量：** 监控业务指标（如订单量波动）、基础设施状态。
4. **安全分析：** 识别网络攻击模式、异常登录行为。

## （二）核心概念对比

为了方便理解，我们可以将 ES 的概念与传统数据库进行类比：

**Elasticsearch 6.x及之前的版本**

| Elasticsearch (旧术语/逻辑) | 关系型数据库 (MySQL) | 说明                            |
| ---------------------- | -------------- | ----------------------------- |
| **Index (索引)**         | Database (数据库) | 存储数据的逻辑容器                     |
| **Type (类型)**          | Table (表)      | ES 7.x 之后已废弃，一个索引下只有一个类型      |
| **Document (文档)**      | Row (行)        | 最小的数据单元，JSON 格式               |
| **Field (字段)**         | Column (列)     | 文档中的属性                        |
| **Mapping (映射)**       | Schema (表结构)   | 定义字段类型（text, keyword, date 等） |

**Elasticsearch 7.x及之后的版本**

| Elasticsearch (旧术语/逻辑) | 关系型数据库 (MySQL) | 说明                            |
| ---------------------- | -------------- | ----------------------------- |
| **Index (索引)**         | Table (表)      | ES 7.x 之后，一个索引下只有一个类型         |
| **Document (文档)**      | Row (行)        | 最小的数据单元，JSON 格式               |
| **Field (字段)**         | Column (列)     | 文档中的属性                        |
| **Mapping (映射)**       | Schema (表结构)   | 定义字段类型（text, keyword, date 等） |

## （三）核心技术：倒排索引 (Inverted Index)

这是 ES 搜索飞快的秘密武器。
- **传统搜索：** 遍历每个文档，看里面有没有某个词（效率低）。
- **倒排索引：** 记录每个“词”出现在哪些“文档”里。
    - _文档1：_ "I love coding"
    - _文档2：_ "I love music"
    - **索引表：**
        - "coding" -> {文档1}
        - "music" -> {文档2}
        - "love" -> {文档1, 文档2} 当你搜 "love" 时，它直接告诉你去读文档 1 和 2，而不需要扫描全文。

## （四）常用字段类型

| 字段类型         | 是否分词 | 倒排索引 | 是否支持排序、聚合            | 是否支持范围查询 | 说明                                                        |
| ------------ | ---- | ---- | -------------------- | -------- | --------------------------------------------------------- |
| text         | 是    | 分词后  | 否（默认不支持，可强制开启，但是不建议） | 否        | 用于全文检索                                                    |
| keyword      | 否    | 完整值  | 是                    | 否        | 用于精确匹配、聚合和排序                                              |
| integer      | 否    | 完整值  | 是                    | 是        | 可以理解为特殊的keyword                                           |
| long         | 否    | 完整值  | 是                    | 是        | 可以理解为特殊的keyword                                           |
| double       | 否    | 完整值  | 是                    | 是        | 可以理解为特殊的keyword                                           |
| scaled_float | 否    | 完整值  | 是                    | 是        | 高精度类型，底层是long类型，因此需要指定**精度**（scaling_factor），超过该精度的部分会被舍弃 |
| date         | 否    | 完整值  | 是                    | 是        | 时间类型                                                      |
| nested       |      |      |                      |          | 用于存储对象，以json的格式存储，内部字段通过指定不同的类型，也可以支持倒排索引、排序、聚合、范围查询      |

**更多类型参考官方文档**: https://www.elastic.co/guide/en/elasticsearch/reference/7.17/mapping-types.html

## （五）常用设置

### 1. 分片数量 (Number of Shards)


分片是 ES 实现**分布式存储**的基石。一个索引可以被拆分成多个物理上的分片。
- **本质：** 每个分片本质上都是一个完整的 **Lucene 索引**。
- **为什么要分片？**
    - **横向扩展：** 如果你的数据量是 1TB，一台服务器存不下，通过分片可以将数据分布到多台服务器上。
    - **并行计算：** 查询请求会同时在多个分片上运行，极大提高了搜索速度。
- **设定建议：**
    - **不可轻易修改：** 在 ES 7.x 之前，主分片数量在索引创建后**不可更改**（若要改必须重建索引/Reindex）。
    - **容量建议：** 单个分片的大小建议控制在 **20GB 到 50GB** 之间。分片过多会增加 Master 节点的负担，分片过大则会导致故障恢复缓慢。

### 2. 副本数量 (Number of Replicas)

副本是主分片（Primary Shard）的完整拷贝。
- **核心作用：**
    1. **高可用（HA）：** 如果某个节点宕机，副本分片会自动升级为主分片，保证数据不丢失、服务不中断。
    2. **提升吞吐量：** 副本分片可以处理**查询请求**。如果你的搜索压力很大，增加副本数可以分担查询负载。
- **特性：**
    - **动态修改：** 副本数量可以随时通过 API 修改，无需重启索引。
    - **存储代价：** 副本会占用双倍（或更多）的磁盘空间，并增加写入时的延迟（因为数据需要同步到副本）。
- **计算公式：** 总分片数 = 主分片数 × (1 + 副本数)。
- 
### 3. 最大查询窗口 (index.max_result_window)

这是一个安全限制参数，默认值是 **10,000**。
- **定义：** 它限制了 `from + size` 查询所能获取的最大文档排名位置。
- **为什么限制 10,000？**
    - **深度分页瓶颈：** 在分布式搜索中，如果你要查第 10,001 到 10,020 条数据，协调节点必须从**每个分片**中各取前 10,020 条数据，然后在内存中进行汇总、全局排序，最后只取最后 20 条。
    - **资源消耗：** 随着 `from` 的增加，占用的 CPU 和内存呈几何级数增长。限制 10,000 是为了防止用户因错误的深度分页请求直接把集群内存拖垮（OOM）。
- **调整建议：**
    - **不建议强行调大：** 尽管可以通过 `_settings` API 调大，但这是治标不治本。
    - **替代方案：** 如果需要深度分页，请使用 `search_after` 或 `Scroll API`（如前文所述）。

# 二、操作索引API

## 1. 创建索引

```json
PUT /my-index-000001
```

## 2. 创建索引（包含索引设置&索引结构（Mapping））

```json
PUT /my-index-000001
{
  "settings": {
    "index": {
      "number_of_shards": 3, // 分片数量
      "number_of_replicas": 2, // 副本数量
      "max_result_window": 100000 // 最大窗口
    }
  },
  "mappings": {
    "properties": {
	  "skuId": { "type": "keyword" },
      "skuName": {
	      "type": "text",
	      "analyzer": "ik_max_word" // 分词器
	  },
      "length": { "type": "integer" },
      "createTime": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "status": { "type": "integer" },
      "stockNum": { "type": "integer" },
      "categotyId": { "type": "long" }
    }
  }
}
```

## 3. 更新索引设置

```json
PUT /my-index-000001/_settings
{
  "index" : {
    "max_result_window" : 1000000
  }
}
```

## 4. 更新索引结构（Mapping）

```json
PUT /my-index-000001/_mapping
{
  "properties": {
    "model": { "type": "keyword" }
  }
}
```

## 5. 同时更新索引设置&更新索引结构（Mapping）

```json
PUT /my-index-000001
{
  "settings": {
    "index": {
      "max_result_window" : 1000000
    }
  },
  "mappings": {
    "properties": {
	  "model": { "type": "keyword" }
    }
  }
}
```

**特别注意，不支持修改已存在的索引字段。**

**参考文档**: https://www.elastic.co/guide/en/elasticsearch/reference/7.17/indices.html

# 三、操作文档API

**对文档的操作，一般是在程序中完成，如果没有直接操作文档的需求，可以跳过这一节，直接阅读第四节查询API。**

## （一）文档总数

### 1. 不带条件

```json
GET /my-index-000001/_count
```

### 2. 带条件

```json
POST /my-index-000001/_count
{
  "query": {
    "term": {
      "status": 1
    }
  }
}
```

## （二）新增文档

1. POST /my-index-000001/_doc/: 新增文档，不指定ID，自动生成ID
2. PUT /my-index-000001/_doc/<_id>: 新增文档，并指定ID
3. PUT /my-index-000001/_create/<_id>: 新增文档，并指定ID
4. POST /my-index-000001/_create/<_id>: 新增文档，并指定ID

**以上API的参数为JSON格式的文档数据。**

## （三）更新文档

### 1. 根据ID覆盖原文档

```json
PUT /my-index-000001/_doc/<_id>
{
  // 完整文档数据
}
```

### 2. 根据ID+脚本更新文档

```json
POST /my-index-000001/_doc/<_id>
{
  "script" : {
    "source": "ctx._source.skuName = ctx._source.skuName + ctx._source.skuId",
    "lang": "painless"
  }
}
```

### 3. 根据条件更新文档

```json
POST /my-index-000001/_update_by_query
{
  "query": { 
    "term": {
      "status: 1
    }
  },
  "script" : {
    "source": "ctx._source.skuName = ctx._source.skuName + ctx._source.skuId",
    "lang": "painless"
  }
}
```

## （四）删除文档

### 1. 根据ID删除文档

```json
DELETE /my-index-000001/_doc/<_id>
```

### 2. 根据条件删除文档

```json
POST /my-index-000001/_delete_by_query
{
  "query": {
    "term": {
      "skuId": "1234567890"
    }
  }
}
```

## （五）后台处理

如果操作比较耗时，可以在URL后面加上wait_for_completion=false参数，命令会直接返回一个taskId，然后根据这个taskId去查询命令执行状态。

**Example**:

```json
POST /my-index-000001/_update_by_query?wait_for_completion=false
{
  "query": { 
    "term": {
      "status: 1
    }
  },
  "script" : {
    "source": "ctx._source.skuName = ctx._source.skuName + ctx._source.skuId",
    "lang": "painless"
  }
}
```

```json
GET /_tasks/<_taskId> // taskId是上面的命令的返回结果
```

# 四、查询API

## （一）根据ID查询

```json
GET /my-index-000001/_doc/<_id>
```

## （二）根据条件查询

```json
GET /my-index-000001/_search
{
  // 条件
}
```

### 1. 精确查询

```json
GET /my-index-000001/_search
{
  "query": {
    "trem": {
	    "skuId": "1234567890"
    }
  }
}
```

### 2. 分词查询

```json
GET /my-index-000001/_search
{
  "query": {
    "match": {
	    "skuName": "华为"
    }
  }
}
```

### 3. 范围查询

```json
GET /my-index-000001/_search
{
  "query": {
    {
      "range": {
        "createTime": {
          "gte": "2025-01-01 00:00:00",
          "lte": "2025-12-31 23:59:59"
        }
      }
    }
  }
}
```

### 4. 多条件查询

```json
GET /my-index-000001/_search?track_total_hits=true
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
	        "skuName": "华为"
          }
        }
      ],
      "should": [
        {
          "match": {
	        "skuName": "手机"
          }
        },
        {
          "match": {
	        "skuName": "平板"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": 1
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "stockNum": 0
          }
        }
      ]
    }
  },
  "from": 20,
  "size": 10
}
```

**解析**:

- **bool**表示一个复合查询；
- **must**中的条件必须全部满足，默认参与评分，评分越高的文档在结果中越靠前；
- **should**可以理解为**OR**，多个条件满足1个即可，也可以**minimum_should_match**参数指定满足条件的个数；
- **filter**中条件也必须全部满足，不参与评分；
- **must_not**中的条件取反，如上面的这个查询，表示库存不等于0；
- 如果只有一个条件，也可以不指定bool，must、should、filter，默认为**must**;
- **from**表示查询偏移量，上面的查询表示查询第20条数据后面的数据；
- **size**表示查询多少条数据；
- URL中的**track_total_hits=true**表示返回准确的命中总数，如果是亿级以上，开启 track_total_hits 可能会让查询响应时间（Latency）明显增加。

### 5. 深分页查询

...待更新...

**更多API参考文档**: https://www.elastic.co/guide/en/elasticsearch/reference/7.17/search.html

# 五、聚合API

**这里只介绍简单的，无需做计算的聚合。**

**入参示例**:

```json
GET /my-index-000001/_search
{
  "query": {
    // 条件
  }
  "size": 0, // 如果只想查询聚合结果，不需要命中的文档数据，可以指定size=0
  "aggs": {
    "aggs_categortId": { // 自定义的聚合名称
      "terms": {
        "field": "categortId", // 必须是 keyword 类型
        "size": 5 // 只返回前 5 名
      }
    }
  }
}
```

**出参示例**:

```json
{
  "hits": { "total": { "value": 100, "relation": "eq" }, "hits": [] },
  "aggregations": {
    "aggs_categortId": {      // 对应入参定义的名字
      "buckets": [           // 桶数组，每个对象代表一个堆
        {
          "key": 1, // 该桶的分类值
          "doc_count": 45 // 属于类目ID为1的文档数量
        },
        {
          "key": 2,
          "doc_count": 35
        },
        {
          "key": 3,
          "doc_count": 20
        }
      ]
    }
  }
}
```

# 六、Elasticsearch 7.x 官方文档

https://www.elastic.co/guide/en/elasticsearch/reference/7.17/index.html
