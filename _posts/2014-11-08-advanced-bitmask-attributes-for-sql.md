---
layout: post
title: "Advanced Bitmask Attributes for SQL"
author: Hooopo
---
# Advanced Bitmask Attributes for SQL

## Use Case

假设你正在为一个电商网站开发这样一种功能，商品可以在后台分配到不同的频道里显示（当然，一个商品可以在多个频道显示）。
另外有一个前提是，频道数目不多（10个以内）。

## Bitmask

最常规的做法是加两张表，一个频道表，另一个是频道和商品中间表，标记频道和商品对应关系。这种做法的缺点是查询的时候需要JOIN两张表。再加上对商品本身的属性进行过滤和排序，查询会非常慢。

一个简化的办法就是使用Bitmask Attributes。

```ruby
class Product < AR
  Channel1 = 1 << 0 # 1
  Channel2 = 1 << 1 # 2
  Channel3 = 1 << 2 # 4
  Channel4 = 1 << 3 # 8
end

# == Schema Information
#
# Table name: products
#
#  id                        :integer          not null, primary key
#  channels                  :integer
#  retail_price              :decimal(8, 2)    default(0.0), not null
#  purchase_price            :decimal(8, 2)    default(0.0), not null
#  image                     :string(255)
#  weight                    :integer
```
### 存储

* 如果一个商品能在Channel1和Channel2里显示： channels => 1 + 2 = 3
* 如果一个商品能在Channel2和Channel2和Channel3显示： channels => 1 + 2 + 4 = 7
* 以此类推

### 查询

* 查询能在Channel1显示的商品: `Product.where("channels & 1 = 1")`
* 查询能在Channel1和Channel2显示的商品：`Product.where("channels & 3 = 3")`
* 查询不能在Channel3显示的商品：`Product.where("channels & 4 = 0")`

### 优缺点

* 优点：更少的字段、更少的表解决了问题
* 缺点：所有需要过滤channels字段的查询都为全表扫描，无法利用索引

### Advanced Bitmask Attributes

继续使用Bitmask的思路，但是把查询转化成IN查询，有效利用索引。

* 查询能在Channel1显示的商品:
```ruby
Product.where(:channels => (1...(1 << 4)).select{|x| x & 1 == 1})
# 产生SQL
SELECT `products`.* FROM `products`  WHERE `products`.`channels` IN (1, 3, 5, 7, 9, 11, 13, 15)"
```
* 查询能在Channel1和Channel2显示的商品：
```ruby
Product.where(:channels => (1...(1 << 4)).select{|x| x & 3 == 3})
# 产生SQL
SELECT `products`.* FROM `products`  WHERE `products`.`channels` IN (3, 7, 11, 15)
```
* 查询不能在Channel3显示的商品：
```ruby
Product.where(:channels => (1...(1 << 4)).select{|x| x & 4 == 0})
# 产生SQL
SELECT `products`.* FROM `products`  WHERE `products`.`channels` IN (1, 2, 3, 8, 9, 10, 11)
```

## 结论

* 中间表方案：需要额外的JOIN Table
* Bitmask Attributes：需要Full Table Scan
* Advanced Bitmask Attributes：No JOIN,No Full Table Scan.