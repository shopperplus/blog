---
layout: post
title: "Elasticsearch 开箱笔记"
author: XguoX
---

和 Solr 一样, **Elasticsearch** 是一个基于全文搜索引擎 Apache Lucene(TM) 基础上的搜索引擎. 没怎么用 Solr, 所以不好做各方面比较.  而厂里的 CRM 系统用的就是 **Elasticsearch**.  **Elasticsearch** 不仅可以支持全文搜索, 同时也支持类似关系数据库的结构化查询. 它可以让你搜索并利用那些在数据库中难以查询的数据. Elasticsearch 还可以做聚合(aggregations), 可以在数据上进行复杂的分析统计.

 **Elasticsearch** 的 RESTful API是基于 HTTP 协议, 以 JSON 作为数据格式的. 所以, 经常写出, 语法上经常写出跟 JavaScript 那样一大串开闭括号.

形如:

```json

{
  "cluster_name" : "elasticsearch_xguox",
  "nodes" : {
    "8fM1_dseTUWL74TiDTCPiA" : {
      "name" : "Thinker",
      "transport_address" : "inet[/127.0.0.1:9300]",
      "host" : "XguoXs-MacBook-Pro.local",
      "version" : "1.6.0",
      "build" : "cdd3ac4",
      "http_address" : "inet[/127.0.0.1:9200]",
      "settings" : {
        "path" : {
          "data" : "/usr/local/var/elasticsearch/",
          "logs" : "/usr/local/var/log/elasticsearch",
          "plugins" : "/usr/local/var/lib/elasticsearch/plugins",
          "home" : "/usr/local/Cellar/elasticsearch/1.6.0"
        },
        "cluster" : {
          "name" : "elasticsearch_xguox"
        },
        "name" : "Thinker",
        "index" : {
          "analysis" : {
            "analyzer" : {
              "ik_smart" : {
                "type" : "ik",
                "use_smart" : "true"
              },
              "ik" : {
                "type" : "org.elasticsearch.index.analysis.IkAnalyzerProvider",
                "alias" : [ "ik_analyzer" ]
              },
              "ik_max_word" : {
                "type" : "ik",
                "use_smart" : "false"
              }
            }
          }
        },
        "client" : {
          "type" : "node"
        },
        "foreground" : "yes",
        "config.ignore_system_properties" : "true",
        "config" : "/usr/local/Cellar/elasticsearch/1.6.0/config/elasticsearch.yml",
        "script" : {
          "inline" : "on",
          "indexed" : "on"
        },
        "network" : {
          "host" : "127.0.0.1"
        }
      }
    }
  }
}
```

而且, 经常是嵌套的很深, 不是漏了逗号就是冒号或者开闭大括号 ヾ(´･ ･｀｡)ノ"

**好吧, 其实上面那段不是正规的数据格式, 只是执行**

`curl "localhost:9200/_nodes/settings?pretty=true"`

得到的结果. 主要是安装完 Elasticsearch 后的一些相关配置信息等等.

#### 安装

Mac OS X 上直接用 homebrew 就可以 install 了

```shell

Distributed search & analytics engine
https://www.elastic.co/products/elasticsearch
/usr/local/Cellar/elasticsearch/1.6.0 (34 files, 29.6M) *
  Built from source
From: https://github.com/Homebrew/homebrew/blob/master/Library/Formula/elasticsearch.rb
==> Caveats
Data:    /usr/local/var/elasticsearch/elasticsearch_xguox/
Logs:    /usr/local/var/log/elasticsearch/elasticsearch_xguox.log
Plugins: /usr/local/Cellar/elasticsearch/1.6.0/libexec/plugins/
Config:  /usr/local/etc/elasticsearch/
plugin script: /usr/local/Cellar/elasticsearch/1.6.0/libexec/bin/plugin

To reload elasticsearch after an upgrade:
  launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.elasticsearch.plist
  launchctl load ~/Library/LaunchAgents/homebrew.mxcl.elasticsearch.plist
Or, if you don't want/need launchctl, you can just run:
  elasticsearch
```

VPS(Ubuntu 14.04) 上可以参考[这里](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-on-an-ubuntu-vps)

![](http://ww4.sinaimg.cn/large/62fdd4d5jw1f293vl4rpmj20p80foac3.jpg)

BTW,

elasticsearch.yml 里面的 `network.host: 127.0.0.1`

默认的 Elasticsearch 是绑定了只允许本地 `127.0.0.1` 访问的, 在本地或者一般的生产环境足够了, 但是, 如果是一个集群跑在多个服务器的话就要在这里设置.

#### 添加插件

![](http://ww2.sinaimg.cn/large/62fdd4d5jw1f29419fwwkj219i0gs420.jpg)

##### 中文分词

Elasticsearch 默认的分词对英语支持的已经挺好的, 但是对中文的分词支持却很渣. 貌似几乎没分词可言. 默认的standard analyser 直接拆分成单个字了.  有个 smartcn 的 analyser 评价也一般.

所以, 用的比较多的插件是 [ik](https://github.com/medcl/elasticsearch-analysis-ik) 和 [mmseg](https://github.com/medcl/elasticsearch-analysis-mmseg). 对应的 Readme 都介绍的挺详细的.

注意拷贝词库就是了, 之前就是搞了个乌龙折腾半天. 直接把 [mmseg](https://github.com/medcl/elasticsearch-analysis-mmseg/tree/master/config/mmseg) 或者 [ik](https://github.com/medcl/elasticsearch-analysis-ik/tree/master/config/ik) 整个拷到跟 elasticsearch.yml  同一层目录下就行 = . =

然后配置一下

```yml

index:
  analysis:
    analyzer:
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
      ik_max_word:
          type: ik
          use_smart: false
      ik_smart:
          type: ik
          use_smart: true
# index:
#   analysis:
#     analyzer:
#       mmseg:
#           alias: [news_analyzer, mmseg_analyzer]
#           type: org.elasticsearch.index.analysis.MMsegAnalyzerProvider
# index.analysis.analyzer.default.type : "mmseg"
# index:
#   analysis:
#     tokenizer:
#       mmseg_maxword:
#           type: mmseg
#           seg_type: "max_word"
#       mmseg_complex:
#           type: mmseg
#           seg_type: "complex"
#       mmseg_simple:
#           type: mmseg
#           seg_type: "simple"
```

几乎全都是开箱即用的节奏 Σ(￣。￣ノ)ノ

--------

##### Related:

[Elasticsearch on Rails](http://xguox.me/elasticsearch-rails.html)

[Elasticsearch More Like This 搜索](http://xguox.me/elasticsearch-more-like-this.html)

[Elasticsearch Aggregations 聚合分析](http://xguox.me/elasticsearch-aggregations.html)

[Upgrade Elasticsearch to 2.3](http://xguox.me/upgrade-elasticsearch-2-3.html)

[Elasticsearch Scroll (Ruby)](http://xguox.me/elasticsearch-scroll.html)

