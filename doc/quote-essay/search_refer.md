### 各种ES查询在Java中的实现

这篇博文的主题是ES的查询，因此我整理了尽可能齐全的ES查询场景，形成下面的图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDWbRDs6SZzeaiajIGBDCVcB42Ab2s07g09ADtiasp4zxTbSYQWyFgyiacYF5276iaMianxUOjU6ptibkicQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

本文基于elasticsearch 7.13.2版本，es从7.0以后，发生了很大的更新。7.3以后，已经不推荐使用TransportClient这个client，取而代之的是Java High Level REST Client。

#### 测试使用的数据示例

首先是，Mysql中的部分测试数据：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDWbRDs6SZzeaiajIGBDCVcBjfLdMiaZ05zSeusRWTkzdYd6B7Ont4yicvCFYB9MlWBnuibP5Bf3ibGukQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Mysql中的一行数据在ES中以一个文档形式存在：

```json
{
  "_index" : "person",
  "_type" : "_doc",
  "_id" : "4",
  "_score" : 1.0,
  "_source" : {
    "address" : "峨眉山",
    "modifyTime" : "2021-06-29 19:46:25",
    "createTime" : "2021-05-14 11:37:07",
    "sect" : "峨嵋派",
    "sex" : "男",
    "skill" : "降龙十八掌",
    "name" : "宋青书",
    "id" : 4,
    "power" : 50,
    "age" : 21
  }
}
```

简单梳理了一下ES JavaAPI的相关体系，感兴趣的可以自己研读一下源码。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDWbRDs6SZzeaiajIGBDCVcBYO7g2JCeE7o3vgUYCxSRDPV1hwOxMVHYqrXDM44U9cPoek6L52icQBw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

接下来，我们用十几个实例，迅速上手ES的查询操作，每个示例将提供SQL语句、ES语句和Java代码。

#### 1 词条查询

所谓词条查询，也就是ES不会对查询条件进行分词处理，只有当词条和查询字符串完全匹配时，才会被查询到。

##### 1.1 等值查询-term

等值查询，即筛选出一个字段等于特定值的所有记录。

SQL：

```sql
select * from person where name = '张无忌';
```

而使用ES查询语句却很不一样（注意查询字段带上keyword）：

```json
GET /person/_search
{
 "query": {
  "term": {
   "name.keyword": {
    "value": "张无忌",
    "boost": 1.0
   }
  }
 }
}
```

ElasticSearch 5.0以后，string类型有重大变更，移除了string类型，string字段被拆分成两种新的数据类型: text用于全文搜索的，而keyword用于关键词搜索。

查询结果：

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : { // 分片信息
    "total" : 1, // 总计分片数
    "successful" : 1, // 查询成功的分片数
    "skipped" : 0, // 跳过查询的分片数
    "failed" : 0  // 查询失败的分片数
  },
  "hits" : { // 命中结果
    "total" : {
      "value" : 1, // 数量
      "relation" : "eq"  // 关系：等于
    },
    "max_score" : 2.8526313,  // 最高分数
    "hits" : [
      {
        "_index" : "person", // 索引
        "_type" : "_doc", // 类型
        "_id" : "1",
        "_score" : 2.8526313,
        "_source" : {
          "address" : "光明顶",
          "modifyTime" : "2021-06-29 16:48:56",
          "createTime" : "2021-05-14 16:50:33",
          "sect" : "明教",
          "sex" : "男",
          "skill" : "九阳神功",
          "name" : "张无忌",
          "id" : 1,
          "power" : 99,
          "age" : 18
        }
      }
    ]
  }
}
```

Java中构造ES请求的方式：（后续例子中只保留SearchSourceBuilder的构建语句）

```java
/**
 * term精确查询
 *
 * @throws IOException
 */

@Autowired
private RestHighLevelClient client;

@Test
public void queryTerm() throws IOException {
 // 根据索引创建查询请求
    SearchRequest searchRequest = new SearchRequest("person");
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    // 构建查询语句
    searchSourceBuilder.query(QueryBuilders.termQuery("name.keyword", "张无忌"));
    System.out.println("searchSourceBuilder=====================" + searchSourceBuilder);
    searchRequest.source(searchSourceBuilder);
    SearchResponse response = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(JSONObject.toJSON(response));
}
```

仔细观察查询结果，会发现ES查询结果中会带有`_score`这一项，ES会根据结果匹配程度进行评分。打分是会耗费性能的，如果确认自己的查询不需要评分，就设置查询语句关闭评分：

```json
GET /person/_search
{
 "query": {
  "constant_score": {
   "filter": {
    "term": {
     "sect.keyword": {
      "value": "张无忌",
      "boost": 1.0
     }
    }
   },
   "boost": 1.0
  }
 }
}
```

Java构建查询语句：

```
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 这样构造的查询条件，将不进行score计算，从而提高查询效率
searchSourceBuilder.query(QueryBuilders.constantScoreQuery(QueryBuilders.termQuery("sect.keyword", "明教")));
```

##### 1.2 多值查询-terms

多条件查询类似Mysql里的IN查询，例如：

```sql
select * from persons where sect in('明教','武当派');
```

ES查询语句：

```json
GET /person/_search
{
 "query": {
  "terms": {
   "sect.keyword": [
    "明教",
    "武当派"
   ],
   "boost": 1.0
  }
 }
}
```

Java实现：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.termsQuery("sect.keyword", Arrays.asList("明教", "武当派")));
}
```

##### 1.3 范围查询-range

范围查询，即查询某字段在特定区间的记录。

SQL:

```sql
select * from pesons where age between 18 and 22;
```

ES查询语句：

```json
GET /person/_search
{
 "query": {
  "range": {
   "age": {
    "from": 10,
    "to": 20,
    "include_lower": true,
    "include_upper": true,
    "boost": 1.0
   }
  }
 }
}
```

Java构建查询条件：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.rangeQuery("age").gte(10).lte(30));
}
```

##### 1.4 前缀查询-prefix

前缀查询类似于SQL中的模糊查询。

SQL：

```sql
select * from persons where sect like '武当%';
```

ES查询语句：

```json
{
 "query": {
  "prefix": {
   "sect.keyword": {
    "value": "武当",
    "boost": 1.0
   }
  }
 }
}
```

Java构建查询条件：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.prefixQuery("sect.keyword","武当"));
```

##### 1.5 通配符查询-wildcard

通配符查询，与前缀查询类似，都属于模糊查询的范畴，但通配符显然功能更强。

SQL：

```sql
select * from persons where name like '张%忌';
```

ES查询语句：

```json
{
 "query": {
  "wildcard": {
   "sect.keyword": {
    "wildcard": "张*忌",
    "boost": 1.0
   }
  }
 }
}
```

Java构建查询条件：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.wildcardQuery("sect.keyword","张*忌"));
```

#### 2 复合查询

前面的例子都是单个条件查询，在实际应用中，我们很有可能会过滤多个值或字段。先看一个简单的例子：

```sql
select * from persons where sex = '女' and sect = '明教';
```

这样的多条件等值查询，就要借用到组合过滤器了，其查询语句是：

```json
{
 "query": {
  "bool": {
   "must": [
    {
        "term": {
      "sex": {
       "value": "女",
       "boost": 1.0
      }
     }
    },
    {
     "term": {
      "sect.keywords": {
       "value": "明教",
       "boost": 1.0
      }
     }
    }
   ],
   "adjust_pure_negative": true,
   "boost": 1.0
  }
 }
}
```

Java构造查询语句：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.boolQuery()
        .must(QueryBuilders.termQuery("sex", "女"))
        .must(QueryBuilders.termQuery("sect.keyword", "明教"))
);
```

##### 2.1 布尔查询

布尔过滤器（bool filter）属于复合过滤器（compound filter）的一种 ，可以接受多个其他过滤器作为参数，并将这些过滤器结合成各式各样的布尔（逻辑）组合。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDWbRDs6SZzeaiajIGBDCVcBCuseFX65iaibEricLia080MH2WkH3yHwuibvgHbtRK2auIRYpxpT1AFXKvg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

bool 过滤器下可以有4种子条件，可以任选其中任意一个或多个。filter是比较特殊的，这里先不说。

```json
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : [],
   }
}
```

- must：所有的语句都必须匹配，与 ‘=’ 等价。
- must_not：所有的语句都不能匹配，与 ‘!=’ 或 not in 等价。
- should：至少有n个语句要匹配，n由参数控制。

**精度控制：**

所有 must 语句必须匹配，所有 `must_not` 语句都必须不匹配，但有多少 should 语句应该匹配呢？默认情况下，没有 should 语句是必须匹配的，只有一个例外：那就是当没有 must 语句的时候，至少有一个 should 语句必须匹配。

我们可以通过 `minimum_should_match` 参数控制需要匹配的 should 语句的数量，它既可以是一个绝对的数字，又可以是个百分比：

```json
GET /person/_search
{
 "query": {
  "bool": {
   "must": [
    {
     "term": {
      "sex": {
       "value": "女",
       "boost": 1.0
      }
     }
    }
   ],
   "should": [
    {
     "term": {
      "address.keyword": {
       "value": "峨眉山",
       "boost": 1.0
      }
     }
    },
    {
     "term": {
      "sect.keyword": {
       "value": "明教",
       "boost": 1.0
      }
     }
    }
   ],
   "adjust_pure_negative": true,
   "minimum_should_match": "1",
   "boost": 1.0
  }
 }
}
```

