---
layout: post
title: "Rails.cache.fetch Relation 的'坑'"
author: XguoX
---

除了页面(Page), 动作(Action), 片段(Fragment)等这三种 Rails 所支持的缓存以外, Rails 还提供了更底层的, 对于特定值或者查询**结果**缓存的支持.

```ruby

[3] pry(main)> Rails.cache.read('first_post_cache')
=> nil

[5] pry(main)> Rails.cache.write('first_post_cache', Post.first, expires_in: 2.minute)
  Post Load (0.3ms)  SELECT  `posts`.* FROM `posts`  ORDER BY `posts`.`id` ASC LIMIT 1
=> true

[6] pry(main)> Rails.cache.read('first_post_cache')
=> #<Post:0x007fcde2587d40
 id: 1,
 title: "Rails.cache.fetch scope 的'坑'",
 body: '',
 created_at: Fri, 03 Jan 2015 23:55:44 CST +08:00,
 updated_at: Thu, 21 Jul 2015 08:00:20 CST +08:00>
```

每次都要先 read 再 write 比较繁琐, 所以, 就有了一个方便些的方法 **`Rails.cache.fetch`**

```ruby

[7] pry(main)> Rails.cache.fetch('cache_first_post', expires_in: 2.minute) { Post.first }
  Post Load (0.4ms)  SELECT  `posts`.* FROM `posts`  ORDER BY `posts`.`id` ASC LIMIT 1
=> #<Post:0x007fcde2474fe8
 id: 1,
 title: "Rails.cache.fetch scope 的'坑'",
 body: ''>

 [9] pry(main)> Rails.cache.fetch('cache_first_post', expires_in: 2.minute) { Post.first }
 => #<Post:0x007fcde228afc0
  id: 1,
  title: "Rails.cache.fetch scope 的'坑'",
  body: '',
  created_at: Fri, 03 Jan 2015 23:55:44 CST +08:00,
  updated_at: Thu, 21 Jul 2015 08:00:20 CST +08:00>

```

咋看上去都还正常运作的, `Rails.cache.fetch` 在对应的 key 如果能找到值的时候就不会再去查数据库了.

再来看下面,

```ruby

[10] pry(main)> Rails.cache.fetch('cache_all_posts', expires_in: 2.minute) { Post.all }
  Post Load (4.8ms)  SELECT `posts`.* FROM `posts`
=> [#<Post:0x007fcddc2dfc38
  id: 1,
  title: "Rails.cache.fetch scope 的'坑'",
  body: '',
  created_at: Fri, 03 Jan 2015 23:55:44 CST +08:00,
  updated_at: Thu, 21 Jul 2015 08:00:20 CST +08:00>,
  #<Post:0x007fcddfab0e28
  id: 2...]>

[11] pry(main)> Rails.cache.fetch('cache_all_posts', expires_in: 2.minute) { Post.all }
  Post Load (4.1ms)  SELECT `posts`.* FROM `posts`
=> [#<Post:0x007fb96b6a3c50
  id: 1,
  title: "Rails.cache.fetch scope 的'坑'",
  body: '',
  created_at: Fri, 03 Jan 2015 23:55:44 CST +08:00,
  updated_at: Thu, 21 Jul 2015 08:00:20 CST +08:00>,
  #<Post:0x007f9a7fd787e8
  id: 2,
```

即便第一次看起来做了缓存(其实也的确有缓存文件在 `tmp/cache/` 下面), 但是, 当再次执行 `Rails.cache.fetch` 的时候, 还是去数据库查了. 实际使用中可能不会这么干把所有 posts 都缓存(Redis/Memcached)到内存里面, 那样的话供给缓存用的内存估计瞬间就被榨干了. 但是, 这里数据库又再去查了一次感觉不科学啊.

搜了下 [Stack Overflow](http://stackoverflow.com/questions/11218917/confusion-caching-active-record-queries-with-rails-cache-fetch), 说是, `User.where('status = 1').limit(1000)` `Post.all` 这些, 返回的实际上是 **scope**, 并非真正的查询. cache 的是 `scope` 不是查询结果.

换成
`Rails.cache.fetch('cache_all_posts', expires_in: 2.minute) { Post.all.load }`
以后, 就跟预想的一样工作了. 但是, 查看(`ActiveSupport::Cache::FileStore`)了一下缓存文件, 当缓存 scope 时候, 文件大小为 **680 bytes**, 而当缓存查询结果的时候, 文件的大小是 MB 级别的.

##### ActiveRecord::Relation 究竟是什么鬼?

```ruby

[7] pry(main)> User.where(id: 1)
  User Load (6.9ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1
=> [<User:0x007fb96ee98b88 id: 1, email: "test@test.com">]
```

看上去返回的结果很像是数组, 实际上, 却不是数组.

```ruby

[8] pry(main)> _.class
=> User::ActiveRecord_Relation(Rails 4)
=> ActiveRecord::Relation(Rails 3)
```

**btw, Rails 3 中执行 all 是返回 Array的, 而 Rails 4 则是 Relation**

**ActiveRecord::Relation** 只有当真正需要知道并使用到里面所包含的对象时候才会被执行. 比如在 controller 中:

```ruby

class PostsController < ApplicationController
  def index
    @posts    = Post.all
    @channels = Channel.all
    @comments = Comment.all
  end
end
```

假设在 `index.html.erb` 中, 没有任何地方用到 `@comments` 的话, 其实是不会执行数据库查询去把所有 comments 抓出来的. 而如果先使用到了 `@channels`, 比如 `@channels.first.name` 的话, 也是一样, 先执行

```sql

  Channel Load (27.8ms)  SELECT `channels`.* FROM `channels`
```

再去执行(假设 view 里面有使用到 `@posts`)

```sql

  Post Load (47.2ms) SELECT `posts`.* FROM `posts`
```

所以, `Rails.cache.fetch('cache_all_posts', expires_in: 2.minute) { Post.all }` 缓存的只是 Relation 对象而不是查询结果. 其实分开来, 使用 `Rails.cache.read` 和 `Rails.cache.write` 也是这样的.  ╮(╯_╰)╭ 说是坑, 好吧, 其实还是没透彻地弄清楚 Rails
