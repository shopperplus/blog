---
layout: post
title: "中间表一定要加唯一索引"
author: Hooopo
---

我们系统大量使用了`has_and_belongs_to_many`这样的关联方式做关联。但却没有为中间表加惟一索引。导致重复数据的产生。比如printer model和产品关联表：

### Identify Duplicate Data

```
SELECT product_id, printer_model_id FROM printer_models_products GROUP BY product_id,printer_model_id HAVING count(*) > 1;
+------------+------------------+
| product_id | printer_model_id |
+------------+------------------+
|     303994 |          3008631 |
|     303994 |          3008632 |
|     303994 |          3008637 |
|     303994 |          3008640 |
|     303994 |          3008641 |
|     303994 |          3008642 |
|     303995 |          3008631 |
|     303995 |          3008632 |
|     303995 |          3008637 |
|     303995 |          3008640 |
|     303995 |          3008641 |
|     303995 |          3008642 |
|     304010 |          1000983 |
|     304010 |          1000984 |
|     308482 |          1000983 |
|     308482 |          1000984 |
|     318461 |          3008631 |
|     318461 |          3008632 |
|     318461 |          3008637 |
|     318461 |          3008640 |
|     318461 |          3008641 |
|     318461 |          3008642 |
|     318462 |          3008631 |
|     318462 |          3008632 |
|     318462 |          3008637 |
|     318462 |          3008640 |
|     318462 |          3008641 |
|     318462 |          3008642 |
+------------+------------------+
```

进一步验证：

```
select * from printer_models_products where product_id = 303994 and printer_model_id = 3008631;
+------------+------------------+
| product_id | printer_model_id |
+------------+------------------+
|     303994 |          3008631 |
|     303994 |          3008631 |
+------------+------------------+
```

### Is It A Big Deal?


```
printer_model = PrinterModel.find 3008631
printer_model.products.uniq.size
  Product Load (0.9ms)  SELECT `products`.* FROM `products` INNER JOIN `printer_models_products` ON `products`.`id` = `printer_models_products`.`product_id` WHERE `printer_models_products`.`printer_model_id` = 3008631
=> 13
[12] pry(main)> printer_model.products.size
=> 17
```
也就是说，如果中间表里存在重复数据，会导致查询结果有重复数据，想要得到正确的结果必须做去重操作（distinct或查询之后再uniq），无论用何种方式都给代码编写和执行效率带来了麻烦。
所以，最直接的办法是从源头限制重复数据，给中间表加惟一索引。但是，对于已经重复的脏数据，如何去修复：

### How To Fix

以products_suppliers这个表为例：

* 增加id列，设为primary_key
* 删除重复数据：`DELETE f FROM products_suppliers AS f JOIN products_suppliers AS g ON f.product_id = g.product_id AND f.supplier_id = g.supplier_id AND f.id > g.id;`
* 添加惟一索引（只有上面删除重复成功才能添加惟一索引）

```ruby
class AddIdInProductsSuppliers < ActiveRecord::Migration
  def change
    add_column :products_suppliers, :id, :primary_key
    remove_index :products_suppliers, :name => :index_products_suppliers_product
    remove_index :products_suppliers, :name => :index_products_suppliers_supplier
    # 删除重复记录
    ActiveRecord::Base.connection.execute("DELETE f FROM products_suppliers AS f JOIN products_suppliers AS g ON f.product_id = g.product_id AND f.supplier_id = g.supplier_id AND f.id > g.id;")
    add_index :products_suppliers, [:product_id, :supplier_id], :unique => true
  end
end
```