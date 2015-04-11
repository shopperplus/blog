---
layout: post
title: "Data Warehouse Schema Desgin"
author: Hooopo
---

之前介绍了数据仓库相关的一些基本概念：维度、事实、维度层次、上钻、下钻等。这篇主要来说一下数据仓库系统如何建模。

## OLTP vs OLAP

分析型系统和操作型系统具有完全不同的目的。操作型系统支持业务的执行过程，而分析型系统支持对业务过程的评价。因此，指导种系统的设计原则也不同。

操作型系统直接支持业务过程的执行。它通过获取业务的事件和细节来构建业务的活动记录。例如，销售系统的订单、发货、支付等；人力系统的雇员雇佣和升迁信息；代码托管系统的commit和pull request信息等。由这些系统记录的活动通常成为事务，而这类系统本身通常成为联机事务处理（OLTP）系统，或简称为事务系统。

操作型系统关注执行过程，因此在事情发生改变时可能需要更新相关数据，并在数据操作有效期结束后清除或归档数据。例如，当一个客户地址发生变动时，地址数据被简单的重写了。

在关系数据库设计领域，广泛被认可的最佳操作型系统模式设计方法是第三范式。

与操作型系统关注业务执行过程不同，分析型系统主要支持对业务过程的评价。

* 本月订单趋势与上个月相比有何不同？
* 与本季度的目标相比，这种趋势说明什么问题？
* 某一个营销策略对销售有何影响?
* 谁是我们的最佳客户？

这些问题涉及到对整个订单流程的度量，无法从单个订单中获得答案。

在分析系统中，不需要创建或修改信息。在操作型系统中不再使用的历史数据对分析型系统来说仍然很重要，这一点之后要提到的`缓慢变化维度`问题中会有更详细的解释。

下面表格是对OLTP和OLAP主要差别的总结：

| | 操作型系统|分析型系统|
|---|---|---|
|目的|执行业务过程|度量业务过程|
|交互类型|插入、更新、查询、删除|查询|
|交互范围|单个事务|聚合事务|
|时间关注|当前的|当前的和历史的|
|设计优化|更新并发性|高性能查询|
|设计原则|基于第三范式（3NF）的实体-关系（ER）设计|维度设计（星型模型）|


## 星型模式（Star Schema）

针对关系型数据库的维度设计被称为星型模式。相关的维度组合成维度表中的列，事实则存储在事实表的各个列中。星型模式的这个称谓来自于其表现形式：当与置于中心位置的事实表连线时，整个模式看起来像星状物，如图：