Java构建查询语句：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.boolQuery()
        .must(QueryBuilders.termQuery("sex", "女"))
        .should(QueryBuilders.termQuery("address.word", "峨眉山"))
        .should(QueryBuilders.termQuery("sect.keyword", "明教"))
        .minimumShouldMatch(1)
);
```

最后，看一个复杂些的例子，将bool的各子句联合使用：

```sql
select 
 *
from
 persons
where 
 sex = '女'
and
 age between 30 and 40
and 
 sect != '明教'
and 
 (address = '峨眉山' OR skill = '暗器')
```

用 Elasticsearch 来表示上面的 SQL 例子：

```json
GET /person/_search
{
 "query": {
  "bool": {
   "must": [
    {
     "term": {
      "sex": {
       "value": "女",
       "boost": 1.0
      }
     }
    },
    {
     "range": {
      "age": {
       "from": 30,
       "to": 40,
       "include_lower": true,
       "include_upper": true,
       "boost": 1.0
      }
     }
    }
   ],
   "must_not": [
    {
     "term": {
      "sect.keyword": {
       "value": "明教",
       "boost": 1.0
      }
     }
    }
   ],
   "should": [
    {
     "term": {
      "address.keyword": {
       "value": "峨眉山",
       "boost": 1.0
      }
     }
    },
    {
     "term": {
      "skill.keyword": {
       "value": "暗器",
       "boost": 1.0
      }
     }
    }
   ],
   "adjust_pure_negative": true,
   "minimum_should_match": "1",
   "boost": 1.0
  }
 }
}
```

用Java构建这个查询条件：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery()
        .must(QueryBuilders.termQuery("sex", "女"))
        .must(QueryBuilders.rangeQuery("age").gte(30).lte(40))
        .mustNot(QueryBuilders.termQuery("sect.keyword", "明教"))
        .should(QueryBuilders.termQuery("address.keyword", "峨眉山"))
        .should(QueryBuilders.rangeQuery("power.keyword").gte(50).lte(80))
        .minimumShouldMatch(1);  // 设置should至少需要满足几个条件

// 将BoolQueryBuilder构建到SearchSourceBuilder中
searchSourceBuilder.query(boolQueryBuilder);
```

##### 2.2 Filter查询

query和filter的区别：query查询的时候，会先比较查询条件，然后计算分值，最后返回文档结果；而filter是先判断是否满足查询条件，如果不满足会缓存查询结果（记录该文档不满足结果），满足的话，就直接缓存结果，filter不会对结果进行评分，能够提高查询效率。

filter的使用方式比较多样，下面用几个例子演示一下。

**方式一，单独使用：**

```json
{
 "query": {
  "bool": {
   "filter": [
    {
     "term": {
      "sex": {
       "value": "男",
       "boost": 1.0
      }
     }
    }
   ],
   "adjust_pure_negative": true,
   "boost": 1.0
  }
 }
}
```

单独使用时，filter与must基本一样，不同的是filter不计算评分，效率更高。

Java构建查询语句：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.boolQuery()
        .filter(QueryBuilders.termQuery("sex", "男"))
);
```

**方式二，和must、must_not同级，相当于子查询：**

```
select * from (select * from persons where sect = '明教')) a where sex = '女';
```

ES查询语句：

```json
{
 "query": {
  "bool": {
   "must": [
    {
     "term": {
      "sect.keyword": {
       "value": "明教",
       "boost": 1.0
      }
     }
    }
   ],
   "filter": [
    {
     "term": {
      "sex": {
       "value": "女",
       "boost": 1.0
      }
     }
    }
   ],
   "adjust_pure_negative": true,
   "boost": 1.0
  }
 }
}
```

Java：

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.boolQuery()
        .must(QueryBuilders.termQuery("sect.keyword", "明教"))
        .filter(QueryBuilders.termQuery("sex", "女"))
);
```

**方式三，将must、must_not置于filter下，这种方式是最常用的：**

```json
{
 "query": {
  "bool": {
   "filter": [
    {
     "bool": {
      "must": [
       {
        "term": {
         "sect.keyword": {
          "value": "明教",
          "boost": 1.0
         }
        }
       },
       {
        "range": {
         "age": {
          "from": 20,
          "to": 35,
          "include_lower": true,
          "include_upper": true,
          "boost": 1.0
         }
        }
       }
      ],
      "must_not": [
       {
        "term": {
         "sex.keyword": {
          "value": "女",
          "boost": 1.0
         }
        }
       }
      ],
      "adjust_pure_negative": true,
      "boost": 1.0
     }
    }
   ],
   "adjust_pure_negative": true,
   "boost": 1.0
  }
 }
}
```

Java:

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 构建查询语句
searchSourceBuilder.query(QueryBuilders.boolQuery()
        .filter(QueryBuilders.boolQuery()
                .must(QueryBuilders.termQuery("sect.keyword", "明教"))
                .must(QueryBuilders.rangeQuery("age").gte(20).lte(35))
                .mustNot(QueryBuilders.termQuery("sex.keyword", "女")))
);
```

#### 3 聚合查询

接下来，我们将用一些案例演示ES聚合查询。

##### 3.1 最值、平均值、求和

案例：查询最大年龄、最小年龄、平均年龄。

SQL：

```sql
select max(age) from persons;
```

ES：

```json
GET /person/_search
{
 "aggregations": {
  "max_age": {
   "max": {
    "field": "age"
   }
  }
 }
}
```

Java：

```java
@Autowired
private RestHighLevelClient client;

@Test
public void maxQueryTest() throws IOException {
 // 聚合查询条件
    AggregationBuilder aggBuilder = AggregationBuilders.max("max_age").field("age");
    SearchRequest searchRequest = new SearchRequest("person");
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    // 将聚合查询条件构建到SearchSourceBuilder中
    searchSourceBuilder.aggregation(aggBuilder);
    System.out.println("searchSourceBuilder----->" + searchSourceBuilder);

    searchRequest.source(searchSourceBuilder);
    // 执行查询，获取SearchResponse
    SearchResponse response = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(JSONObject.toJSON(response));
}
```

使用聚合查询，结果中默认只会返回10条文档数据（当然我们关心的是聚合的结果，而非文档）。返回多少条数据可以自主控制：

```json
GET /person/_search
{
 "size": 20,
 "aggregations": {
  "max_age": {
   "max": {
    "field": "age"
   }
  }
 }
}
```

而Java中只需增加下面一条语句即可：

```java
searchSourceBuilder.size(20);
```

与max类似，其他统计查询也很简单：

```java
AggregationBuilder minBuilder = AggregationBuilders.min("min_age").field("age");
AggregationBuilder avgBuilder = AggregationBuilders.avg("min_age").field("age");
AggregationBuilder sumBuilder = AggregationBuilders.sum("min_age").field("age");
AggregationBuilder countBuilder = AggregationBuilders.count("min_age").field("age");
```

##### 3.2 去重查询

案例：查询一共有多少个门派。

SQL：

```sql
select count(distinct sect) from persons;
```

ES：

```json
{
 "aggregations": {
  "sect_count": {
   "cardinality": {
    "field": "sect.keyword"
   }
  }
 }
}
```

Java：

```java
@Test
public void cardinalityQueryTest() throws IOException {
 // 创建某个索引的request
    SearchRequest searchRequest = new SearchRequest("person");
    // 查询条件
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    // 聚合查询
    AggregationBuilder aggBuilder = AggregationBuilders.cardinality("sect_count").field("sect.keyword");
    searchSourceBuilder.size(0);
    // 将聚合查询构建到查询条件中
    searchSourceBuilder.aggregation(aggBuilder);
    System.out.println("searchSourceBuilder----->" + searchSourceBuilder);

    searchRequest.source(searchSourceBuilder);
    // 执行查询，获取结果
    SearchResponse response = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(JSONObject.toJSON(response));
}
```

##### 3.3 分组聚合

###### 3.3.1 单条件分组

案例：查询每个门派的人数

SQL：

```sql
select sect,count(id) from mytest.persons group by sect;
```

ES：

```json
{
 "size": 0,
 "aggregations": {
  "sect_count": {
   "terms": {
    "field": "sect.keyword",
    "size": 10,
    "min_doc_count": 1,
    "shard_min_doc_count": 0,
    "show_term_doc_count_error": false,
    "order": [
     {
      "_count": "desc"
     },
     {
      "_key": "asc"
     }
    ]
   }
  }
 }
}
```

Java：

```java
SearchRequest searchRequest = new SearchRequest("person");
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
searchSourceBuilder.size(0);
// 按sect分组
AggregationBuilder aggBuilder = AggregationBuilders.terms("sect_count").field("sect.keyword");
searchSourceBuilder.aggregation(aggBuilder);
```

###### 3.3.2 多条件分组

案例：查询每个门派各有多少个男性和女性

SQL：

```sql
select sect,sex,count(id) from mytest.persons group by sect,sex;
```

ES：

```json
{
 "aggregations": {
  "sect_count": {
   "terms": {
    "field": "sect.keyword",
    "size": 10
   },
   "aggregations": {
    "sex_count": {
     "terms": {
      "field": "sex.keyword",
      "size": 10
     }
    }
   }
  }
 }
}
```

##### 3.4 过滤聚合

前面所有聚合的例子请求都省略了 query ，整个请求只不过是一个聚合。这意味着我们对全部数据进行了聚合，但现实应用中，我们常常对特定范围的数据进行聚合，例如下例。

**案例：查询明教中的最大年龄。这涉及到聚合与条件查询一起使用。**

SQL：

```sql
select max(age) from mytest.persons where sect = '明教';
```

ES：

```json
GET /person/_search
{
 "query": {
  "term": {
   "sect.keyword": {
    "value": "明教",
    "boost": 1.0
   }
  }
 },
 "aggregations": {
  "max_age": {
   "max": {
    "field": "age"
   }
  }
 }
}
```

Java：

```java
SearchRequest searchRequest = new SearchRequest("person");
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 聚合查询条件
AggregationBuilder maxBuilder = AggregationBuilders.max("max_age").field("age");
// 等值查询
searchSourceBuilder.query(QueryBuilders.termQuery("sect.keyword", "明教"));
searchSourceBuilder.aggregation(maxBuilder);
```

另外还有一些更复杂的查询例子。

**案例：查询0-20，21-40，41-60，61以上的各有多少人。**

SQL：

```sql
select 
 sum(case when age<=20 then 1 else 0 end) ageGroup1,
 sum(case when age >20 and age <=40 then 1 else 0 end) ageGroup2,
 sum(case when age >40 and age <=60 then 1 else 0 end) ageGroup3,
 sum(case when age >60 and age <=200 then 1 else 0 end) ageGroup4
