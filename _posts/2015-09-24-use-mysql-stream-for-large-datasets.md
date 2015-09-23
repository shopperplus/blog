---
layout: post
title: "Use MySQL stream for large datasets"
author: Hooopo
---

## find_each

在ETL过程中，经常需要处较大的表，较复杂的查询，通常会涉及到JOIN几张表和SUM/COUNT/AVG等聚合计算。

之前介绍过[MySQL怎样插的最快](http://shopperplus.github.io/blog/2014/11/08/fastest-way-to-load-data-in-mysql.html)，但在ETL实践过程中，我发现其实大数据集读取和转换才是最耗时的。

我们知道，当数据量稍微大一点的时候，在Rails里使用简单的`User.all`这样的查询进程都会直接死掉，原因主要是结果集量太大，导致`memory float`。Rails 里的一个流行解决方案是使用`find_each`，原理是把一个查询拆成多条查询，每个只返回固定条数的记录。由于实现机制，`find_each` 对带有order和limit的查询及其不友好。最大的问题其实是拆成多条查询之后性能其实降低了很多，尤其是需要JOIN很多表的情况。

```ruby
Person.find_each(start: 2000, batch_size: 2000) do |person|
  person.party_all_night!
end
```

## mysql2 adapter streaming

[Mysql2 Adapter](https://github.com/brianmario/mysql2#streaming) 有一个 `stream` 选项：

> Mysql2::Client can optionally only fetch rows from the server on demand by setting :stream => true. This is handy when handling very large result sets which might not fit in memory on the client.

使用也很简单：

```ruby
require 'mysql2'

client = Mysql2::Client.new(:host => "localhost",
                            :username => "root",
                            :password => "xxx",
                            :database => "crm_dev"
                            )
result = client.query('SELECT id, email FROM shopperplus_customers', :stream => true)
result.each do |row|
  p row
end
```

## mysql client --quick option

另一种方法是使用mysql的命令行客户端，并且带 [--quick 参数](https://dev.mysql.com/doc/refman/5.1/en/mysql-command-options.html#option_mysql_quick)。

这个思路来自 activewarehouse-etl 的 [MySQLStreamer](https://github.com/activewarehouse/activewarehouse-etl/blob/338f1bf9a17eb000559d0a17c664468db0a14c21/lib/etl/control/source/mysql_streamer.rb#L52-L71):

> The MySQL streamer is a helper with works with the database_source in order to allow you to use the --quick option (which stops MySQL) from building a full result set,  also we don't build a full resultset in Ruby - instead we yield a row at a time

我把它简化一下，是这样子的：

```ruby
require 'open3'

mysql_command = %Q{mysql --quick -h localhost -u root -e "SELECT id, email FROM shopperplus_customers" -D crm_dev --password="xxx" -B}

Open3.popen3(mysql_command) do |stdin, out, err, external|
  while line = out.gets do
    columns = line.strip.split("\t")
    keys ||= columns
    result = keys.zip(columns).to_h

    p result
  end
end
```