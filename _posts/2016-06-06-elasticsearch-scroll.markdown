---
layout: post
title: "Elasticsearch Scroll (Ruby)"
author: XguoX
---

**Elasticsearch** 通过 `from` 和 `size` 参数来实现分页. `size` 表示返回的结果数量, 默认为 10, `from` 则表示从起始结果算起要跳过的结果数量, 默认为 0. 所以, 默认情况下如果返回结果数是 10 个以上, 我们得到的只有前十个结果.

1.X 的时候不过请求的页数多深, 结果数多大都会給你返回结果, 只是越到后面的页数越大或者结果数越多, 执行的效率会逐渐变慢而已.

**页数越大之所以返回效率越差**, 是因为 Elasticsearch 分页的工作方式是从所有主分片的索引中返回排在最前面的 `size` 数量的结果, 然后再把各个分片结果合并以后再排序, 然后再返回前 `size` 个结果.

假设还是默认值, 如果想返回第 1000 页的话, 也就是排在 10001 - 10010 的结果. 而并不是直接一拿就拿到排在第 10001 之后的结果集, 而是返回 10010个结果然而排序, 然后丢弃掉前面的 10000 个结果.

这是只有一个主分片的情况是这么工作的, 而如果假设是 5 个主分片的话. 还是请求第 1000页, 那么 Elasticsearch 要在这 5 个主分片下搜索结果, 然后每个分片得到 10010 个结果, 合并以后排序这 50050 个结果, 最后丢弃前面的 50040 个.

所以, 不管是 Google 还是百度, 都会限定返回的结果页数. 用 Google 搜索 `Elasticsearch` 返回了  466,000 条结果, 然而只给我们前 15 页的.

然后, **升到 2.X 以后**, 如果请求产生的结果数超过一万个的话, 会抛出这样的错误信息

```javascript

{:error=>
 {:root_cause=>
   [{:type=>"query_phase_execution_exception",
     :reason=>
      "Result window is too large, from + size must be less than or equal to: [10000] but was [12000]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level parameter."}],
  :type=>"search_phase_execution_exception",
  :reason=>"all shards failed",
  :phase=>"query_fetch",
  :grouped=>true,
  :failed_shards=>
   [{:shard=>0,
     :index=>"customers",
     :node=>"RZ4Rj4QZQ7G7EIv5Y-CnEw",
     :reason=>
      {:type=>"query_phase_execution_exception",
       :reason=>
        "Result window is too large, from + size must be less than or equal to: [10000] but was [12000]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level parameter."}}]},
:status=>500}
```

Elasticsearch 也很 nice 的建议我们使用 Scroll API.

```ruby

client = Elasticsearch::Model.client
client.indices.delete index: 'test'
1_000.times do |i|
  client.index index: 'test',
               type: 'test',
               id: i+1,
               body: {title: "Test #{i}"}
end
client.indices.refresh index: 'test'

result = client.search index: 'test',
                       scroll: '5m',
                       body: { query: { match: { title: 'test' } }, sort: '_id' }
```

```javascript

//result

{"_scroll_id"=>
  "cXVlcnlUaGVuRmV0Y2g7NTs3OlJaNFJqNFFaUTdHN0VJdjVZLUNuRXc7OTpSWjRSajRRWlE3RzdFSXY1WS1DbkV3Ozg6Ulo0Umo0UVpRN0c3RUl2NVktQ25FdzsxMDpSWjRSajRRWlE3RzdFSXY1WS1DbkV3OzExOlJaNFJqNFFaUTdHN0VJdjVZLUNuRXc7MDs=",
 "took"=>18,
 "timed_out"=>false,
 "_shards"=>{"total"=>5, "successful"=>5, "failed"=>0},
 "hits"=>
  {"total"=>1000,
   "max_score"=>nil,
   "hits"=>
    [{"_index"=>"test", "_type"=>"test", "_id"=>"14", "_score"=>nil, "_source"=>{"title"=>"Test 13"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"19", "_score"=>nil, "_source"=>{"title"=>"Test 18"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"22", "_score"=>nil, "_source"=>{"title"=>"Test 21"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"24", "_score"=>nil, "_source"=>{"title"=>"Test 23"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"25", "_score"=>nil, "_source"=>{"title"=>"Test 24"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"26", "_score"=>nil, "_source"=>{"title"=>"Test 25"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"29", "_score"=>nil, "_source"=>{"title"=>"Test 28"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"40", "_score"=>nil, "_source"=>{"title"=>"Test 39"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"41", "_score"=>nil, "_source"=>{"title"=>"Test 40"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"44", "_score"=>nil, "_source"=>{"title"=>"Test 43"}, "sort"=>[nil]}]}}
```