from 
 mytest.persons;
```

ES：

```json
{
 "size": 0,
 "aggregations": {
  "age_avg": {
   "range": {
    "field": "age",
    "ranges": [
     {
      "from": 0.0,
      "to": 20.0
     },
     {
      "from": 21.0,
      "to": 40.0
     },
     {
      "from": 41.0,
      "to": 60.0
     },
     {
      "from": 61.0,
      "to": 200.0
     }
    ],
    "keyed": false
   }
  }
 }
}
```

查询结果：

```json
"aggregations" : {
  "age_avg" : {
    "buckets" : [
      {
        "key" : "0.0-20.0",
        "from" : 0.0,
        "to" : 20.0,
        "doc_count" : 3
      },
      {
        "key" : "21.0-40.0",
        "from" : 21.0,
        "to" : 40.0,
        "doc_count" : 13
      },
      {
        "key" : "41.0-60.0",
        "from" : 41.0,
        "to" : 60.0,
        "doc_count" : 4
      },
      {
        "key" : "61.0-200.0",
        "from" : 61.0,
        "to" : 200.0,
        "doc_count" : 1
      }
    ]
  }
}
```

以上是ElasticSearch查询的全部内容，丰富详实，堪比操作手册，强烈建议收藏！



### 熟悉Linux命令

#### 基本操作

###### Linux 关机,重启

```sh
# 关机
shutdown -h now

# 重启
shutdown -r now
```

###### 查看系统,CPU信息

```sh
# 查看系统内核信息
uname -a

# 查看系统内核版本
cat /proc/version

# 查看当前用户环境变量
env

cat /proc/cpuinfo

# 查看有几个逻辑cpu, 包括cpu型号
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c

# 查看有几颗cpu,每颗分别是几核
cat /proc/cpuinfo | grep physical | uniq -c

# 查看当前CPU运行在32bit还是64bit模式下, 如果是运行在32bit下也不代表CPU不支持64bit
getconf LONG_BIT

# 结果大于0, 说明支持64bit计算. lm指long mode, 支持lm则是64bit
cat /proc/cpuinfo | grep flags | grep ' lm ' | wc -l
```

###### 建立软连接

```sh
ln -s /usr/local/jdk1.8/ jdk
```

###### rpm相关

```sh
# 查看是否通过rpm安装了该软件
rpm -qa | grep 软件名
```

###### sshkey

```sh
# 创建sshkey
ssh-keygen -t rsa -C your_email@example.com

#id_rsa.pub 的内容拷贝到要控制的服务器的 home/username/.ssh/authorized_keys 中,如果没有则新建(.ssh权限为700, authorized_keys权限为600)
```

###### 命令重命名

```sh
# 在各个用户的.bash_profile中添加重命名配置
alias ll='ls -alF'
```

###### 同步服务器时间

```sh
sudo ntpdate -u ntp.api.bz
```

###### 后台运行命令

```sh
# 后台运行,并且有nohup.out输出
nohup xxx &

# 后台运行, 不输出任何日志
nohup xxx > /dev/null &

# 后台运行, 并将错误信息做标准输出到日志中 
nohup xxx >out.log 2>&1 &
```

###### 强制活动用户退出

```sh
# 命令来完成强制活动用户退出.其中TTY表示终端名称
pkill -kill -t [TTY]
```

###### 查看命令路径

```sh
which <命令>
```

###### 查看进程所有打开最大fd数

```sh
ulimit -n
```

###### 配置dns

```
vim /etc/resolv.conf
```

###### nslookup,查看域名路由表

```sh
nslookup google.com
```

###### last, 最近登录信息列表

```sh
# 最近登录的5个账号
last -n 5
```

###### 设置固定ip

```sh
ifconfig em1  192.168.5.177 netmask 255.255.255.0
```

###### 查看进程内加载的环境变量

```sh
# 也可以去 cd /proc 目录下, 查看进程内存中加载的东西
ps eww -p  XXXXX(进程号)
```

###### 查看进程树找到服务器进程

```sh
ps auwxf
```

###### 查看进程启动路径

```sh
cd /proc/xxx(进程号)
ls -all
# cwd对应的是启动路径
```

###### 添加用户, 配置sudo权限

```sh
# 新增用户
useradd 用户名
passwd 用户名

#增加sudo权限
vim /etc/sudoers
# 修改文件里面的
# root    ALL=(ALL)       ALL
# 用户名 ALL=(ALL)       ALL
```

###### 强制关闭进程名包含xxx的所有进程

```sh
ps aux|grep xxx | grep -v grep | awk '{print $2}' | xargs kill -9
```

#### 磁盘,文件,目录相关操作

###### vim操作

```sh
#normal模式下 g表示全局, x表示查找的内容, y表示替换后的内容
:%s/x/y/g

#normal模式下
0  # 光标移到行首(数字0)
$  # 光标移至行尾
shift + g # 跳到文件最后
gg # 跳到文件头

# 显示行号
:set nu

# 去除行号
:set nonu

# 检索
/xxx(检索内容)  # 从头检索, 按n查找下一个
?xxx(检索内容)  # 从尾部检索
```

###### 打开只读文件,修改后需要保存时(不用切换用户即可保存的方式)

```sh
# 在normal模式下
:w !sudo tee %
```

###### 查看磁盘, 文件目录基本信息

```sh
# 查看磁盘挂载情况
mount

# 查看磁盘分区信息
df

# 查看目录及子目录大小
du -H -h

# 查看当前目录下各个文件, 文件夹占了多少空间, 不会递归
du -sh *
```

###### wc命令

```sh
# 查看文件里有多少行
wc -l filename

# 看文件里有多少个word
wc -w filename

# 文件里最长的那一行是多少个字
wc -L filename

# 统计字节数
wc -c
```

#### 常用压缩, 解压缩命令

###### 压缩命令

```sh
tar czvf xxx.tar 压缩目录

zip -r xxx.zip 压缩目录
```

###### 解压缩命令

```sh
tar zxvf xxx.tar

# 解压到指定文件夹
tar zxvf xxx.tar -C /xxx/yyy/

unzip xxx.zip
```

#### 变更文件所属用户, 用户组

```
chown eagleye.eagleye xxx.log
```

#### cp, scp, mkdir

```sh
#复制
cp xxx.log

# 复制并强制覆盖同名文件
cp -f xxx.log

# 复制文件夹
cp -r xxx(源文件夹) yyy(目标文件夹)

# 远程复制
scp -P ssh端口 username@10.10.10.101:/home/username/xxx /home/xxx

# 级联创建目录
mkdir -p /xxx/yyy/zzz

# 批量创建文件夹, 会在test,main下都创建java, resources文件夹
mkdir -p src/{test,main}/{java,resources}
```

###### 比较两个文件

```sh
diff -u 1.txt 2.txt
```

###### 日志输出的字节数,可以用作性能测试

```sh
# 如果做性能测试, 可以每执行一次, 往日志里面输出 “.” , 这样日志中的字节数就是实际的性能测试运行的次数, 还可以看见实时速率.
tail -f xxx.log | pv -bt
```

###### 查看, 去除特殊字符

```sh
# 查看特殊字符
cat -v xxx.sh

# 去除特殊字符
sed -i 's/^M//g’ env.sh  去除文件的特殊字符, 比如^M:  需要这样输入: ctrl+v+enter
```

###### 处理因系统原因引起的文件中特殊字符的问题

```sh
# 可以转换为该系统下的文件格式
cat file.sh > file.sh_bak

# 先将file.sh中文件内容复制下来然后运行, 然后粘贴内容, 最后ctrl + d 保存退出
cat > file1.sh

# 在vim中通过如下设置文件编码和文件格式
:set fileencodings=utf-8 ，然后 w （存盘）一下即可转化为 utf8 格式，
:set fileformat=unix

# 在mac下使用dos2unix进行文件格式化
find . -name "*.sh" | xargs dos2unix
```

###### tee, 重定向的同时输出到屏幕

```sh
awk ‘{print $0}’ xxx.log | tee test.log
```

#### 检索相关

###### grep

```sh
# 反向匹配, 查找不包含xxx的内容
grep -v xxx

# 排除所有空行
grep -v '^/pre>

# 返回结果 2,则说明第二行是空行
grep -n “^$” 111.txt    

# 查询以abc开头的行
grep -n “^abc” 111.txt 

# 同时列出该词语出现在文章的第几行
grep 'xxx' -n xxx.log

# 计算一下该字串出现的次数
grep 'xxx' -c xxx.log

