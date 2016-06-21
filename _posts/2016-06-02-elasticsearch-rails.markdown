---
layout: post
title: "Elasticsearch on Rails"
author: XguoX
---

[上一篇](http://xguox.me/elasticsearch-ik-mmseg-homebrew-ubuntu.html)扯了一大通都只是 **Elasticsearch** 的安装配置, 现在扯点集成到 **Rails** 上的东西.

`elasticsearch-rails` 这个 Repository 是由三个 Gem 组成,

```ruby

gem 'elasticsearch-model'
gem 'elasticsearch-rails'
gem 'elasticsearch-persistence'
```

总觉得这里起名略蛋疼 = . = 一般用到前两个较多.

```ruby

class Post < ActiveRecord::Base
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks

  # settings do
  #   mappings dynamic: 'false' do
  #     indexes :title, type: 'string', analyzer: 'ik_smart'
  #     indexes :keywords, type: 'string', analyzer: 'ik_smart'
  #     indexes :body, type: 'string', analyzer: 'ik_smart'
  #     indexes :user_name, type: 'string', analyzer: 'ik_smart'
  #   end
  # end

  # def as_indexed_json(options={})
  #   as_json(
  #     only: ['title', 'body', 'keywords'],
  #     methods: [:user_name]
  #   )
  # end
end
```

`Elasticsearch::Model::Callbacks` 这个模块主要是当 model 更新以后回调更新索引. 对于厂里的 CRM 来说, 因为是数据仓库, 更多是做一些查询, 分析之类的应用, 而不是主要应用于增删改查, 所以, 其实我们没有 include 这个模块.
手工执行一下 `Post.import` 基本工作就完成了.

也可以只 [import](https://github.com/elastic/elasticsearch-rails/blob/master/elasticsearch-model/lib/elasticsearch/model/importing.rb) 特定 scope 或者查询下面的记录.

```ruby

Post.import scope: 'published'
#
Post.import query: -> { where(user_id: user_id) }
```

title, keywords, body 是 posts 表的字段, user_name 是 Post 类的一个方法.

上面用了 `settings` 设置映射以后还要定义 `as_indexed_json`, 否则该方法来自 **`Elasticsearch::Model::Serializing`** 默认实现(只)会序列化所有原有字段(比如包括 created_at, updated_at 等). `settings` 设置也不会起作用.

`mappings` 方法的参数 `dynamic: 'false'`
是对文档新增 field 的处理, 默认为 `true`, 也就是会动态加入这个 field, false 的话则无视之.
设置为 false 并不会改变文档的 `_source`. `_source` 仍然是只包含已经索引的整个 JSON 文档, 任何新增 field 都不会被添加到映射, 也不会被搜索到.

```ruby

[5] pry(main)> Post.search('blahblahblah').records.all
  Post Search (11.9ms) {index: "posts", type: "post", q: "blahblahblah"}
  Post Load (0.2ms)  SELECT `posts`.* FROM `posts` WHERE 1=0
=> []
[6] pry(main)> Post.search('cool').records.all
  Post Search (12.8ms) {index: "posts", type: "post", q: "cool"}
  Post Load (0.4ms)  SELECT `posts`.* FROM `posts` WHERE `posts`.`id` IN (13, 108)
=> [#<Post:0x007f8819c2d690
  id: 13,
  title: "Peanut and Peach nut",
  body:
```

```ruby

[10] pry(main)> resp = Post.search('一张照片')
=> #<Elasticsearch::Model::Response::Response:0x007fcc13200008
@klass=[PROXY] Post (call 'Post.connection' to establish a connection),
@search=
  #<Elasticsearch::Model::Searching::SearchRequest:0x007fcc13200238
   @definition={:index=>"posts", :type=>"post", :q=>"一张照片"},
   @klass=[PROXY] Post (call 'Post.connection' to establish a connection),
   @options={}>>

[11] pry(main)> resp.records
=> #<Elasticsearch::Model::Response::Records:0x007fcc11719ca8
@klass=[PROXY] Post (call 'Post.connection' to establish a connection),
@response=
 #<Elasticsearch::Model::Response::Response:0x007fcc117c93b0
  @klass=[PROXY] Post (call 'Post.connection' to establish a connection),
  @records=#<Elasticsearch::Model::Response::Records:0x007fcc11719ca8 ...>,
  @search=
   #<Elasticsearch::Model::Searching::SearchRequest:0x007fcc117c8578
    @definition={:index=>"posts", :type=>"post", :q=>"一张照片"},
    @klass=[PROXY] Post (call 'Post.connection' to establish a connection),
    @options={}>>>
```

```ruby

[9] pry(main)> resp.records.first
  Post Load (0.8ms)  SELECT `posts`.* FROM `posts` WHERE `posts`.`id` IN (61, 4, 82, 15, 90, 72, 114, 112, 28, 17)
=> #<Post:0x007fcc18c23238
 id: 61,
 title: "让布列松看不顺眼的摄影师",
 body: "BlahBlahBlahBlah...........",
 user_id: 1,
 slug: "bdgh2Lyf4z3jF",
 created_at: Mon, 10 Feb 2015 12:51:04 CST +08:00,
 updated_at: Tue, 09 Aug 2015 08:00:07 CST +08:00,
 comments_count: 0,
 keywords: "Photo 摄影 布列松",
 popular: false>
```

```ruby

[12] pry(main)> resp.results
=> #<Elasticsearch::Model::Response::Results:0x007fcc11693568
@klass=[PROXY] Post (call 'Post.connection' to establish a connection),
@response=
  #<Elasticsearch::Model::Response::Response:0x007fcc117c93b0
   @klass=[PROXY] Post (call 'Post.connection' to establish a connection),
   @records=
    #<Elasticsearch::Model::Response::Records:0x007fcc11719ca8
     @klass=[PROXY] Post (call 'Post.connection' to establish a connection),
     @response=#<Elasticsearch::Model::Response::Response:0x007fcc117c93b0 ...>>,
   @results=#<Elasticsearch::Model::Response::Results:0x007fcc11693568 ...>,
   @search=
    #<Elasticsearch::Model::Searching::SearchRequest:0x007fcc117c8578
     @definition={:index=>"posts", :type=>"post", :q=>"一张照片"},
     @klass=[PROXY] Post (call 'Post.connection' to establish a connection),
     @options={}>>>

[8] pry(main)> resp.results.first
 Post Search (39.4ms) {index: "posts", type: "post", q: "一张照片"}
=> #<Elasticsearch::Model::Response::Result:0x007fcc18d44270
@result=
 {"_index"=>"posts",
  "_type"=>"post",
  "_id"=>"61",
  "_score"=>0.21295425,
  "_source"=>
   {"title"=>"让布列松看不顺眼的摄影师",
    "body"=> "BlahBlahBlahBlah...........",
    "keywords"=>"Photo 摄影 布列松",
    "user_name"=>"XguoX"}}>
```
用 `results` 返回的是对应的 JSON, 而用 `records` 返回的则是经过数据库查询的记录. 毫无疑问, 使用 results 的话性能会好一些.

**查看某个 model, 比如 Post 的映射配置**
`curl  'http://localhost:9200/posts?pretty'`

返回结果
![](http://ww3.sinaimg.cn/large/62fdd4d5jw1f2ewz27e0jj20tc1aetdv.jpg)

对比一下, 同样的关键词使用 ik, ik_smart 以及使用标准 anlayser 之间分词区别.
![](http://ww3.sinaimg.cn/large/62fdd4d5jw1f2ewz19m8xj227s0pk7ep.jpg)


索引文档之间的关联关系, 假设 Post `has_many` Comments. 可以像下面这样进行序列化:

```ruby

def as_indexed_json(options={})
  as_json(
    only: ['title', 'body', 'keywords'],
    methods: [:user_name],
    include: {comments: {only: :content}}
  )
end
```

```ruby

[19] pry(main)> Post.last.__elasticsearch__.as_indexed_json
  Post Load (0.3ms)  SELECT  `posts`.* FROM `posts`  ORDER BY `posts`.`id` DESC LIMIT 1
  User Load (0.2ms)  SELECT  `users`.* FROM `users` WHERE `users`.`id` = 1 LIMIT 1
  Comment Load (0.2ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`post_id` = 61
=> {"title"=>"让布列松看不顺眼的摄影师",
 "body"=>"blahblahblah.......",
 "keywords"=>"Photo 摄影 布列松",
 "user_name"=>"XguoX",
 "comments"=>[{"content"=>"Here is 评论..."}]}
```

**更改了映射设置以后要更新索引**,

```ruby

Post.__elasticsearch__.create_index! force: true
Post.__elasticsearch__.refresh_index!
```

**手动更新单个文档**

```ruby

[8] pry(main)> Post.first.__elasticsearch__.index_document
  Post Load (0.4ms)  SELECT  `posts`.* FROM `posts`  ORDER BY `posts`.`id` ASC LIMIT 1
  User Load (0.4ms)  SELECT  `users`.* FROM `users` WHERE `users`.`id` = 2 LIMIT 1
=> {"_index"=>"posts", "_type"=>"post", "_id"=>"1", "_version"=>2, "created"=>false}
```

--------

##### Related:
[Elasticsearch 开箱笔记](http://xguox.me/elasticsearch-ik-mmseg-homebrew-ubuntu.html)

[Elasticsearch More Like This 搜索](http://xguox.me/elasticsearch-more-like-this.html)

[Elasticsearch Aggregations 聚合分析](http://xguox.me/elasticsearch-aggregations.html)

[Upgrade Elasticsearch to 2.3](http://xguox.me/upgrade-elasticsearch-2-3.html)

[Elasticsearch Scroll (Ruby)](http://xguox.me/elasticsearch-scroll.html)

