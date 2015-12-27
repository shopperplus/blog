---
layout: post
title: "Breaking Up a Monolithic Rails App Without MicroService"
author: Hooopo
---

一个业务系统随着不断的迭代，功能越来越多，代码量也随之越来越大，成为Monolithic系统。主流的拆分方案是MicroService，当然，也有Cookpad这种继续Monolithic的任性做法。那么，除了Monolithic和MicroService之外，还有没有其他的拆分方案呢？在比较各种方案之前，我们先来了解一下以下3个指标：

1. **业务逻辑共享**：不需要为相同的功能写两套甚至更多的重复代码。
2. **可以独立部署**：每个小应用在访问量和应用类型是有很大差别的，比如管理后台可能只有自己人在用，访问量小，但多耗时的请求。前台网站和API可能并发访问量高，需要部署更多的进程。独立部署的好处是可以节省内存，针对各种的访问量调整进程数量。
3. **没有远程调用**：除了需要给远程调用编写额外的API接口之外，远程调用带来的额外性能开销是巨大的，无论是服务端还是客户端。同时，远程调用给编码带来的复杂性也随之增加，本来一个`update_attributes`方法轻易能做到的功能，再引入远程调用之后，需要在代码里去处理超时、处理异常、重试，也就是把一个单机问题变成了分布式系统问题。另外一个难解的问题就是数据库事务，带有写入操作的远程调用处理事务更加困难。

上面提到的3点，Monolithic满足了1和3，而MicroService满足了1和2。那么有没有3点都满足的呢？答案是Engine！

## Core Engine 和  Feature Engine

首先，Rails Engine是什么和Rails Engine怎么用就不在本文介绍了，可以通过下面两个链接来了解：

* http://guides.rubyonrails.org/engines.html
* https://github.com/JuanitoFatas/Guides/blob/master/guides/edge-translation/engines-zh_CN.md

好吧，Core Engine 和  Feature Engine这两个概念是不存在的，为了区分我自己启的。一提到Engine，我们首先想到的是Devise和Forem这种有完整MVC结构，mount到宿主应用上，提供扩展功能的 Feature Engine。然而，另外一种用于业务逻辑共享的 Engine，暂且称为 Core Engine。

Core Engine 里主要是Model，但也不仅仅是Model，还包括Migration、Model依赖的Gem、全局配置和常量等。

## Engine和普通Gem的区别

Gem主要用来共享通用逻辑，而Engine可以用来共享业务逻辑。并且，Engine可以 hook到 Rails 的生命周期里，比如设置auto load 路径，eager load 路径、migration 文件路径等。
Rails App 使用的是 Gemfile，而Engine使用的是gemspec。两者虽然相似，但有细微的区别，在使用的时候需要注意。比如，gemspec里是不能引用 git 上的资源，只能引用rubygems或本地的gem。另外，gemspec只负责安装依赖，但不管加载，所以除了像Gemfile一样指定了gem，还要手动require 所引入的gem。

## Demo

首先介绍一下我们网站的一些背景：

* 网站前台（Frontend）：主要是注册、登录、shopping cart、checkout、payment、product and catalog 显示、搜索、推荐等功能。
* 网站管理后台（Backend）：主要是财务、订单处理、包裹处理、产品和类目管理、促销管理、采购、库存管理、客服系统、第三方平台订单同步等。
* API：提供给Android和iOS应用

我们的目标是把网站拆分成网站前台、网站管理后台、API，并满足上面提到的三点：业务逻辑共享、可以独立部署、无远程调用。

![Shared Core Engine](https://camo.githubusercontent.com/0f2463acb9375394c91a8a387e34a07337f6840e/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f3530352d373932646230313039636431623032332e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

下面开始演示抽出Core Engine涉及到的一些细节问题：

lib/core/engine.rb

```ruby
require 'rails/all'
require 'mysql2'
require 'active_merchant'
require 'kaminari'
require 'cancan'
require 'ancestry'
require 'state_machine'
# other require

module Core
  class Engine < ::Rails::Engine
  end
end
```
宿主App Gemfile,以backend为例：

```ruby
source 'https://rubygems.org/'
gem 'rails'
gem 'core', :path => '../core'
```

目录结构：

```
├── core
├── backend
├── api
└── frontend
```

 让宿主项目可以使用Core Engine里的Migration文件：

```ruby
module Core
  class Engine < ::Rails::Engine

    # run bundle exec rake db:migrate 可以migrate core 内部的 migrations
    initializer :append_migrations do |app|
      unless app.root.to_s == root.to_s
        app.config.paths["db/migrate"] += config.paths["db/migrate"].expanded
      end
    end
  end
end
```

Auto Load Path等配置：

```ruby
module Core
  class Engine < ::Rails::Engine
    config.autoload_paths += %W(
      #{config.root}/lib
      #{config.root}/app/models/concerns
      #{config.root}/app/models/rpush
      #{config.root}/app/service_objects
    )

    config.eager_load_paths += %W(
      #{config.root}/app/mailers
      #{config.root}/lib
      #{config.root}/app/models/concerns
      #{config.root}/app/models/rpush
      #{config.root}/app/uploaders
    )
  end
end
```

cap deploy之前自动同步最新的代码：

```ruby
set :core_path, '/var/www/core'

task :sync_core, :roles => :web do
  b = ENV['CORE_BRANCH'] || branch
  bash = StringIO.new(<<-BASH)
    if [ -d #{core_path} ]; then
      cd #{core_path} &&  git remote update && git checkout origin/#{b}
    else
      git clone -b #{b} git@xxxx/core.git #{core_path}
    fi
  BASH
  upload(bash, "/tmp/fetch_core.sh")
  run "/bin/bash /tmp/fetch_core.sh"
  run "ln -sf #{core_path} #{deploy_to}/releases/"
  run "ln -sf #{core_path} #{deploy_to}/"
end

before 'bundle:install', 'sync_core'
```

除了Model可以共享之外，路由其实也可以通过Core Engine的方式共享。一些情况，在管理后台发邮件和rake task也需要调用前台App的路由结构，但由于网站管理后台和网站前台分离，只能通过硬编码来拼路由或是引入MicroService，而Core Engine的方式可以共享Rails App的任意部分，抽出Route Engine 之后，只需要这样调用：

```ruby
MyApp::Engine.routes.url_helpers.new_post_path
```

## Thin Model Fat Service Object

理论上，Core Engine里的Model应该是网站前台和管理后台共用那部分，而只有前台使用的Model单独留在前台，只有后台使用的Model单独留在后台。但由于Rails的关联机制等因素，这一点很难做到。退而求其次，我们可以优化到：Core Engine包含所有Model，但网站前台业务逻辑代码只放在网站前台，网站管理后台的业务逻辑代码只放在管理后台。要做到这一点就必须改变之前Fat Model的代码组织方式：

```ruby
class Product < AR
  def method_for_frontend
  end

  def method_for_backend
  end
end
```

上面代码在Monolithic App里无任何问题。在Core Engine里，就会让网站前台和管理后台都加载了不属于自己的逻辑。解决办法就是拥抱[Service Object](https://ruby-china.org/topics/24780)，让Model里只存放共用的逻辑和方法，其他移除到Service Object或Concern里，这样就可以做更细粒度的加载。

## 结论

如果不是多语言混合型团队，没有必要引入MicroService，Core Engine是一个不错的过渡。