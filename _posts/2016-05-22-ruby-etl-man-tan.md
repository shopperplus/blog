---
layout: post
title: "Ruby ETL 漫谈"
author: Hooopo
---

## [activewarehouse-etl](https://github.com/activewarehouse/activewarehouse-etl)
ActiveWarehouse ETL 应该是最早的Ruby ETL工具，并且十分强大，支持各种应用场景，但缺点是年久失修，没有再维护。ActiveWarehosue ETl目标是解决全部ETL相关的需求，是一个重量型工具，一般新的ETL工具的功能很少是他没有的，但其内部结构十分复杂，并且依赖ActiveRecord。另外一个缺点是，其支持的Source和Destination太过老旧，比如，对Postgresql不友好。

## [kiba](https://github.com/thbar/kiba)

Kiba 是 ActiveWarehouse ETL 的维护者新开发的一个Gem，与 ActiveWarehouse ETL恰恰相反，Kiba 是一个轻量级的ETL工具。目前非常活跃，文档丰富，并且确实很轻，几百行代码勾勒出ETL的基本结构。但缺点呢，也很明显，它空无一物，什么都没提供，就是说，如果你的项目想用 Kiba 来解决实际ELT问题还差很远，需要自己去实现很多功能，如果不是很熟悉，要踩的坑还很多。另外，Kiba和Sidekiq一样，提供Kiba Pro，收费版，据说功能很多，支持以下特性：

* multi-threading (and later, multi-machines) - commonly requested
* built-in sources/transforms/destinations for common tasks
lookups
* upserts / bulk load modules
* connectors optimized for parallelism (HTTP pagination extraction)
* connectors for the cloud (RedShift, ...)
* built-in helpers for common operations (debugging, limiting, caching...)
* premium support

## [ETL](https://github.com/square/ETL)

ETL 是 Square 公司开源的一个轻量级的 ETL gem，与 ActiveWarehouse ETL 不同的是，Square ETL直接操作SQL，不依赖ActiveRecord。但可能只是解决Square公司的特定问题，目前只支持的是MySQL Source和Destination，并且Source和Destination必须在同一个Server上。另一个致命的缺点是不支持增量更新：

* https://github.com/square/ETL/issues/11
* https://github.com/square/ETL/issues/13

## [embulk](https://github.com/embulk/embulk)

![](https://ruby-china-files.b0.upaiyun.com/photo/2016/3fdd6d582cd883fa3e7725ccf6b757d3.png)

Embulk 的特点是其丰富的插件和并行处理能力。但其依赖JRuby和Java让Ruby开发者上手很难，并且其体系相当复杂，如果不是数据量巨大就不用考虑了。

## [kiba-plus](https://github.com/hooopo/kiba-plus)

Kiba Plus是一个基于Kiba的增强，包含常用的Source和Destination实现，例如MySQL、Postgresql、CSV等。它具有以下优点：

* 支持Postgresql：其实支持Postgresql也就意味着很容易支持基于Pg的其他数据库，比如Greenplum、CitusData、Redshift等。题外话，MySQL和Pg在OLTP领域可以说不相上下，但在OLAP领域MySQL已经被Pg甩出一条街了，支持Pg意义重大。
* Fast Load && Insert：由于不依赖ActiveRecord和Sequel之类ORM工具，直接使用mysql2和pg gem操作SQL，能够利用更多数据库自身特性，比如streaming和mysql load infile or pg copy来更快的读取和插入。
* Incremental Insert：增量更新是ETL中不可缺少的部分，如果没有增量更新，即使再快的插入速度和再强的并行处理能力，随着数据量增长，也不可能每次把所以数据都重新插入一遍。

一个从MySQL导入到Pg的例子:

```ruby
require 'kiba/plus'

SOURCE_URL = 'mysql://root@localhost/shopperplus'

DEST_URL   = 'postgresql://hooopo@localhost:5432/crm2_dev'

source Kiba::Plus::Source::Mysql, { :connect_url => SOURCE_URL,
                           :query => %Q{SELECT id, email, 'hooopo' AS first_name, 'Wang' AS last_name FROM customers}
                         }

destination Kiba::Plus::Destination::PgBulk2, { :connect_url => DEST_URL,
                                :table_name => "customers",
                                :truncate => true,
                                :columns => [:id, :email, :first_name, :last_name],
                                :incremental => false
                              }

post_process do
  result = PG.connect(DEST_URL).query("SELECT COUNT(*) AS num FROM customers")
  puts "Insert total: #{result.first['num']}"
end
```
执行：

```
bundle exec kiba customer_mysql_to_pg.etl
```
输出：

```
# I, [2016-05-16T01:53:36.832565 #87909]  INFO -- : TRUNCATE TABLE customers;
# I, [2016-05-16T01:53:36.841770 #87909]  INFO -- : COPY customers (id, email, first_name, last_name) FROM STDIN WITH DELIMITER ',' NULL '\N' CSV
# Insert total: 428972
```