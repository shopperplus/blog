---
layout: post
title: "ETL with Ruby"
author: Hooopo
---

## Why ETL
前面两篇介绍了数据仓库相关的基本概念和星型模型等常用建模方式。

* [Data Warehouse Concepts and Overview](http://shopperplus.github.io/blog/2015/03/28/data-warehouse-concepts-and-overview.html)
* [Data Warehouse Schema Desgin](http://shopperplus.github.io/blog/2015/04/12/data-warehouse-schema-desgin.html)

这篇主要来谈ETL。星型模型让分析查询变得容易，但存在以下问题：

* 一般先有OLTP系统，后有OLAP系统，分析系统只是辅助工具。
* 我们的事务系统不是按星型模型设计的，数据来源是OLTP类型应用。
* 数据源除了OLTP系统的数据，还需要与其他来源数据进行组合。

所以，需要引入一套ETL流程来把OLTP数据转化为星型模型，然后才能方便分析。

```
 +------------------+          +-----------+      +-----------+      +---------+ 
 |OLTP Data         +--------->+           |      |           |      |         | 
 +------------------+          |           |      |           |      |         | 
                               |           +----> |           +----> |         | 
 +------------------+          | Extract   |      |           |      |         | 
 |Google Analytics  +--------->+           |      |           |      |         | 
 +------------------+          |           |      |           |      |Analytics| 
                               | Transform |      |Star Schema|      |   &&    | 
 +------------------+          |           |      |           |      |Reporting| 
 |CRM Data          +--------->+           |      |           |      |         | 
 +------------------+          | Load      |      |           |      |         | 
                               |           +----> |           +----> |         | 
 +------------------+          |           |      |           |      |         | 
 |Others            +--------->+           |      |           |      |         | 
 +------------------+          +-----------+      +-----------+      +---------+ 
```

那么，ETL是什么？

* Extract data from a source
* Transform the data for storing it in proper format or structure for querying and analysis purpose
* Load it into the target

其实ETL工作我们并不陌生，正像 [thbar](https://github.com/thbar) 所说的 [Rubyists — Are you doing ETL unknowingly?](http://thibautbarrere.com/2015/03/25/rubyists-are-you-doing-etl-unknowingly/),下面这些都属于ETL范畴：

* Write a script to migrate a legacy database to a new schema.
* Automate processing of your data to generate a report.
* Synchronize all or part of the data between 2 systems on a regular basis.
* Prepare your data for indexing/searching.
* Aggregate heterogeneous data sources into one consistent database.
* Clean-up dirty or bogus data.
* Geocode rows in an app to present them through a map app.
* Implement a data export process for your users.

## ETL Tools

和 Ruby 相关的ETL开源工具只有以下几个：

* 与AR结合紧密的activewarehouse-etl：https://github.com/activewarehouse/activewarehouse-etl
* Square出品的轻量级ETL工具：https://github.com/square/ETL
* activewarehouse-etl维护者thbar新作：https://github.com/thbar/kiba

值得说明一下的是，由于ActiveRecord完全是针对OLTP场景设计的ORM工具，我们用AR来导数据一定会觉得巨慢，即使做了各种优化手段。这不是AR的错，是你没选对工具。ETL工具针对这种场景做了优化，一般都是批量加载数据，速度可以说是有了质的提升。可以参考之前写过的[Fastest way to load data in MySQL](http://shopperplus.github.io/blog/2014/11/08/fastest-way-to-load-data-in-mysql.html)。


如果想了解Activewarehosue ETL，这里有一个专门介绍Activewarehouse ETL的slide也值得一看：https://speakerdeck.com/thbar/transforming-data-with-ruby-and-activewarehouse-etl

## Incremental Update & MySQL stream

针对大数据场景，Activewarehouse ETL会提供增量更新和流读取的方式，只需要简单配置：

```ruby
source :in, {
  :type => :database,
  :target => :operational_database
  :table => "people",
  :join => "addresses on people.address_id = addresses.id",
  :select => "people.email, addresses.city, addresses.state, people.created_at"
  :conditions => "people.unsubscribed = 0 AND people.date_of_death IS NULL"
  :order => "people.created_at",
  :new_records_only => "people.updated_at", # 增量导入
  :mysqlstream => true                      # 按流的方式
},
[
  :email,
  :city,
  :state
]
```

## 处理缓慢变化维度

维度建模的数据仓库中，有一个概念叫Slowly Changing Dimensions，中文一般翻译成“缓慢变化维”，经常被简写为SCD。缓慢变化维的提出是因为在现实世界中，维度的属性并不是静态的，它会随着时间的流失发生缓慢的变化。这种随时间发生变化的维度我们一般称之为缓慢变化维，并且把处理维度表的历史变化信息的问题称为处理缓慢变化维的问题，有时也简称为处理SCD的问题。常见类型有两种（[Wikipedia](http://en.wikipedia.org/wiki/Slowly_changing_dimension) 里面介绍了6种）：

### 类型1 (Type 1): 覆盖旧记录。
有些 Dimension 表从业务上讲不需要保存历史记 录或者只需要对原有记录进行修改。比如说 Customer 表中有 Customer 地址的属性,原有的地址输入错误我们需要修改这个属性而 不需要对原有的错误地址进行保存,这个时候 就可以使用 SCD Type 1

### 类型2 (Type 2): 增加新记录。
Type 2 是精确 捕获Dimension 表历史变化的一种标准的方法,它通过对数据源表的Change Data Capture (CDC) 机制来捕获数据源的变化,然后在 Dimension 表中插入一个新的记录再使旧的相应的记录时效。

缓慢变化维度一般设计成这样的结构：

```
# customer dimension
+-----------------+------------+------+-----+---------+----------------+
| id              | int(11)    | NO   | PRI | NULL    | auto_increment |
| customer_key    | int(11)    | YES  | MUL | NULL    |                |
| customer_type   | int(11)    | YES  | MUL | 1       |                |
| effective_start | datetime   | YES  |     | NULL    |                |
| effective_end   | datetime   | YES  |     | NULL    |                |
| latest_version  | tinyint(1) | YES  |     | 1       |                |
+-----------------+------------+------+-----+---------+----------------+
```

Activewarehouse ETL 对SCD问题也提供了解决方案。

## 案例

https://github.com/hooopo/etltest

上面是一个基于Activewarehouse ETL和Activewarehouse的Rails项目，演示从ETL到分析报表展示功能如何实现。
￼
最后，如果你了解了ETL的概念，以后看什么都是ETL了...