# 比对的时候，不计较大小写的不同
grep 'xxx' -i xxx.log
```

###### awk

```sh
# 以':' 为分隔符,如果第五域有user则输出该行
awk -F ':' '{if ($5 ~ /user/) print $0}' /etc/passwd 

# 统计单个文件中某个字符（串）(中文无效)出现的次数
awk -v RS='character' 'END {print --NR}' xxx.txt
```

###### find检索命令

```sh
# 在目录下找后缀是.mysql的文件
find /home/eagleye -name '*.mysql' -print

# 会从 /usr 目录开始往下找，找最近3天之内存取过的文件。
find /usr -atime 3 –print

# 会从 /usr 目录开始往下找，找最近5天之内修改过的文件。
find /usr -ctime 5 –print

# 会从 /doc 目录开始往下找，找jacky 的、文件名开头是 j的文件。  
find /doc -user jacky -name 'j*' –print

# 会从 /doc 目录开始往下找，找寻文件名是 ja 开头或者 ma开头的文件。
find /doc \( -name 'ja*' -o- -name 'ma*' \) –print

#  会从 /doc 目录开始往下找，找到凡是文件名结尾为 bak的文件，把它删除掉。-exec 选项是执行的意思，rm 是删除命令，{ } 表示文件名，“\;”是规定的命令结尾。 
find /doc -name '*bak' -exec rm {} \;
```

#### 网络相关

###### 查看什么进程使用了该端口

```sh
lsof -i:por
```

###### 获取本机ip地址

```sh
/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"
```

###### iptables

```sh
# 查看iptables状态
service iptables status

# 要封停一个ip
iptables -I INPUT -s ***.***.***.*** -j DROP

# 要解封一个IP，使用下面这条命令：
iptables -D INPUT -s ***.***.***.*** -j DROP

备注: 参数-I是表示Insert（添加），-D表示Delete（删除）。后面跟的是规则，INPUT表示入站，***.***.***.***表示要封停的IP，DROP表示放弃连接。

#开启9090端口的访问
/sbin/iptables -I INPUT -p tcp --dport 9090 -j ACCEPT 

# 防火墙开启、关闭、重启
/etc/init.d/iptables status
/etc/init.d/iptables start
/etc/init.d/iptables stop
/etc/init.d/iptables restart
```

###### nc命令, tcp调试利器

```sh
#给某一个endpoint发送TCP请求,就将data的内容发送到对端
nc 192.168.0.11 8000 < data.txt

#nc可以当做服务器，监听某个端口号,把某一次请求的内容存储到received_data里
nc -l 8000 > received_data

#上边只监听一次，如果多次可以加上-k参数
nc -lk 8000
```

###### tcpdump

```sh
# dump出本机12301端口的tcp包
tcpdump -i em1 tcp port 12301 -s 1500 -w abc.pcap
```

###### 跟踪网络路由路径

```sh
# traceroute默认使用udp方式, 如果是-I则改成icmp方式
traceroute -I www.163.com

# 从ttl第3跳跟踪
traceroute -M 3 www.163.com  

# 加上端口跟踪
traceroute -p 8080 192.168.10.11ss
# 显示本地打开的所有端口
ss -l 

# 显示每个进程具体打开的socket
ss -pl 

# 显示所有tcp socket
ss -t -a 

# 显示所有的UDP Socekt
ss -u -a 

# 显示所有已建立的SMTP连接
ss -o state established '( dport = :smtp or sport = :smtp )'  

# 显示所有已建立的HTTP连接 
ss -o state established '( dport = :http or sport = :http )'  

