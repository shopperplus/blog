---
layout: post
title: "Trie 树的应用之敏感词过滤"
author: Hooopo
---

![](https://gp1.wac.edgecastcdn.net/806614/photos/photos.500px.net/65000163/55dcc47e193e5540faaa5d97a70f06206f12cbbc/5.jpg)

## Trie

trie，又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

![](http://upload.wikimedia.org/wikipedia/commons/thumb/b/be/Trie_example.svg/320px-Trie_example.svg.png)

上面的介绍和例子来自维基百科，描述一个保存了8个键的trie结构，"A", "to", "tea", "ted", "ten", "i", "in", and "inn". 图示中标注出完整的单词，只是为了演示trie的原理。

## 敏感词过滤

敏感词过滤或关键词提取是Trie树的一个典型应用。拿待过滤词ABCDEF举例，

* ABCDEF
* BCDEF
* CDEF
* DEF
* EF
* F

依次用上面的子串在Trie里walk，看能否walk到叶子节点，如果能，说明包含关键词，并提取出来。

```ruby
require 'triez'

words = %w[大叶榕 细叶榕 高山榕 木棉 洋紫荆 樟树 芒果 蝴蝶果 人面子]

trie = Triez.new

words.each { |word| trie[word] = 1 }

post_text = "行道树芒果引市民争相采摘。"

results = Hash.new(0)

(0..post_text.size - 1).each do |i|
  trie.walk(post_text[i..-1]).each do |word_in_trie, _|
    results[word_in_trie] = results[word_in_trie] + 1
  end
end

puts results #=> {"芒果" => 1}
```

## 去噪

用户可以通过在输入内容中加入空格、特殊符号来干扰敏感词过滤工作。比如芒果可以写成 `芒 果`、`芒**果`、`芒%果`。
解决办法是在过滤之前去噪。还是上面的代码，加上去噪处理：

```ruby
require 'triez'

words = %w[大叶榕 细叶榕 高山榕 木棉 洋紫荆 樟树 芒果 蝴蝶果 人面子]

trie = Triez.new

words.each { |word| trie[word] = 1 }

post_text = "行道树芒&&果引市民争相采摘。"

results = Hash.new(0)

post_text = post_text.gsub(/[\s&%$@*]+/, "")

(0..post_text.size - 1).each do |i|
  trie.walk(post_text[i..-1]).each do |word_in_trie, _|
    results[word_in_trie] = results[word_in_trie] + 1
  end
end
```