---
layout: post
title: "CRC32 Usage In MySQL"
author: Hooopo
---

CRC全称为Cyclic Redundancy Check，又叫循环冗余校验。CRC32是CRC算法的一种，常用于校验网络上传输的文件。

## How to Use CRC32 In MySQL & Ruby

```text
mysql> SELECT CRC32('hello');
+----------------+
| CRC32('hello') |
+----------------+
|      907060870 |
+----------------+
```

```ruby
require 'zlib'
Zlib::crc32('hello') # => 907060870
```

## CRC32 Vs MD5

这里我们不谈校验，只说MySQL里如何利用CRC32来加快查询。首先我们得知道CRC32的基本特征：

1. CRC32函数返回值的范围是0-4294967296（2的32次方减1)
2. 相比MD5，CRC32函数很容易碰撞

## CRC32的使用场景

由上述两个基本特性可知，CRC32在MySQL里只需要用bigint存储，而MD5需要varchar来存储。但是CRC32很容易碰撞，这适合做索引么？

场景：我们在做一个爬虫，对于一个URL，先去数据库里查询是否存在，如果不存在，插入到数据库中。大家都知道这种类型应用表会增长非常迅速，如果简单的

```text
SELECT * FROM urls WHERE url = 'http://wwww.shopperplus.com';`
```

会每次全表扫描，效率非常低。如果在url列上面加索引会快一些，但由于url是varchar类型，字段本身的存储空间和索引占用的存储空间都比较大。

如果加一个crc32_url列，并且只在这个列上加索引，索引空间就会小很多。

```
SELECT * FROM urls WHERE crc_url = 907060870;
```
上面这种做法一定不可以，因为前面已经提到crc32函数会产生碰撞，也就是说值为907060870的不止有`hello`。一个小技巧是只使用crc32列来过滤：

```
SELECT * FROM urls WHERE crc_url = 907060870 AND url = 'hello';
```
来看一下有碰撞时，索引利用情况：

```text
explain SELECT * FROM urls WHERE crc32_url = 907060870 AND url = 'hello'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: urls
         type: ref
          key: index_urls_on_crc32_url
      key_len: 4
          ref: const
         rows: 2
        Extra: Using where
```

这样一来，大部分查询还是只需要扫描一行就获得结果。对于少部分碰撞的记录，只需要多扫描几行也可以正确获得结果。

url的场景从varchar到bigint的优化其实效果不是特别明显。另一个例子是文本，假如我们有一个text类型的字段（文章内容、评论、微博之类），每次插入之前要判断一下这个内容是否在数据库里存在了。如果使用crc32的技巧，改善的空间还是蛮大的。