![star schema](https://camo.githubusercontent.com/b62f85699da533d25434de94ce3e556d9d5d84b4/68747470733a2f2f707974686f6e686f737465642e6f72672f63756265732f5f696d616765732f736368656d615f737461722e706e67)

在星型模型基础上，将维度表做规范化设计，又衍生出了雪花模型，如图：

![flake schema](https://camo.githubusercontent.com/a2782e4fc831b7b8e1948233458112d6c9954798/68747470733a2f2f707974686f6e686f737465642e6f72672f63756265732f5f696d616765732f736368656d615f736e6f77666c616b652e706e67)

下面拿一个简单的订单系统举例，来看一下维度和事实表的设计：

```
# 地址维度表(adderss_dimension)
+---------+--------------+------+-----+---------+----------------+
| Field   | Type         | Null | Key | Default | Extra          |
+---------+--------------+------+-----+---------+----------------+
| id      | int(11)      | NO   | PRI | NULL    | auto_increment |
| country | varchar(255) | YES  | MUL | NULL    |                |
| state   | varchar(255) | YES  | MUL | NULL    |                |
| city    | varchar(255) | YES  | MUL | NULL    |                |
+---------+--------------+------+-----+---------+----------------+

# 发货方式维度表(shipping_method_dimension)：
+-------+--------------+------+-----+---------+----------------+
| Field | Type         | Null | Key | Default | Extra          |
+-------+--------------+------+-----+---------+----------------+
| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
| name  | varchar(255) | YES  | MUL | NULL    |                |
+-------+--------------+------+-----+---------+----------------+

# 订单来源维度表（source_dimension）:
+-------+--------------+------+-----+---------+----------------+
| Field | Type         | Null | Key | Default | Extra          |
+-------+--------------+------+-----+---------+----------------+
| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
| name  | varchar(255) | YES  | MUL | NULL    |                |
+-------+--------------+------+-----+---------+----------------+

# 日期维度表(date_dimension)：
+-------------------------------+--------------+------+-----+---------+----------------+
| Field                         | Type         | Null | Key | Default | Extra          |
+-------------------------------+--------------+------+-----+---------+----------------+
| id                            | int(11)      | NO   | PRI | NULL    | auto_increment |
| date                          | varchar(255) | YES  | MUL | NULL    |                |
| full_date_description         | text         | YES  |     | NULL    |                |
| calendar_week                 | varchar(255) | YES  | MUL | NULL    |                |
| calendar_week_number_in_year  | int(11)      | YES  | MUL | NULL    |                |
| calendar_month_name           | varchar(255) | YES  | MUL | NULL    |                |
| calendar_month_number_in_year | int(11)      | YES  | MUL | NULL    |                |
| calendar_year_month           | varchar(255) | YES  | MUL | NULL    |                |
| calendar_quarter              | varchar(255) | YES  | MUL | NULL    |                |
| calendar_year_quarter         | varchar(255) | YES  | MUL | NULL    |                |
| calendar_year                 | varchar(255) | YES  | MUL | NULL    |                |
| sql_date_stamp                | date         | YES  | MUL | NULL    |                |
+-------------------------------+--------------+------+-----+---------+----------------+

# 订单事实表(order_facts)：
+--------------------+--------------+------+-----+---------+----------------+
| Field              | Type         | Null | Key | Default | Extra          |
+--------------------+--------------+------+-----+---------+----------------+
| id                 | int(11)      | NO   | PRI | NULL    | auto_increment |
| date_id            | int(11)      | YES  | MUL | NULL    |                |
| shipping_method_id | int(11)      | YES  | MUL | NULL    |                |
| customer_id        | int(11)      | YES  | MUL | NULL    |                |
| payment_method_id  | int(11)      | YES  | MUL | NULL    |                |
| zip_id             | int(11)      | YES  | MUL | NULL    |                |
| address_id         | int(11)      | YES  | MUL | NULL    |                |
| source_id          | int(11)      | YES  | MUL | NULL    |                |
| tax                | decimal(8,2) | YES  |     | 0.00    |                |
| shipping_cost      | decimal(8,2) | YES  |     | 0.00    |                |
| total              | decimal(8,2) | YES  |     | 0.00    |                |
| sub_total          | decimal(8,2) | YES  |     | 0.00    |                |
| refund_amount      | decimal(8,2) | YES  |     | 0.00    |                |
| coupon_discount    | decimal(8,2) | YES  |     | 0.00    |                |
| gross_profit       | decimal(8,2) | YES  |     | 0.00    |                |
+--------------------+--------------+------+-----+---------+----------------+
```

最终的星型结构是这样：

```
                                                            
 +---------------+                                 +------+ 
 |        address+----------+      +---------------+date  | 
 +---------------+        +-v------v--+            +------+ 
                          |order facts|                     
 +---------------+        +-^------^--+            +------+ 
 |shipping method+----------+      +---------------+source| 
 +---------------+                                 +------+ 
                                                            
```

这里比较特殊的是时间维度表，时间维度在星型模型里是一个通用维度，粒度只精确到天，这也是由分析型系统对实时性要求不高的特性决定的。其中地址也有做类似处理，在ETL过程中将街道等细节信息去掉，只精确到了城市级别。

## 星型模式的查询

数据库结构已经设计完毕，我们来做一些简单的查询，比如，想知道2013年11月销量最好的前10个城市是什么？

```sql
SELECT address_dimension.country, address_dimension.state, address_dimension.city, SUM(total) as sum_total, date_dimension. calendar_year_month
FROM `order_facts`
INNER JOIN `date_dimension` ON `date_dimension`.`id` = `order_facts`.`date_id`
INNER JOIN `address_dimension` ON `address_dimension`.`id` = `order_facts`. `address_id`
WHERE calendar_year_month = '2013-11'
GROUP BY country, state, city, calendar_year_month
ORDER by sum_total DESC
LIMIT 10
```

结果：

```
+---------+------------------+-------------+-------------+---------------------+
| country | state            | city        | sum_total   | calendar_year_month |
+---------+------------------+-------------+-------------+---------------------+
| Canada  | Ontario          | Toronto     | 23914512.00 | 2013-11             |
| Canada  | Quebec           | Montreal    | 22403280.00 | 2013-11             |
| Canada  | Alberta          | Calgary     | 13671801.00 | 2013-11             |
| Canada  | British Columbia | Vancouver   |  9201099.00 | 2013-11             |
| Canada  | Ontario          | Ottawa      |  8771859.00 | 2013-11             |
| Canada  | Ontario          | Mississauga |  8627691.00 | 2013-11             |
| Canada  | Manitoba         | Winnipeg    |  8308380.00 | 2013-11             |
| Canada  | Alberta          | Edmonton    |  7773006.00 | 2013-11             |
| Canada  | British Columbia | Richmond    |  6602364.00 | 2013-11             |
| Canada  | Quebec           | Lachine     |  5624025.00 | 2013-11             |
+---------+------------------+-------------+-------------+---------------------+
```

从上面的SQL可以发现，设计成星型模式之后，几乎所有多维分析报表问题都可以通过上面一种查询方式得到答案。