#找出所有连接X服务器的进程
ss -x src /tmp/.X11-unix/*  

#列出当前socket统计信息
ss -s 

解释：netstat是遍历/proc下面每个PID目录，ss直接读/proc/net下面的统计信息。所以ss执行的时候消耗资源以及消耗的时间都比netstat少很多
```

###### netstat

```sh
# 输出每个ip的连接数，以及总的各个状态的连接数
netstat -n | awk '/^tcp/ {n=split($(NF-1),array,":");if(n<=2)++S[array[(1)]];else++S[array[(4)]];++s[$NF];++N} END {for(a in S){printf("%-20s %s\n", a, S[a]);++I}printf("%-20s %s\n","TOTAL_IP",I);for(a in s) printf("%-20s %s\n",a, s[a]);printf("%-20s %s\n","TOTAL_LINK",N);}'

# 统计所有连接状态, 
# CLOSED：无连接是活动的或正在进行
# LISTEN：服务器在等待进入呼叫
# SYN_RECV：一个连接请求已经到达，等待确认
# SYN_SENT：应用已经开始，打开一个连接
# ESTABLISHED：正常数据传输状态
# FIN_WAIT1：应用说它已经完成
# FIN_WAIT2：另一边已同意释放
# ITMED_WAIT：等待所有分组死掉
# CLOSING：两边同时尝试关闭
# TIME_WAIT：主动关闭连接一端还没有等到另一端反馈期间的状态
# LAST_ACK：等待所有分组死掉
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'

# 查找较多time_wait连接
netstat -n|grep TIME_WAIT|awk '{print $5}'|sort|uniq -c|sort -rn|head -n20

#查看所有使用的端口
netstat -tunlp
```

#### 监控linux性能命令

###### top

按大写的 F 或 O 键，然后按 a-z 可以将进程按照相应的列进行排序, 然后回车。而大写的 R 键可以将当前的排序倒转。

```sh
PID 进程id
PPID 父进程id
RUSER Real user name
UID 进程所有者的用户id
USER 进程所有者的用户名
GROUP 进程所有者的组名
TTY 启动进程的终端名。不是从终端启动的进程则显示为 ?
PR 优先级
NI nice值。负值表示高优先级，正值表示低优先级
P 最后使用的CPU，仅在多CPU环境下有意义
%CPU 上次更新到现在的CPU时间占用百分比
TIME 进程使用的CPU时间总计，单位秒
TIME+ 进程使用的CPU时间总计，单位1/100秒
%MEM 进程使用的物理内存百分比
VIRT 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
SWAP 进程使用的虚拟内存中，被换出的大小，单位kb。
RES 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
CODE 可执行代码占用的物理内存大小，单位kb
DATA 可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
SHR 共享内存大小，单位kb
nFLT 页面错误次数
nDRT 最后一次写入到现在，被修改过的页面数。
S 进程状态。D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程
COMMAND 命令名/命令行
WCHAN 若该进程在睡眠，则显示睡眠中的系统函数名
Flags 任务标志，参考 sched.h
```

###### dmesg,查看系统日志

```sh
dmesg
```

###### iostat,磁盘IO情况监控

```sh
iostat -xz 1

# r/s, w/s, rkB/s, wkB/s：分别表示每秒读写次数和每秒读写数据量（千字节）。读写量过大，可能会引起性能问题。
# await：IO操作的平均等待时间，单位是毫秒。这是应用程序在和磁盘交互时，需要消耗的时间，包括IO等待和实际操作的耗时。如果这个数值过大，可能是硬件设备遇到了瓶颈或者出现故障。
# avgqu-sz：向设备发出的请求平均数量。如果这个数值大于1，可能是硬件设备已经饱和（部分前端硬件设备支持并行写入）。
# %util：设备利用率。这个数值表示设备的繁忙程度，经验值是如果超过60，可能会影响IO性能（可以参照IO操作平均等待时间）。如果到达100%，说明硬件设备已经饱和。
# 如果显示的是逻辑设备的数据，那么设备利用率不代表后端实际的硬件设备已经饱和。值得注意的是，即使IO性能不理想，也不一定意味这应用程序性能会不好，可以利用诸如预读取、写缓存等策略提升应用性能。
```

#### free,内存使用情况

```sh
free -m

eg:

     total       used       free     shared    buffers     cached
Mem:          1002        769        232          0         62        421
-/+ buffers/cache:          286        715
Swap:          1153          0       1153
```

第一部分Mem行:

```sh
total 内存总数: 1002M
used 已经使用的内存数: 769M
free 空闲的内存数: 232M
shared 当前已经废弃不用,总是0
buffers Buffer 缓存内存数: 62M
cached Page 缓存内存数:421M
```

**关系：total(1002M) = used(769M) + free(232M)**

第二部分(-/+ buffers/cache):

```sh
(-buffers/cache) used内存数：286M (指的第一部分Mem行中的used – buffers – cached)
(+buffers/cache) free内存数: 715M (指的第一部分Mem行中的free + buffers + cached)
```

可见-buffers/cache反映的是被程序实实在在吃掉的内存,而+buffers/cache反映的是可以挪用的内存总数.

第三部分是指交换分区

###### sar,查看网络吞吐状态

```sh
# sar命令在这里可以查看网络设备的吞吐率。在排查性能问题时，可以通过网络设备的吞吐量，判断网络设备是否已经饱和。

sar -n DEV 1

#
# sar命令在这里用于查看TCP连接状态，其中包括：
# active/s：每秒本地发起的TCP连接数，既通过connect调用创建的TCP连接；
# passive/s：每秒远程发起的TCP连接数，即通过accept调用创建的TCP连接；
# retrans/s：每秒TCP重传数量；
# TCP连接数可以用来判断性能问题是否由于建立了过多的连接，进一步可以判断是主动发起的连接，还是被动接受的连接。TCP重传可能是因为网络环境恶劣，或者服务器压力过大导致丢包
sar -n TCP,ETCP 1
```

###### vmstat, 给定时间监控CPU使用率, 内存使用, 虚拟内存交互, IO读写

```
# 2表示每2秒采集一次状态信息, 1表示只采集一次(忽略既是一直采集)

vmstat 2 1

eg:
r b swpd free buff cache si so bi bo in cs us sy id wa
1 0 0 3499840 315836 3819660 0 0 0 1 2 0 0 0 100 0
0 0 0 3499584 315836 3819660 0 0 0 0 88 158 0 0 100 0
0 0 0 3499708 315836 3819660 0 0 0 2 86 162 0 0 100 0
0 0 0 3499708 315836 3819660 0 0 0 10 81 151 0 0 100 0
1 0 0 3499732 315836 3819660 0 0 0 2 83 154 0 0 100 0
```

- r 表示运行队列(就是说多少个进程真的分配到CPU)，我测试的服务器目前CPU比较空闲，没什么程序在跑，当这个值超过了CPU数目，就会出现CPU瓶颈了。这个也和top的负载有关系，一般负载超过了3就比较高，超过了5就高，超过了10就不正常了，服务器的状态很危险。top的负载类似每秒的运行队列。如果运行队列过大，表示你的CPU很繁忙，一般会造成CPU使用率很高。
- b 表示阻塞的进程,这个不多说，进程阻塞，大家懂的。
- swpd 虚拟内存已使用的大小，如果大于0，表示你的机器物理内存不足了，如果不是程序内存泄露的原因，那么你该升级内存了或者把耗内存的任务迁移到其他机器。
- free 空闲的物理内存的大小，我的机器内存总共8G，剩余3415M。
- buff Linux/Unix系统是用来存储，目录里面有什么内容，权限等的缓存，我本机大概占用300多M
- cache cache直接用来记忆我们打开的文件,给文件做缓冲，我本机大概占用300多M(这里是Linux/Unix的聪明之处，把空闲的物理内存的一部分拿来做文件和目录的缓存，是为了提高 程序执行的性能，当程序使用内存时，buffer/cached会很快地被使用。)
- si 每秒从磁盘读入虚拟内存的大小，如果这个值大于0，表示物理内存不够用或者内存泄露了，要查找耗内存进程解决掉。我的机器内存充裕，一切正常。
- so 每秒虚拟内存写入磁盘的大小，如果这个值大于0，同上。
- bi 块设备每秒接收的块数量，这里的块设备是指系统上所有的磁盘和其他块设备，默认块大小是1024byte，我本机上没什么IO操作，所以一直是0，但是我曾在处理拷贝大量数据(2-3T)的机器上看过可以达到140000/s，磁盘写入速度差不多140M每秒
- bo 块设备每秒发送的块数量，例如我们读取文件，bo就要大于0。bi和bo一般都要接近0，不然就是IO过于频繁，需要调整。
- in 每秒CPU的中断次数，包括时间中断
- cs 每秒上下文切换次数，例如我们调用系统函数，就要进行上下文切换，线程的切换，也要进程上下文切换，这个值要越小越好，太大了，要考虑调低线程或者进程的数目,例如在apache和nginx这种web服务器中，我们一般做性能测试时会进行几千并发甚至几万并发的测试，选择web服务器的进程可以由进程或者线程的峰值一直下调，压测，直到cs到一个比较小的值，这个进程和线程数就是比较合适的值了。系统调用也是，每次调用系统函数，我们的代码就会进入内核空间，导致上下文切换，这个是很耗资源，也要尽量避免频繁调用系统函数。上下文切换次数过多表示你的CPU大部分浪费在上下文切换，导致CPU干正经事的时间少了，CPU没有充分利用，是不可取的。
- us 用户CPU时间，我曾经在一个做加密解密很频繁的服务器上，可以看到us接近100,r运行队列达到80(机器在做压力测试，性能表现不佳)。
- sy 系统CPU时间，如果太高，表示系统调用时间长，例如是IO操作频繁。
- id 空闲 CPU时间，一般来说，id + us + sy = 100,一般我认为id是空闲CPU使用率，us是用户CPU使用率，sy是系统CPU使用率。
- wt 等待IO CPU时间。

### SpringBoot自带的工具类

#### 断言

1. 断言是一个逻辑判断，用于检查不应该发生的情况
2. Assert 关键字在 JDK1.4 中引入，可通过 JVM 参数`-enableassertions`开启
3. SpringBoot 中提供了 Assert 断言工具类，通常用于数据合法性检查

```java
// 要求参数 object 必须为非空（Not Null），否则抛出异常，不予放行  
// 参数 message 参数用于定制异常信息。  
void notNull(Object object, String message)  
// 要求参数必须空（Null），否则抛出异常，不予『放行』。  
// 和 notNull() 方法断言规则相反  
void isNull(Object object, String message)  
// 要求参数必须为真（True），否则抛出异常，不予『放行』。  
void isTrue(boolean expression, String message)  
// 要求参数（List/Set）必须非空（Not Empty），否则抛出异常，不予放行  
void notEmpty(Collection collection, String message)  
// 要求参数（String）必须有长度（即，Not Empty），否则抛出异常，不予放行  
void hasLength(String text, String message)  
// 要求参数（String）必须有内容（即，Not Blank），否则抛出异常，不予放行  
void hasText(String text, String message)  
// 要求参数是指定类型的实例，否则抛出异常，不予放行  
void isInstanceOf(Class type, Object obj, String message)  
// 要求参数 `subType` 必须是参数 superType 的子类或实现类，否则抛出异常，不予放行  
void isAssignable(Class superType, Class subType, String message)  
```

#### 对象、数组、集合

##### ObjectUtils

1. 获取对象的基本信息

```java
// 获取对象的类名。参数为 null 时，返回字符串："null"   
String nullSafeClassName(Object obj)  
// 参数为 null 时，返回 0  
int nullSafeHashCode(Object object)  
// 参数为 null 时，返回字符串："null"  
String nullSafeToString(boolean[] array)  
// 获取对象 HashCode（十六进制形式字符串）。参数为 null 时，返回 0   
String getIdentityHexString(Object obj)  
// 获取对象的类名和 HashCode。 参数为 null 时，返回字符串：""   
String identityToString(Object obj)  
// 相当于 toString()方法，但参数为 null 时，返回字符串：""  
String getDisplayString(Object obj)  
```

1. 判断工具

```java
// 判断数组是否为空  
boolean isEmpty(Object[] array)  
// 判断参数对象是否是数组  
boolean isArray(Object obj)  
// 判断数组中是否包含指定元素  
boolean containsElement(Object[] array, Object element)  
// 相等，或同为 null时，返回 true  
boolean nullSafeEquals(Object o1, Object o2)  
/*  
判断参数对象是否为空，判断标准为：  
   Optional: Optional.empty()  
      Array: length == 0  
CharSequence: length == 0  
 Collection: Collection.isEmpty()  
        Map: Map.isEmpty()  
*/  
boolean isEmpty(Object obj)  
```

1. 其他工具方法

```java
// 向参数数组的末尾追加新元素，并返回一个新数组  
<A, O extends A> A[] addObjectToArray(A[] array, O obj)  
// 原生基础类型数组 --> 包装类数组  
Object[] toObjectArray(Object source)  
```

##### StringUtils

1. 字符串判断工具

```java
// 判断字符串是否为 null，或 ""。注意，包含空白符的字符串为非空  
boolean isEmpty(Object str)  
// 判断字符串是否是以指定内容结束。忽略大小写  
boolean endsWithIgnoreCase(String str, String suffix)  
// 判断字符串是否已指定内容开头。忽略大小写  
boolean startsWithIgnoreCase(String str, String prefix)   
// 是否包含空白符  
boolean containsWhitespace(String str)  
// 判断字符串非空且长度不为 0，即，Not Empty  
boolean hasLength(CharSequence str)  
// 判断字符串是否包含实际内容，即非仅包含空白符，也就是 Not Blank  
boolean hasText(CharSequence str)  
// 判断字符串指定索引处是否包含一个子串。  
boolean substringMatch(CharSequence str, int index, CharSequence substring)  
// 计算一个字符串中指定子串的出现次数  
int countOccurrencesOf(String str, String sub)  
```

1. 字符串操作工具

```java
// 查找并替换指定子串  
String replace(String inString, String oldPattern, String newPattern)  
// 去除尾部的特定字符  
String trimTrailingCharacter(String str, char trailingCharacter)   
// 去除头部的特定字符  
String trimLeadingCharacter(String str, char leadingCharacter)  
// 去除头部的空白符  
String trimLeadingWhitespace(String str)  
// 去除头部的空白符  
String trimTrailingWhitespace(String str)  
// 去除头部和尾部的空白符  
String trimWhitespace(String str)  
// 删除开头、结尾和中间的空白符  
String trimAllWhitespace(String str)  
// 删除指定子串  
String delete(String inString, String pattern)  
// 删除指定字符（可以是多个）  
String deleteAny(String inString, String charsToDelete)  
// 对数组的每一项执行 trim() 方法  
String[] trimArrayElements(String[] array)  
// 将 URL 字符串进行解码  
String uriDecode(String source, Charset charset)  
```

1. 路径相关工具方法

```java
// 解析路径字符串，优化其中的 “..”   
String cleanPath(String path)  
// 解析路径字符串，解析出文件名部分  
String getFilename(String path)  
// 解析路径字符串，解析出文件后缀名  
String getFilenameExtension(String path)  
// 比较两个两个字符串，判断是否是同一个路径。会自动处理路径中的 “..”   
boolean pathEquals(String path1, String path2)  
// 删除文件路径名中的后缀部分  
String stripFilenameExtension(String path)   
// 以 “. 作为分隔符，获取其最后一部分  
String unqualify(String qualifiedName)  
// 以指定字符作为分隔符，获取其最后一部分  
String unqualify(String qualifiedName, char separator)  
```

##### CollectionUtils

1. 集合判断工具

```java
// 判断 List/Set 是否为空  
boolean isEmpty(Collection<?> collection)  
// 判断 Map 是否为空  
boolean isEmpty(Map<?,?> map)  
// 判断 List/Set 中是否包含某个对象  
boolean containsInstance(Collection<?> collection, Object element)  
// 以迭代器的方式，判断 List/Set 中是否包含某个对象  
boolean contains(Iterator<?> iterator, Object element)  
// 判断 List/Set 是否包含某些对象中的任意一个  
boolean containsAny(Collection<?> source, Collection<?> candidates)  
// 判断 List/Set 中的每个元素是否唯一。即 List/Set 中不存在重复元素  
boolean hasUniqueObject(Collection<?> collection)  
```

1. 集合操作工具

```java
// 将 Array 中的元素都添加到 List/Set 中  
<E> void mergeArrayIntoCollection(Object array, Collection<E> collection)    
// 将 Properties 中的键值对都添加到 Map 中  
<K,V> void mergePropertiesIntoMap(Properties props, Map<K,V> map)  
// 返回 List 中最后一个元素  
<T> T lastElement(List<T> list)    
// 返回 Set 中最后一个元素  
<T> T lastElement(Set<T> set)   
// 返回参数 candidates 中第一个存在于参数 source 中的元素  
<E> E findFirstMatch(Collection<?> source, Collection<E> candidates)  
// 返回 List/Set 中指定类型的元素。  
<T> T findValueOfType(Collection<?> collection, Class<T> type)  
// 返回 List/Set 中指定类型的元素。如果第一种类型未找到，则查找第二种类型，以此类推  
Object findValueOfType(Collection<?> collection, Class<?>[] types)  
// 返回 List/Set 中元素的类型  
Class<?> findCommonElementType(Collection<?> collection)  
```

#### 文件、资源、IO 流

##### FileCopyUtils

1. 输入

```java
// 从文件中读入到字节数组中  
byte[] copyToByteArray(File in)  
// 从输入流中读入到字节数组中  
byte[] copyToByteArray(InputStream in)  
// 从输入流中读入到字符串中  
String copyToString(Reader in)  
```

1. 输出

```java
// 从字节数组到文件  
void copy(byte[] in, File out)  
// 从文件到文件  
int copy(File in, File out)  
// 从字节数组到输出流  
void copy(byte[] in, OutputStream out)   
// 从输入流到输出流  
int copy(InputStream in, OutputStream out)   
// 从输入流到输出流  
int copy(Reader in, Writer out)  
// 从字符串到输出流  
void copy(String in, Writer out)  
```

##### ResourceUtils

1. 从资源路径获取文件

```java
// 判断字符串是否是一个合法的 URL 字符串。  
static boolean isUrl(String resourceLocation)  
// 获取 URL  
static URL getURL(String resourceLocation)   
// 获取文件（在 JAR 包内无法正常使用，需要是一个独立的文件）  
static File getFile(String resourceLocation)  
```

1. Resource

```java
// 文件系统资源 D:\...  
FileSystemResource  
// URL 资源，如 file://... http://...  
UrlResource  
// 类路径下的资源，classpth:...  
ClassPathResource  
// Web 容器上下文中的资源（jar 包、war 包）  
ServletContextResource  
// 判断资源是否存在  
boolean exists()  
// 从资源中获得 File 对象  
File getFile()  
// 从资源中获得 URI 对象  
URI getURI()  
// 从资源中获得 URI 对象  
URL getURL()  
// 获得资源的 InputStream  
InputStream getInputStream()  
// 获得资源的描述信息  
String getDescription()  
```

##### StreamUtils

1. 输入

```java
void copy(byte[] in, OutputStream out)  
int copy(InputStream in, OutputStream out)  
void copy(String in, Charset charset, OutputStream out)  
long copyRange(InputStream in, OutputStream out, long start, long end)  
```

1. 输出

```java
byte[] copyToByteArray(InputStream in)  
String copyToString(InputStream in, Charset charset)  
// 舍弃输入流中的内容  
int drain(InputStream in)   
```

#### 反射、AOP

##### ReflectionUtils

1. 获取方法

```java
// 在类中查找指定方法  
Method findMethod(Class<?> clazz, String name)   
// 同上，额外提供方法参数类型作查找条件  
Method findMethod(Class<?> clazz, String name, Class<?>... paramTypes)   
// 获得类中所有方法，包括继承而来的  
Method[] getAllDeclaredMethods(Class<?> leafClass)   
// 在类中查找指定构造方法  
Constructor<T> accessibleConstructor(Class<T> clazz, Class<?>... parameterTypes)   
// 是否是 equals() 方法  
boolean isEqualsMethod(Method method)   
// 是否是 hashCode() 方法   
boolean isHashCodeMethod(Method method)   
// 是否是 toString() 方法  
boolean isToStringMethod(Method method)   
// 是否是从 Object 类继承而来的方法  
boolean isObjectMethod(Method method)   
// 检查一个方法是否声明抛出指定异常  
boolean declaresException(Method method, Class<?> exceptionType)   
```

1. 执行方法

```java
// 执行方法  
Object invokeMethod(Method method, Object target)    
// 同上，提供方法参数  
Object invokeMethod(Method method, Object target, Object... args)   
// 取消 Java 权限检查。以便后续执行该私有方法  
void makeAccessible(Method method)   
// 取消 Java 权限检查。以便后续执行私有构造方法  
void makeAccessible(Constructor<?> ctor)   
```

1. 获取字段

```java
// 在类中查找指定属性  
Field findField(Class<?> clazz, String name)   
// 同上，多提供了属性的类型  
Field findField(Class<?> clazz, String name, Class<?> type)   
// 是否为一个 "public static final" 属性  
boolean isPublicStaticFinal(Field field)   
```

1. 设置字段

```java
// 获取 target 对象的 field 属性值  
Object getField(Field field, Object target)   
// 设置 target 对象的 field 属性值，值为 value  
void setField(Field field, Object target, Object value)   
// 同类对象属性对等赋值  
void shallowCopyFieldState(Object src, Object dest)  
// 取消 Java 的权限控制检查。以便后续读写该私有属性  
void makeAccessible(Field field)   
// 对类的每个属性执行 callback  
void doWithFields(Class<?> clazz, ReflectionUtils.FieldCallback fc)   
// 同上，多了个属性过滤功能。  
void doWithFields(Class<?> clazz, ReflectionUtils.FieldCallback fc,   
                 ReflectionUtils.FieldFilter ff)   
