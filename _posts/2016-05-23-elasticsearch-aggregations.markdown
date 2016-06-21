---
layout: post
title: "Elasticsearch Aggregations 聚合分析"
author: XguoX
---

**Elasticsearch** 除了全文搜索以外还有一个主要功能, 就是数据的聚合分析, **Aggregations**. 有点类似于 SQL 中的 `GROUP BY`.

Elasticsearch 的 [Aggregations API](https://www.elastic.co/guide/en/elasticsearch/reference/1.7/search-aggregations.html) 给出了一大堆的用法.

1.X 主要分两类: **Bucket Aggregations** 和 **Metrics Aggregations**, 2.X 多了一类 **Pipeline aggregations**. Pipeline aggregations 比较新, 官方的说法是在未来的改动会较大, 甚至会移除, 所以, 暂时不讨论先.

**Metrics Aggregations** 顾名思义, 主要是用于计算特定的度量字段, 其实也不一定是文档的某个特定字段值, 可以是文档通过 script 生成的值. Metrics Aggregations 是不能有子聚合的(sub-aggregations)
**Bucket Aggregations** 英语的 Bucket 有'桶'的意思, 按照官方的说法, Bucket Aggregations 定义了一些特定的条件, 比如 'country' 字段值为 'Canada', 或者 'gender' 字段值为 'Male', 'comments_count' 在区间 `[{ to: 50 },{ from: 50, to: 150 }, { from: 150, to: 500 }]` 之中等等, 只要文档满足这些条件就丢进这个桶(Bucket)里面.   Bucketing aggregations 可以有子聚合(sub-aggregations), sub-aggregations 可以是 Bucketing aggregations 也可以是 Metrics Aggregations. 子聚合是在父聚合"桶里面"的文档集合上继续做聚合分析的.
大部分情况下, 配合这两种聚合类型一起用才能发挥更多功效.

##### Aggregations 的语法结构:

```javascript

"aggs" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]
        [,"aggs" : { [<sub_aggregation>]+ } ]
    }
    [,"<aggregation_name_2>" : { ... } ]
}
```

如果喜欢敲多几个字母的话, `aggs` 可以用 `aggregations`来代替.

举个栗子:

```javascript

➜ curl -XPOST "http://localhost:9200/plus-customers/plus-customer/_search?pretty" -d '
{
    "size": 0,
    "query": {
        "filtered": {
            "query": {
                "match_all": {}
            },
            "filter": {
                "bool": {
                    "must": [{
                        "term": {
                            "gender": 1
                        }
                    }]
                }
            }
        }
    },
    "aggs": {
        "group_by": {
            "terms": {
                "field": "country"
            },
            "aggs": {
                "avg_metric": {
                    "avg": {
                        "field": "orders_count"
                    }
                }
            }
        }
    }
}
'
```

返回结果:

```javascript

{
  "took" : 183,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "hits" : {
    "total" : 9930000,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "group_by" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [ {
        "key" : "Canada",
        "doc_count" : 1299880,
        "avg_metric" : {
          "value" : 0.628643724696356
        }
      }, {
        "key" : "United States",
        "doc_count" : 8630120,
        "avg_metric" : {
          "value" : 3.4285714285714286
        }
      } ]
    }
  }
}
```

`size: 0` 是指只返回聚合结果, 不加这个条件的话, ["hits"]["hits"] 对应的值会是一个包含所有匹配文档的数组.


##### 当使用 Filter Aggregation 时候

```javascript

➜ curl -XPOST "http://localhost:9200/plus-customers/plus-customer/_search?pretty" -d '
  {
    "size": 0,
    "aggs": {
      "agg1": {
          "filter": {
              "term": {
                  "country": "Canada",
                  "state": "Quebec"
              }
          },
          "aggs": {
              "agg2": {
                  "terms": {
                      "field": "gender"
                  }
              }
          }
      }
    }
  }
'
// Elasticsearch 1.X 这样还能正常运行, 到了 2.X 就会报错了
// [term] query does not support different field names, use [bool] query instead
// 要用下面这样来替代了

➜ curl -XPOST "http://localhost:9200/plus-customers/plus-customer/_search?pretty" -d '
  {
    "size": 0,
    "aggs": {
      "agg1": {
          "filter": {
            "bool": {
              "must": [
                {
                  "match": {
                    "country": "Canada"
                  }
                },
                {
                  "match": {
                    "state": "Quebec"
                  }
                }
              ]
            }
          },
          "aggs": {
              "agg2": {
                  "terms": {
                      "field": "gender"
                  }
              }
          }
      }
    }
  }
'
```

##### Elasticsearch Aggregation 不支持分页

默认结果是只返回 `doc_count` 前十的的 keys, 如果要使结果返回所有的 keys 的话, 需要加上 `"size": 0`, 比如:

```javascript

➜  crm git:(master)  curl -XPOST "http://localhost:9200/plus-customers/plus-customer/_search?pretty" -d '{
  "size": 0,
  "aggs": {
      "agg1": {
          "terms": {
            "field": "state",
            "size": 0
          }
      }
  }
}'
```

如果不加 `"size": 0`, 返回是这样子的:

```javascript

{
  "took" : 35,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "hits" : {
    "total" : 448140,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "agg1" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 297971,
      "buckets" : [ {
        "key" : "",
        "doc_count" : 33440
      }, {
        "key" : "Delaware",
        "doc_count" : 27357
      }, {
        "key" : "New jersey",
        "doc_count" : 23021
      }, {
        "key" : "South carolina",
        "doc_count" : 14211
      }, {
        "key" : "Utah",
        "doc_count" : 10169
      }, {
        "key" : "Virginia",
        "doc_count" : 9920
      }, {
        "key" : "Pennsylvania",
        "doc_count" : 9879
      }, {
        "key" : "Connecticut",
        "doc_count" : 8978
      }, {
        "key" : "Arizona",
        "doc_count" : 7231
      }, {
        "key" : "California",
        "doc_count" : 5963
      } ]
    }
  }
}
```

一直以来都有很多人抱怨为嘛不给 Aggregation 加上分页, 这里有讨论[https://github.com/elastic/elasticsearch/issues/4915](https://github.com/elastic/elasticsearch/issues/4915)
目前数据量不算太巨大的时候, 还是可以搞假分页的(仅仅视觉上), 不过每次还是把所有的结果都取出来了.

##### Related:
[Elasticsearch 开箱笔记](http://xguox.me/elasticsearch-ik-mmseg-homebrew-ubuntu.html)

[Elasticsearch on Rails](http://xguox.me/elasticsearch-rails.html)

[Elasticsearch More Like This 搜索](http://xguox.me/elasticsearch-more-like-this.html)

[Upgrade Elasticsearch to 2.3](http://xguox.me/upgrade-elasticsearch-2-3.html)

[Elasticsearch Scroll (Ruby)](http://xguox.me/elasticsearch-scroll.html)