可以看到每一次 scroll 返回的结果都带有 `_scroll_id`, 然后后续接着利用这个 `_scroll_id` 来滚动搜索.

```ruby

result = client.scroll scroll: '5m', scroll_id: result['_scroll_id']
```

```javascript

=> {"_scroll_id"=>
  "cXVlcnlUaGVuRmV0Y2g7NTs3OlJaNFJqNFFaUTdHN0VJdjVZLUNuRXc7OTpSWjRSajRRWlE3RzdFSXY1WS1DbkV3Ozg6Ulo0Umo0UVpRN0c3RUl2NVktQ25FdzsxMDpSWjRSajRRWlE3RzdFSXY1WS1DbkV3OzExOlJaNFJqNFFaUTdHN0VJdjVZLUNuRXc7MDs=",
 "took"=>5,
 "timed_out"=>false,
 "_shards"=>{"total"=>5, "successful"=>5, "failed"=>0},
 "hits"=>
  {"total"=>1000,
   "max_score"=>nil,
   "hits"=>
    [{"_index"=>"test", "_type"=>"test", "_id"=>"48", "_score"=>nil, "_source"=>{"title"=>"Test 47"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"52", "_score"=>nil, "_source"=>{"title"=>"Test 51"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"60", "_score"=>nil, "_source"=>{"title"=>"Test 59"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"73", "_score"=>nil, "_source"=>{"title"=>"Test 72"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"79", "_score"=>nil, "_source"=>{"title"=>"Test 78"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"84", "_score"=>nil, "_source"=>{"title"=>"Test 83"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"89", "_score"=>nil, "_source"=>{"title"=>"Test 88"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"92", "_score"=>nil, "_source"=>{"title"=>"Test 91"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"98", "_score"=>nil, "_source"=>{"title"=>"Test 97"}, "sort"=>[nil]},
     {"_index"=>"test", "_type"=>"test", "_id"=>"99", "_score"=>nil, "_source"=>{"title"=>"Test 98"}, "sort"=>[nil]}]}}
```
`scroll: '5m'` 是指这个 _scroll_id 的有效时长为 5分钟, 除了分钟以外, 年月日时分秒都可以.
btw, 官方说每次 scroll 以后产生新的 _scroll_id, 不知道是不是理解错误, 除非开启一个新的 scroll 会生成新的 _scroll_id, 不然, 如果在有效时间内继续滚动的话返回的 _scroll_id 是一样的.

如果只对查询的结果总体感兴趣而不需要对总体排序的话, 可以使用更为高效的 `scan` 模式,

```ruby

r = client.search index: 'test', search_type: 'scan', scroll: '5m', size: 10

# Call the `scroll` API until empty results are returned
while r = client.scroll(scroll_id: r['_scroll_id'], scroll: '5m') and not r['hits']['hits'].empty? do
  puts "--- BATCH #{defined?($i) ? $i += 1 : $i = 1} -------------------------------------------------"
  puts r['hits']['hits'].map { |d| d['_source']['title'] }.inspect
  puts
end
```
--------

##### Related:
[Elasticsearch 开箱笔记](http://xguox.me/elasticsearch-ik-mmseg-homebrew-ubuntu.html)

[Elasticsearch on Rails](http://xguox.me/elasticsearch-rails.html)

[Elasticsearch More Like This 搜索](http://xguox.me/elasticsearch-more-like-this.html)

[Elasticsearch Aggregations 聚合分析](http://xguox.me/elasticsearch-aggregations.html)

[Upgrade Elasticsearch to 2.3](http://xguox.me/upgrade-elasticsearch-2-3.html)

[https://github.com/elastic/elasticsearch-ruby/blob/master/elasticsearch-api/lib/elasticsearch/api/actions/scroll.rb](https://github.com/elastic/elasticsearch-ruby/blob/master/elasticsearch-api/lib/elasticsearch/api/actions/scroll.rb)