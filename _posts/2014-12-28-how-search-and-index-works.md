---
layout: post
title: "How search and index works?"
author: Hooopo
---

这篇文章主要目的是简单介绍搜索引擎的检索过程，并通过简单的Ruby代码实现。当然，由于本人能力有限，对搜索也是一知半解，注定话题不会太深，也不会太广...这不是 Solr 配置详解这种过于务实的话题，也不是某个搜索排序算法的深入研究的高深话题。

## Tokenize (分词)

Tokenize 的过程是把Query或待索引的文本拆成最小的单元，也就是分词。包括以下过程：

* Preprocess text
  * Synonym substitution
  * Remove illegal expressions
  * Remove stopwords.
* Splitting and preparations for tokenizing
  * Split the text into words.
  * Truncate the amount of tokens.
* Creating tokens / strings
  * Downcase each token.
  * Stemming each token.
  *

**Synonym Rings** (同义词环)
同义词环（如图），把一组定义为等价关系的词汇连接起来，以提供搜索之用。事实上，这些词通常不是真正的同义词。例如，你在检查搜索日志，并和用户对话之后，你将发现不同的人在寻找同样的东西时，会输入不同的术语。某人要找 food processor（食物搅拌机），可能输入 `blender` (搅拌机)，或下图中任何一个术语（或者常见的错误拼法）。不同类型网站，有着各自领域的同义词。

![](https://camo.githubusercontent.com/fd218a73442258bb08528d3f02e7e47c0e57c110/687474703a2f2f7365616e636f6e6e6f6c6c792e63612f7765622f303539363532373334392f696d616765732f696e666f335f303930322e6a7067)

**Stopwords** （停词）正如字面意思，停词。有些高频词，例如“a”，对用户查询的意义不大，同时在索引中又占据比较多的资源，最重要的是这些词还会影响匹配文档的相关度，所以，干脆在分词的时候就把它去掉。


**Automatic Stemming** (自动词干)功能，是搜索引擎的一个实用功能，把一个术语扩展，包含其他共享相同词干的术语。如果词干搜索机制很强，可能会把 `computer` 这个搜索术语视为和 `computers`、`computation`、`computing` 甚至 `comput` 这些术语共享相同的词干 `comput`。

![](https://camo.githubusercontent.com/4e33f7f06e36361d59382ba60eb926564396b81d/687474703a2f2f73332e616d617a6f6e6177732e636f6d2f617765736f6d655f73637265656e73686f742f363839303733343f4157534163636573734b657949643d305237464d57374158525643594d41505450523226457870697265733d31343139373632303333265369676e61747572653d7547716759664e66736a636f6d396e4e69465766787174394c3130253344)

其他过程很容易理解了，整个Tokenize用Ruby代码的简单表示：

```ruby
def tokenize(text)
  text.gsub(/&/, "and").                      # Synonym substitution
       gsub(/[^\w\s]+/, "").                  # Remove illegal expressions
       gsub(/\b(a|an|is)\b/i, "").            # Remove stopwords
       split(/\s+/).map do |token|            # Split the text into words
         token.downcase.sub(/(.*?)s\Z/, '\1') # Downcase each token
       end
end

tokenize("How search & index works?") #=> ["how", "search", "and", "index", "work"]
```

其中Stemming过程过于简陋，但相关算法其实已经非常成熟了，比如Porter、Hunspell等算法。其中Ruby实现有：

* http://locknet.ro/archive/2009-10-29-ann-ruby-stemmer.html
* https://github.com/aurelian/ruby-stemmer
* https://github.com/romanbsd/fast-stemmer

## Inverted Index (倒排索引)

倒排索引（英语：Inverted index），也常被称为反向索引、置入档案或反向档案，是一种索引方法，被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。它是文档检索系统中最常用的数据结构。

以英文为例，下面是要被索引的文本：

```text
T_0="it is what it is"
T_1="what is it"
T_2="it is a banana"
```

**正向索引**：

```ruby
docs = {
  1 => "it is what it is",
  2 => "what is it",
  3 => "it is a banana"
}
```

**倒排索引**：

```text
 "a":      {2}
 "banana": {2}
 "is":     {0, 1, 2}
 "it":     {0, 1, 2}
 "what":   {0, 1}
 ```

检索的条件"what", "is" 和 "it" 将对应这个集合：`{0,1}` & `{0,1,2}` & `{0,1,2}` = `{0,1}`。即：0号和1号文档包含 what is it这三个关键词。

接着上面的Tokenize，用Ruby简单实现倒排索引过程， 其中对Query 分词和对文档分词使用的是同一方式（有时候两者会有微小差异）：

```ruby
require 'set'
require 'pry'

docs = {
  1 => "it is what it is",
  2 => "what is it",
  3 => "it is a banana"
}

def tokenize(text)
  text.gsub(/&/, "and").
       gsub(/[^\w\s]+/, "").
       gsub(/\b(a|an|is)\b/i, "").
       split(/\s+/).map do |token|
         token.downcase.sub(/(.*?)s\Z/, '\1')
       end
end

index = {}

docs.each do |id, doc|
  tokenize(doc).each do |word|
    if index.has_key?(word)
      index[word] << id
    else
      index[word] = Set.new([id])
    end
  end
end

def search(text, index)
  tokenize(text).map{ |word| index[word] || Set.new }.reduce(:&)
end

index_store = init_index_store(Docs)   => {"it"=>#<Set: {1, 2, 3}>, "what"=>#<Set: {1, 2}>, "banana"=>#<Set: {3}>}
search("what is it?", index_store)     => #<Set: {1, 2}>
```

至此，一个最简单是search engine就实现了，当然，我们还没涉及到排序，搜索过程使用的是最简单的 集合布尔模型。下面会涉及 `TF-IDF` 相关度排序内容。

## Sort By  Relevance (按相关度排序)

TODO
