---
layout: post
title: "Heroku-style config management with capistrano and dotenv"
author: Hooopo
---

# 12-factor Style

通常，应用的 *配置* 在不同 [部署](http://12factor.net/zh_cn/codebase) (预发布、生产环境、开发环境等等)间会有很大差异。这其中包括：

* 数据库，Memcached，以及其他 [后端服务](http://12factor.net/zh_cn/backing-services) 的配置
* 第三方服务的证书，如 Amazon S3、Twitter 等
* 每份部署特有的配置，如域名等

有些应用在代码中使用常量保存配置，这与 `12-factor` 所要求的**代码和配置严格分离**显然大相径庭。配置文件在各部署间存在大幅差异，代码却完全一致。

![](http://upload-images.jianshu.io/upload_images/505-2cdf658452a1301a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

判断一个应用是否正确地将配置排除在代码之外，一个简单的方法是看该应用的基准代码是否可以立刻开源，而不用担心会暴露任何敏感的信息。

`12-Factor` 推荐将应用的配置存储于 *环境变量* 中（ *env vars*, *env* ）。环境变量可以非常方便地在不同的部署间做修改，却不动一行代码。更多关于`12-factor` 的内容请访问 [12factor.net](http://12factor.net).

# Rails Style 存在的问题

Rails中大部分实践是遵守了 12-Factor 规则，除了日志和配置。Rails Way的配置方式是yml文件，例如：database.yml、secret.yml等。简单的应用使用yml文件和 `12-factor` 推荐的环境变量其实区别不大，不过当项目不断膨胀，配置数量也随着增加，yml方式变得越来越难以维护。yml存在的问题表现在：

1. 需要不断的修改.gitignore文件
2. 搭建各种环境（dev、production、test、staging）时需要不断copy example文件
3. 配置散落各处
4. 每增加yml文件，需要在应用里增加load和parse逻辑
5. 不小心会被签进版本控制里

下面是目前项目的配置文件，每次搭建环境修改这些配置文件是最痛苦的事情...

```
shared/config$ ls |grep yml
s3.yml
redis.yml
twilio.yml
paypal.yml
sunspot.yml
database.yml
icontact.yml
http_auth.yml
beanstream.yml
commercehub.yml
canada_post.yml
google_content_api.yml
```

# 系统环境变量

最直接的方式是使用shell 环境变量：

```bash
$ export TWILIO_ACCOUNT_SID=AC1234...
$ export TWILIO_AUTH_TOKEN=abc12...
```

在Ruby中使用ENV读取：

```ruby
puts ENV['TWILIO_ACCOUNT_SID']
puts ENV['TWILIO_AUTH_TOKEN']
```
shell环境变量由于没有持久化，又引出了 .bashrc、.bash_profile、~/.profile、/etc/environment等方式。都不是很完美，权限问题、login shell问题的坑都等着你呢。最重要的是，上面提到这些都不方便应付单机部署多个应用的场景。

# Envyable、dotenv、Figaro

[Envyable](https://github.com/philnash/envyable)、[dotenv](https://github.com/bkeepers/dotenv)、[Figaro](https://github.com/laserlemon/figaro)等工具，在应用程序中把配置注入到`ENV`中避免了上面提到的各种问题。Envyable、dotenv、Figaro 无论是实现上还是使用上其实大同小异，下面只拿dotenv来说：

In Gemfile:

```ruby
gem 'dotenv-rails', :require => 'dotenv/rails-now'
```
config/application.rb

```ruby
Bundler.require(*Rails.groups)
Dotenv::Railtie.load
HOSTNAME = ENV['HOSTNAME']
```

.env 文件：

```
S3_BUCKET=YOURS3BUCKET
SECRET_KEY=YOURSECRETKEYGOESHERE
```

gitignore忽略.env，并且cap设置软链：

```ruby
set :linked_files, fetch(:linked_files, []).push('config/database.yml', '.env')
```
调用的时候仍然是 `ENV['SECRET_KEY']`。当然，dotenv还支持多环境模式，比如 `.env.production` 文件只对production环境生效。

# Heroku-style

dotenv已经很完美了，但用过Heroku的都知道这还不够，看一下Hero哭设置环境变量的方式：

```bash
heroku config:set GITHUB_USERNAME=joesmith
Adding config vars and restarting myapp... done, v12GITHUB_USERNAME: joesmith

$ heroku config
GITHUB_USERNAME: joesmith
OTHER_VAR: production

$ heroku config:get
GITHUB_USERNAMEjoesmith

$ heroku config:unset GITHUB_USERNAME
Unsetting GITHUB_USERNAME and restarting myapp... done, v13
```

[capistrano-twelvefactor](https://github.com/hooopo/capistrano-twelvefactor) + dotenv 可以打造出和Heroku一样酷的体验。

详细的步骤capistrano-twelvefactor上面都有写，下面只说和dotenv配合需要做的：

```ruby
# config/deploy/production.rb
set :environment_file, deploy_path.join("shared/.env")

after 'config:set',   "deploy:symlink:linked_files"
after 'config:unset', "deploy:symlink:linked_files"
```
然后就可以使用这几个命令查看和修改环境配置了：

* bundle exec cap production config:list
* bundle exec cap production config:set[FOO=bar]
* bundle exec cap production config:unset[FOO]

注：上面需要capistrano 3 + dotenv，Envyable和Figaro目前无法使用Heroku style.

# Multi Server && Apps

对于单台服务器的应用，capistrano-twelvefactor + dotenv 足以应付，同时，由于capistrano支持集群部署，单个应用多服务器其实也是可以搞定的。但有时候不同应用之间其实也需要共享配置文件的，比如，S3配置变化了，依赖这个帐号的所有应用都应该得到同步。另外的一个问题是，对于多服务器场景，配置存储在.env会出现服务器之间配置不同步现象。如果要解决上述问题，可能引入[etcd](https://github.com/coreos/etcd)或 [zookeeper](https://zookeeper.apache.org/)是一个不错的选择。