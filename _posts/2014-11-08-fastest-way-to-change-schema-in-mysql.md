---
layout: post
title: "Fastest Way To Change Schema In MySQL"
author: Hooopo
---
在默认情况，Rails的migration会为每个语句单独生成一条SQL来修改表结构。这对新建的表来说是无所谓的，一个create table 或 alter table都是瞬间完成。但是，对于已经存在大量数据的表来说，这样效率极低。刚开始没有注意这样的问题，order表的写法：

```ruby
rename_column :orders, "shipping_first_name", "origin_shipping_first_name"
rename_column :orders, "shipping_last_name", "origin_shipping_last_name"
remove_column :orders, "coupon_value"
...
```
结果就是，每一个rename_column和remove_column执行时间大约4分钟，一共20多个这样的语句，migrate总计执行了近两个小时。因为alter table时mysql需要把整个order表copy to tmp table，而order表是非常庞大的。

解决办法是把上面的所有修改order的migration合并成一个，并且设置bulk 选项。

```ruby
class RenameOldOrderAddressRelatedColumn < ActiveRecord::Migration
  def up
    change_table(:orders, :bulk => true) do |t|
      t.remove :coupon_code
      t.remove :coupon_value
      t.remove :order_promotion_id
      t.rename :order_status_code_id, :state
      t.remove :in_process
      [:first_name,
       :last_name,
       :phone,
       :street_address,
       :city,
       :zip,
       :company,
       :street_address_2,
       :country,
       :state].each do |column|
        t.rename "shipping_#{column}", "origin_shipping_#{column}"
        t.rename "billing_#{column}", "origin_billing_#{column}"
      end
    end
  end
end
```

这样，上面的migration只生成一条SQL语句：

```
ALTER TABLE `orders` DROP `coupon_code`, DROP `coupon_value`, DROP `order_promotion_id`, CHANGE `shipping_first_name` `origin_shipping_first_name` varchar(50) DEFAULT NULL, CHANGE `billing_first_name` `origin_billing_first_name` varchar(50) DEFAULT NULL, CHANGE `shipping_last_name` `origin_shipping_last_name` varchar(50) DEFAULT NULL, CHANGE `billing_last_name` `origin_billing_last_name` varchar(50) DEFAULT NULL, CHANGE `shipping_phone` `origin_shipping_phone` varchar(20) DEFAULT NULL, CHANGE `billing_phone` `origin_billing_phone` varchar(20) DEFAULT NULL, CHANGE `shipping_street_address` `origin_shipping_street_address` varchar(255) DEFAULT NULL, CHANGE `billing_street_address` `origin_billing_street_address` varchar(255) DEFAULT NULL, CHANGE `shipping_city` `origin_shipping_city` varchar(255) DEFAULT NULL, CHANGE `billing_city` `origin_billing_city` varchar(255) DEFAULT NULL, CHANGE `shipping_zip` `origin_shipping_zip` varchar(20) DEFAULT NULL, CHANGE `billing_zip` `origin_billing_zip` varchar(20) DEFAULT NULL, CHANGE `shipping_company` `origin_shipping_company` varchar(255) DEFAULT NULL, CHANGE `billing_company` `origin_billing_company` varchar(255) DEFAULT NULL, CHANGE `shipping_street_address_2` `origin_shipping_street_address_2` varchar(255) DEFAULT NULL, CHANGE `billing_street_address_2` `origin_billing_street_address_2` varchar(255) DEFAULT NULL, CHANGE `shipping_country` `origin_shipping_country` varchar(255) DEFAULT NULL, CHANGE `billing_country` `origin_billing_country` varchar(255) DEFAULT NULL, CHANGE `shipping_state` `origin_shipping_state` varchar(255) DEFAULT NULL, CHANGE `billing_state` `origin_billing_state` varchar(255) DEFAULT NULL
```

通过上面修改，整个migration执行时间降到了十分钟以内。
