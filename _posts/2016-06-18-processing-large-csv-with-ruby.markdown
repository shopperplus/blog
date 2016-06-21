---
layout: post
title: "使用 Ruby 处理大型 CSV 文件"
author: XguoX
---
处理大文件是一项非常耗内存的操作, 有时候甚至会跑光服务器上的物理内存和虚拟内存. 下面来看看使用 Ruby 来处理大型 CSV 文件的几种方式, 同时测试一下这几种方式的内存消耗以及性能.

### 准备测试用的 CSV 数据文件.
在开始之前, 让我们来准备一个拥有一百万行的 CSV 文件 **data.csv**(大约 75mb)用于测试.

```ruby

require 'csv'
require_relative './helpers'

headers = ['id', 'name', 'email', 'city', 'street', 'country']

name    = "Pink Panther"
email   = "pink.panther@example.com"
city    = "Pink City"
street  = "Pink Road"
country = "Pink Country"

print_memory_usage do
  print_time_spent do
    CSV.open('data.csv', 'w', write_headers: true, headers: headers) do |csv|
      1_000_000.times do |i|
        csv << [i, name, email, city, street, country]
      end
    end
  end
end
```

### 内存及时间的消耗

上面这个脚本需要引用到 `helpers.rb` 脚本, `helpers.rb` 定义了两个 helper 方法来测量并打印出内存以及时间的消耗情况.

```ruby

require 'benchmark'

def print_memory_usage
  memory_before = `ps -o rss= -p #{Process.pid}`.to_i
  yield
  memory_after = `ps -o rss= -p #{Process.pid}`.to_i

  puts "Memory: #{((memory_after - memory_before) / 1024.0).round(2)} MB"
end

def print_time_spent
  time = Benchmark.realtime do
    yield
  end

  puts "Time: #{time.round(2)}"
end
```

执行并生成 CSV 文件:

```ruby

$ ruby generate_csv.rb
Time: 7.14
Memory: 4.79 MB
```

不同的机器输出结果也不尽相同, 不过, 幸运的是, 得益于 garbage collector (GC) 回收已使用过的内存, 这个 Ruby 进程耗掉的内存(4.79MB)并不是很夸张. 同时生成的数据文件如图为 74MB.

![](http://ww2.sinaimg.cn/large/62fdd4d5jw1f2m3u6s1twj20ze038gm6.jpg)

### CSV.read
```ruby

# parse1.rb
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    csv = CSV.read('data.csv', headers: true)
    sum = 0

    csv.each do |row|
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end

```
执行结果:

```ruby

$ ruby parse1.rb

Sum: 499999500000
Time: 21.3
Memory: 1277.07 MB
```

惊人的超过 1GB 内存消耗啊.

`CSV.read` 源码

```ruby

 # File csv.rb, line 1750
def read
  rows = to_a
  if @use_headers
    Table.new(rows)
  else
    rows
  end
end

```

### CSV.parse

```ruby

# parse2.rb
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    content = File.read('data.csv')
    csv = CSV.parse(content, headers: true)
    sum = 0

    csv.each do |row|
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end
```

执行结果:

```ruby

$ ruby parse2.rb
Sum: 499999500000
Time: 21.88
Memory: 1362.89 MB
```

可以看到, 内存消耗的比刚刚第一个脚本还略多, 差距大概就是刚好 CSV 文件大小.

`CSV.parse` 源码

```ruby

# File csv.rb, line 1293
def self.parse(*args, &block)
  csv = new(*args)
  if block.nil?  # slurp contents, if no block is given
    begin
      csv.read
    ensure
      csv.close
    end
  else           # or pass each row to a provided block
    csv.each(&block)
  end
end
```

### CSV.new

```ruby

# parse3.rb
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    content = File.read('data.csv')
    csv = CSV.new(content, headers: true)
    sum = 0

    while row = csv.shift
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end
```

执行结果:

```ruby

$ ruby parse3.rb
Sum: 499999500000
Time: 16.89
Memory: 76.72 MB
```

这次结果可以看到, 因为只是在内存中加载了整个文件内容, 所以内存的消耗大概就是文件的大小(74MB), 而处理时间也快了不少. 当我们只是想逐行逐行的操作而不是读取要一次过读取一整个文件时候, 这种方法非常奏效.

### 通过 IO 对象一行一行解析
虽然有了很大的进步, 但是, 使用 IO 文件对象还可以做得更好.

```ruby

# parse4.rb
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    File.open('data.csv', 'r') do |file|
      csv = CSV.new(file, headers: true)
      sum = 0

      while row = csv.shift
        sum += row['id'].to_i
      end

      puts "Sum: #{sum}"
    end
  end
end
```

执行结果:

```ruby

$ ruby parse4.rb
Sum: 499999500000
Time: 13.78
Memory: 2.64 MB
```

仅仅用了 2.64MB, 速度也稍稍又快了一些.

### CSV.foreach

```ruby

# parse5.rb
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    sum = 0

    CSV.foreach('data.csv', headers: true) do |row|
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end
```

执行结果跟上一个差不多:

```ruby

$ ruby parse5.rb
Sum: 499999500000
Time: 15.54
Memory: 2.62 MB
```

`CSV.foreach` 源码

```ruby

# File csv.rb, line 1118
def self.foreach(path, options = Hash.new, &block)
  return to_enum(__method__, path, options) unless block
  open(path, options) do |csv|
    csv.each(&block)
  end
end
```

Source Link 来自: [http://dalibornasevic.com/posts/68-processing-large-csv-files-with-ruby](http://dalibornasevic.com/posts/68-processing-large-csv-files-with-ruby)
