---
layout: post
title: "Fastest Way To Load Data In MySQL"
author: Hooopo
---

![fast](http://www.runrares.ro/_imgArticol/viteza.jpg)

[Mass inserting data in Rails without killing your performance](https://www.coffeepowered.net/2009/01/23/mass-inserting-data-in-rails-without-killing-your-performance/) 这篇文章里提到Rails快速批量插入数据的几种方式：

1. Use transactions
2. Get down and dirty with the raw SQL
3. A single mass insert
4. Use INSERT statements with multiple VALUES lists with [ar-extensions](https://github.com/zdennis/ar-extensions) OR [activerecord-import](https://github.com/zdennis/activerecord-import)

第四种方式是AR方式的70倍左右，已经很快了，还有更快的吗？

From MySQL Doc:
> When loading a table from a text file, use LOAD DATA INFILE. This is usually 20 times faster than using INSERT statements

SQL语句：

```
(15955.1ms)  LOAD DATA LOCAL INFILE '/Users/hooopo/data/out/product_sales_facts.txt'
REPLACE INTO TABLE product_sale_facts FIELDS TERMINATED BY ',' (`id`,`date_id`,`order_id`,`product_id`,`address_id`,`unit_price`,`purchase_price`,`gross_profit`,`quantity`,`channel_id`,`gift`)
```

插入行数：

```
wc -l < /Users/hooopo/data/out/product_sales_facts.txt
 1271413
```
大概130w数据用时16秒，平均每秒插入8w条记录左右。如果想用Gem，可以试试 adapter_extensions 和 load_data_infile：

* https://github.com/activewarehouse/adapter_extensions
* https://github.com/EmmanuelOga/load_data_infile
* https://github.com/jsuchal/activerecord-fast-import

MySQL Doc里关于LOAD INFILE的详细介绍：

* http://dev.mysql.com/doc/refman/5.5/en/load-data.html
* http://dev.mysql.com/doc/refman/5.5/en/insert-speed.html
