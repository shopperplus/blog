---
layout: post
title: "开源 BI 工具 Metabase 简介"
author: Hooopo
---

> Metabase is the easy, open source way for everyone in your company to ask questions and learn from data.

这是 [Metabase](http://www.metabase.com/) 官网上的介绍。BI 工具其实非常多，但却没有一种适合所有场景，各种产品的定位也各不相同。个人觉得 Metabase 相对于其他 BI 产品具有以下特性：

## 不懂 SQL  也可以很快掌握业务数据

一般来说，BI 产品的用户都是业务人员（大部分不懂 SQL ），Metabase 把数据分析常用的查询通过通过一个易于操作的界面来操作，这样，不懂 SQL 的业务人员也可以快速掌握业务数据。 下面举个简单的例子来看一下，如果销售人员想知道每月的订单数量该如何操作：

![不懂 SQL  也可以很快掌握业务数据](https://camo.githubusercontent.com/6025405b0dd84becd1f80851f6f2914d7c7bb595/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f3530352d323762363033623861393133373563342e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

只需要点几下，就可以得出一个直观的可视化结果，当然，除了折线图之外，还可以选择饼图、柱状图、表格等。对于查询的结果，可以导出到 CSV。

看到这里，一定会有同学发现，这种单表查询统计太简单，真实情况的业务分析可能需要 JOIN 几张表或使用一些 SQL function 才能得到结果。然而，对于熟悉 SQL 的业务或开发人员，也可以通过 SQL 来获得业务数据，如图：

![使用 SQL 获得复杂的业务数据](https://camo.githubusercontent.com/9bbef2d64b360b2ee29c49a87ada01f821300548/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f3530352d373938663331393938656135353461312e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

## 业务数据与团队共享

上面这些业务数据都可以保存并且分享给团队里其他成员。除此之外，团队中开发人员也可以把复杂的查询写好，把结果共享给业务人员。这是团队共享业务数据的应用场景。

![团队共享业务数据](https://camo.githubusercontent.com/da90d2536fef4fba1950acd2b4561db42c0a5f33/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f3530352d393363646231303032626434623963372e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

## 开源、部署方便

Metabase 支持多种数据源，包括MySQL、Postgresql 和 H2，看 Roadmap 即将支持的是 Redshift。

部署 Metabase 同样非常简单，在 Mac 上下载之后点击即用，其他平台也只需运行一个 jar 包而已。同时支持的部署环境是：

* Heroku
* Amazon Web Service
* Docker

## 与 ChartIO 的对比

![ChartIO](https://camo.githubusercontent.com/b5c3239256c59a3b0c16b002cdc0782eef5870e2/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f3530352d326433393433613234623264356665392e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

[ChartIO](https://chartio.com) 支持各种数据源，通过拖拽方式来获取业务数据，并生成图表，从这方面讲，ChartIO 和 Metabase 的定位是相同的。不过 ChartIO是一个 收费的 SaaS 服务，而 Metabase 是开源免费的软件程序，他们之间的关系有点像 Github 和 Gitlab，不过从目前的状况看，ChartIO 成熟度要优于 Metbase 很多。

## 与 ETL 结合

虽说 Metabase 可以让不懂 SQL 的业务人员轻松分析业务数据。但由于 OLTP 数据库的结构本身是不利于业务分析的，更不要说数据量大的情况，OLTP 数据库 JOIN 几张表之后的查询效率更让人难以接受。

一个拟补的方案是，开发人员只需要做一些简单的 [ETL](http://shopperplus.github.io/blog/2015/04/15/etl-with-ruby.html) 操作，把 OLTP 库先转化为适合分析的[星型模型](http://shopperplus.github.io/blog/2015/04/12/data-warehouse-schema-desgin.html)。

对于业务分析方面还没有任何基础的公司来说，Metabase 也许是一个不错的开始。