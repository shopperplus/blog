---
layout: post
title: "Elasticsearch More Like This 搜索"
author: XguoX
---

**Elasticsearch** 封装好了**More Like This 搜索**功能, 用来基于给定的某个或某组文档, 返回与之相似的文档.
语法如下:

```json

{
    "more_like_this" : {
        "fields" : ["title", "description"],
        "docs" : [
        {
            "_index" : "imdb",
            "_type" : "movies",
            "_id" : "1"
        },
        {
            "_index" : "imdb",
            "_type" : "movies",
            "_id" : "2"
        }],
        "min_term_freq" : 1,
        "max_query_terms" : 12
    }
}
```

结合 **Rails** 例子:

```ruby

@post = Post.last
query = {:query=>{:more_like_this=>{docs:[{_index: 'posts',_type: 'post', _id: @post.id}], :min_term_freq=>1, :min_doc_freq=>1}}}
records = Post.search(query).records
```

![](http://ww2.sinaimg.cn/large/62fdd4d5jw1f2dt5fc3ysj227o09cdik.jpg)

这个查询的参数有三个,分别是 `Document Input Parameters`, `Term Selection Parametersedit`, `Query Formation Parameters`

**Document Input Parameters** 是必须有的查询参数, 可以是 `docs`, `ids` 或者 `like_text`之一.

**Term Selection Parametersedit** 可以是 `max_query_terms`, `min_term_freq`, `min_doc_freq`, `max_doc_freq`, `min_word_length`, `max_word_length`, `stop_words`, `analyzer`

**Query Formation Parameters** 可以是 `minimum_should_match`, `boost_terms`, `include`, `boost`

相关释义[文档](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/query-dsl-mlt-query.html)写的还是挺清楚的.


--------

##### Related:
[Elasticsearch 开箱笔记](http://xguox.me/elasticsearch-ik-mmseg-homebrew-ubuntu.html)

[Elasticsearch on Rails](http://xguox.me/elasticsearch-rails.html)

[Elasticsearch Aggregations 聚合分析](http://xguox.me/elasticsearch-aggregations.html)

[Upgrade Elasticsearch to 2.3](http://xguox.me/upgrade-elasticsearch-2-3.html)

[Elasticsearch Scroll (Ruby)](http://xguox.me/elasticsearch-scroll.html)

