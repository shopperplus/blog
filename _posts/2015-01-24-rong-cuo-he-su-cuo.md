---
layout: post
title: "容错和速错"
author: Hooopo
---

程序出现异常，不想让用户看到，会给一个友好的提示，这种做法一般称作“容错”。另一方面，我们希望自己的程序能够把隐藏错误尽早暴露出来，及时修复，这种思路称之为“速错”。拼写错误、源数据错误、逻辑错误都需要速错。当然，容错和速错也并不总是对立的，分清何时应用哪种策略非常重要。

## Hash#fetch over Hash#[]

当`Hash`的key值是已知的情况，比如状态枚举。优先使用`fetch`可以避免拼写错误带来的意外。举例：

```ruby
class Event < AR
  STATUS = {
    :open => 1,
    :closed => 0
  }
end
Event.update_all(:status => Event::STATUS[:close], "created_at > '2011-1-1'") # => update status to null
Event.update_all(:status => Event::STATUS.fetch(:close), "created_at > '2011-1-1'") # => raise KeyError: key not found: :close
```
上面例子由于错误拼写，把`closed`拼成 `close`，造成了一个`Silent failure`，而才有`fetch`方法就会在拼错时直接抛出异常，避免了之后的错误。

## 使用常量

除此之外，声明常量也可以带来同样效果，本质是给输入加上了拼写检查。

```ruby
class Event < AR
  CLOSED = 0
  OPEN = 1
end
Event.update_all(:status => Event::CLOSE, "created_at > '2011-1-1'")       # => raise NameError: uninitialized constant CLOSE
```

## 善用attr method

经常看到有人问，attr method有什么用，直接使用实例变量不好么，这样的问题。

如果不是对`getter`和 `setter` 有额外的封装，两者是一样的，但用attr method调用和上面两个例子一样，也起到了错误检查的作用。我们知道Ruby的实例变量有一个隐藏特性，实例变量不需要定义就可以使用，不会报错。当然，在启动Ruby加 `-w` 参数是可以warning提示的。

```
class Event
  def initialze(attrs)
    @closed = attrs[:closed] || true
  end

  def xxx
    if @close
      # some code will not run
    end
  end
end
```

如果我们使用attr reader，调用close方法直接就会抛出undefind method异常，让拼写错误尽早现形。

## save! over save

经常看到一段事物代码里用`save`的情况，`save`不会抛异常，这样事物的意义就失去了。显然，这属于不理解事物运作机制的错误用法。在没有事物的代码里，如何选择`save`还是`save!`也是非常困难的。我个人的习惯是，在不需要错误回显的情况，一律使用`save!`和`update_attributes!`这样能够`Fail Fast` 的方法。

## 不要滥用 rescue

如果你的代码里经常见到`begin rescue`,这就是一种`bad small`。当然，调用外部接口时，异常一定要捕获，但有一部分新手会在自己写的一堆应用逻辑外面套上`begin resuce`,并且不明确捕获的错误类型。当你问他，你要捕获哪种错误的时候，他一定回答不出。对于自己代码里的逻辑错误不应该去捕获错误，而是查出错误的来源，从源头上解决问题。即使要捕获，也应该有一个明确的类型：

```ruby
begin
rescue KeyError => e
  ...
end
```
用异常做控制流的做法也不少见，但不是本文讨论的话题..

## 数据源错误

容错的思想带来的一个问题是总想隐藏问题，不是直接去从源头解决。当一个产品数据被误删导致用户订单页面出错，不应该去容错，到处写`order_item.product.try(:name)`，而是应该去恢复被删数据。

同理，数据库出现脏数据，不应该去改变代码的写法，比如，把`save`改成`save(false)`，去跳过验证。应该做的也是清理数据源，否则就需要无穷无尽的“容错”代码。