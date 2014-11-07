---
layout: post
title: "You do not know NewRelic"
author: Hooopo
---

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/0443c9ac79a6492aad4810c472308b97.png)


## Enabling garbage collection instrumentation

默认情况，Newrelic的Transactions response time overview 页面是没有统计GC信息的。如果想要开启只需要在`config/initializers/new_relic_unicorn.rb`添加如下代码(如果不是unicorn只需要最后一句)：

```ruby
# -*- encoding : utf-8 -*-
# Ensure the agent is started using Unicorn.
# This is needed when using Unicorn and preload_app is not set to true.
# See https://docs.newrelic.com/docs/ruby/no-data-with-unicorn
if defined? Unicorn
  ::NewRelic::Agent.manual_start()
  ::NewRelic::Agent.after_fork(:force_reconnect => true)
end

GC::Profiler.enable
```

## Recording deployments

New Relic 支持自定义Event，结合 Capistrano 可以把每次发布的时间点划到图表上，题图上到三条粉丝细线就是deployment时刻。
同时，还可以把changelog信息一并记录下来，便于追踪各个版本差异产生的原因，如下图：

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/7ffc389aed2e1443bbcd5ddf2f74d32c.png)


实现这样酷的功能只需要两行代码，在`deploy.rb`里：

```ruby
require 'new_relic/recipes'

after "deploy:updated", "newrelic:notice_deployment"`
```

你不需要发布就可以测试这个功能是否设置正确：

```ruby
cap newrelic:notice_deployment -Snewrelic_desc="Deploying beta Krakatau release"
```

## Availability monitoring

默认情况下，Newrelic会在网站响应速度低到一定阀值邮件通知，也可以手动设置 `Availability monitoring`，
需要提供某个页面到URL，如果这个URL一分钟内ping不通就会收到网站 downtime 邮件。

## Instrumentation Redis

只需Gemfile里添加 `newrelic-redis` 这个gem，前提是你用了 `redis-rb`。

一定有同学会问，Redis都那么快了还需要监控？

原因很简单：

默认情况下Newrelic会把请求时间算在Ruby／View里，这样你发现一个很慢的页面渲染，你无法定位到具体是什么东西那么慢。
加上 `newrelic-redis`之后这部分时间被清晰到记录在Database catalog，并且可以和view其他部分区分开, 细致到每个redis指令执行到时间：

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/9532c2a296f8e30b1107f6d93eb1e2fe.png)

## Instrumentation Rack Middleware

只需要升级 `newrelic_rpm` 到 3.9.0 或以上。

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/22bdbf838583e4d6578ccf01c458aad0.png)

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/f9e38c001dd51e040f68223e41ff399c.png)

## JS Error Notification

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/a171111fbe6cf8d6275c5f2d83ca8db9.png)


就是ExceptionNotify的JS云端版本。当然，这个功能已经有很多人在做了，并不是newrelic首创。但集成到newrelic里却是非常自然，监控异常的同时还可以监控JS错误，何乐而不为呢..

## NewRelic Insights

![](https://ruby-china-files.b0.upaiyun.com/photo/2014/74dab3fcea4ddc6d836c4b04c238ab86.png)

这是一个beta项目，NewRelic把自己收集的数据再开放给你。通过自定义的NRQL，让你可以按照自己关心的方式自定义Dashboard。有了API，你可以再把这些数据倒回自己倒应用，和自己已有的数据做进一步分析。

## Custom Variable

Google Analytics 很早就有了，有了这个东西可玩性就提高了一个层次。能做什么就由开发者自己来想像了。

## NewRelic 和 Twitter的吞吐量相当，数据库使用的是 MySQL（Percona）

Renferer: http://www.slideshare.net/newrelic/how-to-build-a-saas-app-with-twitterlike-throughput-on-just-9-servers

------

相关文档
* https://docs.newrelic.com/docs/ruby/garbage-collection
* https://docs.newrelic.com/docs/ruby/recording-deployments-with-the-ruby-agent
* http://blog.newrelic.com/2014/07/02/ruby-agent-now-automatically-instruments-rack-middlewares