// 同上，但不包括继承而来的属性  
void doWithLocalFields(Class<?> clazz, ReflectionUtils.FieldCallback fc)   
```

##### AopUtils

1. 判断代理类型

```java
// 判断是不是 Spring 代理对象  
boolean isAopProxy()  
// 判断是不是 jdk 动态代理对象  
isJdkDynamicProxy()  
// 判断是不是 CGLIB 代理对象  
boolean isCglibProxy()  
```

1. 获取被代理对象的 class

```java
// 获取被代理的目标 class  
Class<?> getTargetClass()  
```

##### AopContext

1. 获取当前对象的代理对象

```java
Object currentProxy()
```

### 22条API设计的最佳实践

- 1. 对URL使用kebab-case（短横线小写隔开形式）
- 2. 参数使用camelCase（驼峰形式）
- 3. 指向集合的复数名称
- 4. URL以集合开始，以标识符结束
- 5. 让动词远离你的资源URL
- 6. 对非资源URL使用动词
- 7. JSON属性使用camelCase驼峰形式
- 8. 监控
- 9. 不要使用table_name作为资源名
- 10. 使用API设计工具
- 11. 使用简单序数作为版本
- 12. 在你的响应体中包括总资源数
- 13. 接受limit和offset参数
- 14. 获取字段查询参数
- 15. 不要在URL中通过认证令牌
- 16. 验证内容类型
- 17. 对CRUD函数使用HTTP方法
- 18. 在嵌套资源的URL中使用关系
- 19. CORS（跨源资源共享）
- 20. 安全
- 21. 错误
- 22. 黄金法则]

任何API设计都遵循一种叫做“面向资源设计”的原则：

- 资源：资源是数据的一部分，例如：用户
- 集合：一组资源称为集合，例如：用户列表
- URL：标识资源或集合的位置，例如：/user

#### 1. 对URL使用kebab-case（短横线小写隔开形式）

例如，如果你想要获得订单列表。

不应该：

```
/systemOrders或/system_orders
```

应该：

```
/system-orders
```

#### 2. 参数使用camelCase（驼峰形式）

例如，如果你想从一个特定的商店购买产品。

不应该：

```
/system-orders/{order_id}
```

或者：

```
/system-orders/{OrderId}
```

应该：

```
/system-orders/{orderId}
```

#### 3. 指向集合的复数名称

如果你想获得系统的所有用户。

不应该：

```
GET /user
```

或：

```
GET /User
```

应该：

```
GET /users
```

#### 4. URL以集合开始，以标识符结束

如果要保持概念的单一性和一致性。

不应该：

```
GET /shops/:shopId/category/:categoryId/price
```

这很糟糕，因为它指向的是一个属性而不是资源。

应该：

```
GET /shops/:shopId/或GET /category/:categoryId
```

#### 5. 让动词远离你的资源URL

不要在URL中使用动词来表达你的意图。相反，使用适当的HTTP方法来描述操作。

不应该：

```
POST /updateuser/{userId}
```

或：

```
GET /getusers
```

应该：

```
PUT /user/{userId}
```

#### 6. 对非资源URL使用动词

如果你有一个端点，它只返回一个操作。在这种情况下，你可以使用动词。例如，如果你想要向用户重新发送警报。

应该：

```
POST /alarm/245743/resend
```

请记住，这些不是我们的CRUD操作。相反，它们被认为是在我们的系统中执行特定工作的函数。

#### 7. JSON属性使用camelCase驼峰形式

如果你正在构建一个请求体或响应体为JSON的系统，那么属性名应该使用驼峰大小写。

不应该：

```
{
   user_name: "Mohammad Faisal"
   user_id: "1"
}
```

应该：

```
{
   userName: "Mohammad Faisal"
   userId: "1"
}
```

#### 8. 监控

RESTful HTTP服务必须实现/health和/version和/metricsAPI端点。他们将提供以下信息。

**/health**

用200 OK状态码响应对/health的请求。

**/version**

用版本号响应对/version的请求。

**/metrics**

这个端点将提供各种指标，如平均响应时间。

也强烈推荐使用/debug和/status端点。

#### 9. 不要使用table_name作为资源名

不要只使用表名作为资源名。从长远来看，这种懒惰是有害的。

不应该：

```
product_order
```

应该：

```
product-orders
```

这是因为公开底层体系结构不是你的目的。

#### 10. 使用API设计工具

有许多好的API设计工具用于编写好的文档，例如：

- API蓝图：https://apiblueprint.org/
- Swagger：https://swagger.io/

[![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfemsX2t6xK3VRGhlF8YzR9XFn6OeBkmpvKr0v1gZ4oBrjiaU89vzficqGYdvltmDlNoS9M5jNibMmwcA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

拥有良好而详细的文档可以为API使用者带来良好的用户体验。

#### 11. 使用简单序数作为版本

始终对API使用版本控制，并将其向左移动，使其具有最大的作用域。版本号应该是v1，v2等等。

应该：http://api.domain.com/v1/shops/3/products

始终在API中使用版本控制，因为如果API被外部实体使用，更改端点可能会破坏它们的功能。

#### 12. 在你的响应体中包括总资源数

如果API返回一个对象列表，则响应中总是包含资源的总数。你可以为此使用total属性。

不应该：

```
{
  users: [ 
     ...
  ]
}
```

应该：

```
{
  users: [ 
     ...
  ],
  total: 34
}
```

#### 13. 接受limit和offset参数

在GET操作中始终接受limit和offset参数。

应该：

```
GET /shops?offset=5&limit=5
```

这是因为它对于前端的分页是必要的。

#### 14. 获取字段查询参数

返回的数据量也应该考虑在内。添加一个fields参数，只公开API中必需的字段。

例子：

只返回商店的名称，地址和联系方式。

```
GET /shops?fields=id,name,address,contact
```

在某些情况下，它还有助于减少响应大小。

#### 15. 不要在URL中通过认证令牌

这是一种非常糟糕的做法，因为url经常被记录，而身份验证令牌也会被不必要地记录。

不应该：

```
GET /shops/123?token=some_kind_of_authenticaiton_token
```

相反，通过头部传递它们：

```
Authorization: Bearer xxxxxx, Extra yyyyy
```

此外，授权令牌应该是短暂有效期的。

#### 16.验证内容类型

服务器不应该假定内容类型。例如，如果你接受application/x-www-form-urlencoded，那么攻击者可以创建一个表单并触发一个简单的POST请求。

因此，始终验证内容类型，如果你想使用默认的内容类型，请使用：

```
content-type: application/json
```

#### 17. 对CRUD函数使用HTTP方法

HTTP方法用于解释CRUD功能。

GET：检索资源的表示形式。

POST：创建新的资源和子资源。

PUT：更新现有资源。

PATCH：更新现有资源，它只更新提供的字段，而不更新其他字段。

DELETE：删除已存在的资源。

#### 18. 在嵌套资源的URL中使用关系

以下是一些实际例子：

- GET /shops/2/products：从shop 2获取所有产品的列表。
- GET /shops/2/products/31：获取产品31的详细信息，产品31属于shop 2。
- DELETE /shops/2/products/31：应该删除产品31，它属于商店2。
- PUT /shops/2/products/31：应该更新产品31的信息，只在resource-URL上使用PUT，而不是集合。
- POST /shops：应该创建一个新的商店，并返回创建的新商店的详细信息。在集合url上使用POST。

#### 19. CORS（跨源资源共享）

一定要为所有面向公共的API支持CORS（跨源资源共享）头部。

考虑支持CORS允许的“*”来源，并通过有效的OAuth令牌强制授权。

避免将用户凭证与原始验证相结合。

#### 20. 安全

在所有端点、资源和服务上实施HTTPS（tls加密）。

强制并要求所有回调url、推送通知端点和webhooks使用HTTPS。

#### 21. 错误

当客户端向服务发出无效或不正确的请求，或向服务传递无效或不正确的数据，而服务拒绝该请求时，就会出现错误，或者更具体地说，出现服务错误。

例子包括无效的身份验证凭证、不正确的参数、未知的版本id等。

- 当由于一个或多个服务错误而拒绝客户端请求时，一定要返回4xx HTTP错误代码。
- 考虑处理所有属性，然后在单个响应中返回多个验证问题。

#### 22. 黄金法则

如果您对API格式的决定有疑问，这些黄金规则可以帮助我们做出正确的决定。

- 扁平比嵌套好。
- 简单胜于复杂。
- 字符串比数字好。
- 一致性比定制更好。

### Maven 教程

- **一、为什么使用Maven这样的构建工具**
- **二、maven是什么**
- **四、第一个maven**
- **五、仓库和坐标**
- **六、依赖**
- **七、生命周期**
- **九、maven工程的依赖高级特性**
- **十、build配置**

#### 一、为什么使用Maven这样的构建工具

**① 一个项目就是一个工程**

如果项目非常庞大，就不适合使用package来划分模块，最好是每一个模块对应一个工程，利于分工协作。借助于maven就可以将一个项目拆分成多个工程

**② 项目中使用jar包，需要“复制”、“粘贴”项目的lib中**

同样的jar包重复的出现在不同的项目工程中，你需要做不停的复制粘贴的重复工作。借助于maven，可以将jar包保存在“仓库”中，不管在哪个项目只要使用引用即可就行。

**③ jar包需要的时候每次都要自己准备好或到官网下载**

借助于maven我们可以使用统一的规范方式下载jar包，规范

**④ jar包版本不一致的风险**

不同的项目在使用jar包的时候，有可能会导致各个项目的jar包版本不一致，导致未执行错误。借助于maven，所有的jar包都放在“仓库”中，所有的项目都使用仓库的一份jar包。

**⑤ 一个jar包依赖其他的jar包需要自己手动的加入到项目中**

FileUpload组件->IO组件，commons-fileupload-1.3.jar依赖于commons-io-2.0.1.jar

极大的浪费了我们导入包的时间成本，也极大的增加了学习成本。借助于maven，它会自动的将依赖的jar包导入进来。

#### 二、maven是什么

**① maven是一款服务于java平台的自动化构建工具**

make->Ant->Maven->Gradle

**② 构建**

构建定义：把动态的Web工程经过编译得到的编译结果部署到服务器上的整个过程。

编译：java源文件[.java]->编译->Classz字节码文件[.class]

部署：最终在sevlet容器中部署的不是动态web工程，而是编译后的文件

**③ 构建的各个环节**

- 清理clean：将以前编译得到的旧文件class字节码文件删除
- 编译compile：将java源程序编译成class字节码文件
- 测试test：自动测试，自动调用junit程序
- 报告report：测试程序执行的结果
- 打包package：动态Web工程打War包，java工程打jar包
- 安装install：Maven特定的概念-----将打包得到的文件复制到“仓库”中的指定位置
- 部署deploy：将动态Web工程生成的war包复制到Servlet容器下，使其可以运行

#### 四、第一个maven

**① 创建约定的目录结构** （maven工程必须按照约定的目录结构创建）

> 根目录：
>
> 工程名 
>
> |---src：源码 
>
> |---|---main:存放主程序 
>
> |---|---|---java：java源码文件 
>
> |---|---|---resource：存放框架的配置文件 
>
> |---|---test：存放测试程序 
>
> |---pop.xml：maven的核心配置文件

我们按照上面的文件夹目录结构手动创建一下，不用任何IDE环境（手动的其实最有助于我们理解maven）

##### 文件内容如下

在src/main/java/com/hzg/maven目录下新建文件Hello.java，内容如下

```java
package com.hzg.maven;
public class Hello {
  public String sayHello(String name){
    return "Hello "+name+"!";
  }
}
```

POM文件内容：

```xml
<?xml version="1.0" ?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.hzg.maven</groupId>
    <artifactId>Hello</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <name>Hello</name>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**② 常用maven命令**

- mvn clean：清理
- mvn compile：编译主程序
- mvn test-compile：编译测试程序
- mvn test：执行测试
- mvn package：打包
- mvn install：安装

**执行maven命令必须进入到pom.xml的目录中进行执行**

进入到项目的pom.xml目录之后，就可以执行啦。

**1、运行 mvn compile**

OK，运行完毕，你在pom.xml配置的依赖的包已经导入到仓库了，问题来了，**仓库默认的位置在哪？**

**仓库的默认位置：** c:\Usrs[登录当前系统的用户名].m2\repository

刚才执行完compile之后，之前的文件夹发生了变化，我们发现Hello项目里里多了一个target文件夹。文件夹的内容为：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDkhNrhKaVYA7LiahZPiaP9Vn52Sz8f1xmtibLqAS4k0DLG8phdtcNpQiadQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

发现target里主要存放的就是编译后的字节码文件

**2、运行mvn test-compile** ，target文件夹下面除了classes之外多了test-classes文件夹

**3、运行mvn package** ，target文件夹下面又多了一个`打好的jar包

4、运行mvn clean，发现整个target文件夹都没了。又回到了编译之前我们手动创建的文件夹

#### 五、仓库和坐标

**① pom.xml：** Project Object Model 项目对象模型。它是maven的核心配置文件，所有的构建的配置都在这里设置。

**② 坐标：** 使用下面的三个向量在仓库中唯一的定位一个maven工程

![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPD2CryMU3lzWXmXNhx96VcPUgGFOWaxEs6jIaqQW5ZTv9iaLibxPYQecFA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**③ maven工程的坐标与仓库中路径的关系：**

**![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDZ6BPHwDUNNcMh4fkWMVZkg1U01KVff87CJRu4K0jxC2GAu8RvBbsiaA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)**

maven坐标和仓库对应的映射关系：[groupId][artifactId][version][artifactId]-[version].jar

去本地仓库看一下此目录：org\springframework\spring-core\4.3.4.RELEASE\spring-core-4.3.4.RELEASE.jar

果然是完全对应的

**④ 仓库**

仓库的分类：

**1、本地仓库：** 当前电脑上的仓库，路径上已经说过了哦

**2、远程仓库：**

- 私服：搭建在局域网中，一般公司都会有私服，私服一般使用nexus来搭建。具体搭建过程可以查询其他资料
- 中央仓库：架设在Internet上，像刚才的springframework就是在中央仓库上

#### 六、依赖

**① maven解析依赖信息时会到本地仓库中取查找被依赖的jar包**

- 对于本地仓库中没有的会去中央仓库去查找maven坐标来获取jar包，获取到jar之后会下载到本地仓库
- 对于中央仓库也找不到依赖的jar包的时候，就会编译失败了

**② 如果依赖的是自己或者团队开发的maven工程，需要先使用install命令把被依赖的maven工程的jar包导入到本地仓库中**

举例：现在我再创建第二个maven工程HelloFriend，其中用到了第一个Hello工程里类的sayHello(String name)方法。我们在给HelloFriend项目使用 mvn compile命令进行编译的时候，会提示缺少依赖Hello的jar包。怎么办呢？

到第一个maven工程中执行 mvn install后，你再去看一下本地仓库，你会发现有了Hello项目的jar包。一旦本地仓库有了依赖的maven工程的jar包后，你再到HelloFriend项目中使用 mvn compile命令的时候，可以成功编译

**③ 依赖范围**

**![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDwJ20YzJlFf74L74UnuOpPpt3huQbHVM93ibv9lddlNibLD6XrcT6TCGg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)**

*scope就是依赖的范围*

**1、compile，** 默认值，适用于所有阶段（开发、测试、部署、运行），本jar会一直存在所有阶段。

**2、provided，** 只在开发、测试阶段使用，目的是不让Servlet容器和你本地仓库的jar包冲突 。如servlet.jar。

**3、runtime，** 只在运行时使用，如JDBC驱动，适用运行和测试阶段。

**4、test，** 只在测试时使用，用于编译和运行测试代码。不会随项目发布。

**5、system，** 类似provided，需要显式提供包含依赖的jar，Maven不会在Repository中查找它。

#### 七、生命周期

Maven有三套相互独立的生命周期，请注意这里说的是“三套”，而且“相互独立”，初学者容易将Maven的生命周期看成一个整体，其实不然。这三套生命周期分别是：

**① Clean Lifecycle 在进行真正的构建之前进行一些清理工作。** Clean生命周期一共包含了三个阶段：

- pre-clean 执行一些需要在clean之前完成的工作
- clean 移除所有上一次构建生成的文件
- post-clean 执行一些需要在clean之后立刻完成的工作

**② Default Lifecycle 构建的核心部分，编译，测试，打包，部署等等。**

- validate
- generate-sources
- process-sources
- generate-resources
- process-resources 复制并处理资源文件，至目标目录，准备打包
- compile 编译项目的源代码
- process-classes
- generate-test-sources
- process-test-sources
- generate-test-resources
- process-test-resources 复制并处理资源文件，至目标测试目录
- test-compile 编译测试源代码
- process-test-classes
- test 使用合适的单元测试框架运行测试。这些测试代码不会被打包或部署
- prepare-package
- package 接受编译好的代码，打包成可发布的格式，如 JAR
- pre-integration-test
- integration-test
- post-integration-test
- verify
- install 将包安装至本地仓库，以让其它项目依赖。
- deploy 将最终的包复制到远程的仓库，以让其它开发人员与项目共享

那我们在Hello的项目中执行 mvn install 命令，通过日志看看中间经历了什么？

通过日志我们发现，其实执行mvn install，其中已经执行了compile 和 test 。

**总结：** 不论你要执行生命周期的哪一个阶段，maven都是从这个生命周期的开始执行

**插件：** 每个阶段都有插件（plugin），看上面标红的。插件的职责就是执行它对应的命令。

**③ Site Lifecycle 生成项目报告，站点，发布站点。**

- pre-site 执行一些需要在生成站点文档之前完成的工作
- site 生成项目的站点文档
- post-site 执行一些需要在生成站点文档之后完成的工作，并且为部署做准备
- site-deploy 将生成的站点文档部署到特定的服务器上

#### 九、maven工程的依赖高级特性

**① 依赖的传递性**

**![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDCxDI6EEGBM6OBDQAWtTLOL6XXmpyzyowibDblCDM4yOMw9wp4JrX6MQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)**

WebMavenDemo项目依赖JavaMavenService1 JavaMavenService1项目依赖JavaMavenService2

pom.xml文件配置好依赖关系后，必须首先mvn install后，依赖的jar包才能使用。

- WebMavenDemo的pom.xml文件想能编译通过，JavaMavenService1必须mvn install
- JavaMavenService的pom.xml文件想能编译通过，JavaMavenService2必须mvn install

**传递性：**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDicmxKg1W9Vj5t0VWs6SOo3XHPiceU3QAZZibZTFr4E5oia4lK7NHabqtIA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)**

在Eclipse中，为JavaMavenService2中增加了一个spring-core.jar包后，会惊喜的发现依赖的两个项目都自动的增加了这个jar包，这就是依赖的传递性。

> “
>
> 注意：非compile范围的依赖是不能传递的。

**② 依赖版本的原则：**

**1、路径最短者优先原则**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPD3H8hRZpx1gha2jKuEWNgst5pf3UbuZprmEdtQNCqbqPBtAfkWCgia1Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)**

Service2的log4j的版本是1.2.7版本，Service1排除了此包的依赖，自己加了一个Log4j的1.2.9的版本，那么WebMavenDemo项目遵守路径最短优先原则，Log4j的版本和Sercive1的版本一致。

**2、路径相同先声明优先原则**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDxaYKMUTBFKlntRB3qz7R3Xj0Msh8Fg2AyhGGjGb54bLNhGXq3c97uQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)**

这种场景依赖关系发生了变化，WebMavenDemo项目依赖Sercive1和Service2，它俩是同一个路径，那么谁在WebMavenDemo的pom.xml中先声明的依赖就用谁的版本。

**③ 统一管理依赖的版本：**

**![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfeW8hXribEwVib2PBU2tUsGPDlWdwmDNZu1U0AzPosJkic2e5RgooiaicKUKNaOrjP9Lfa3E8oMhrYic1Fg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)**

为了统一管理版本号，可以使用properties标签，里面可以自定义版本的标签名。在使用的地方使用${自定义标签名}

#### 十、build配置

```xml
<build>
  <!-- 项目的名字 -->
  <finalName>WebMavenDemo</finalName>
  <!-- 描述项目中资源的位置 -->
  <resources>
    <!-- 自定义资源1 -->
    <resource>
      <!-- 资源目录 -->
      <directory>src/main/java</directory>
      <!-- 包括哪些文件参与打包 -->
      <includes>
        <include>**/*.xml</include>
      </includes>
      <!-- 排除哪些文件不参与打包 -->
      <excludes>
        <exclude>**/*.txt</exclude>
          <exclude>**/*.doc</exclude>
      </excludes>
    </resource>
  </resources>
  <!-- 设置构建时候的插件 -->
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>2.1</version>
      <configuration>
        <!-- 源代码编译版本 -->
        <source>1.8</source>
        <!-- 目标平台编译版本 -->
        <target>1.8</target>
      </configuration>
    </plugin>
    <!-- 资源插件（资源的插件） -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-resources-plugin</artifactId>
      <version>2.1</version>
      <executions>
        <execution>
          <phase>compile</phase>
        </execution>
      </executions>
      <configuration>
        <encoding>UTF-8</encoding>
      </configuration>
    </plugin>
    <!-- war插件(将项目打成war包) -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-war-plugin</artifactId>
      <version>2.1</version>
      <configuration>
        <!-- war包名字 -->
        <warName>WebMavenDemo1</warName>
      </configuration>
    </plugin>
  </plugins>
</build>
```

配置好build后，执行mvn package之后，在maven工程指定的target目录里war包和文件都按照配置的生成了好了，maven的所有的内容就整理完了。