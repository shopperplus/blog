---
layout: post
title: "ActiveRecord Object Instantiate and Clone"
author: Hooopo
---

## initialize

最常用的就是使用`new`方法初始化一个`AR`对象，比如 `User.new(:name => "user name")`. `new` 方法通过 `initialize` 方法构造出一个新对象。大家几乎天天在用这个方法，不多说了。

```ruby
# https://github.com/rails/rails/blob/f916aa247bddba0c58c50822886bc29e8556df76/activerecord/lib/active_record/core.rb#L276-L297

def initialize(attributes = nil, options = {})
  @attributes = self.class._default_attributes.dup

  init_internals
  initialize_internals_callback

  self.class.define_attribute_methods
  # +options+ argument is only needed to make protected_attributes gem easier to hook.
  # Remove it when we drop support to this gem.
  init_attributes(attributes, options) if attributes

  yield self if block_given?
  _run_initialize_callbacks
end
```

## allocate + init_with

`new` 方法构造出来的对象在你未save之前都是 `new_record`，即：

```ruby
user.new_record? #=> true
```

在一些场景，我们已经有了对象`id`和Raw Attributes，我们想要初始化一个AR对象。最经典的场景就是从缓存读出Raw Attributes之后。这时，使用 `new` 方法就不合适。还好，Rails 给我们提供了一个后门：

```ruby
# https://github.com/rails/rails/blob/f916aa247bddba0c58c50822886bc29e8556df76/activerecord/lib/active_record/core.rb#L309-L322
user = User.allocate
user.init_with("attributes" => attributes, "new_record" => false)
```

我们知道，一个对象由类定义和属性组成，已知类定义之后，我们只要把属性填充给实例对象就完成了初始化过程。其实，这也绕过了使用`new`方法来构造对象所需的内部复杂工序。

## instancate

除此之外，如果你的attributes数据是从`DB`获取来的，你可以使用更高级的`instancate`方法，它对来源数据进行一次类型转换，然后再调用上述的 `allocate` + `init_with`。`find_by_sql` 和`STI`就是直接依赖 `instancate` 来初始化AR 对象。

```ruby
# https://github.com/rails/rails/blob/298ec6de55e3adb7d012ba6a9db8b4dd5fd95779/activerecord/lib/active_record/persistence.rb#L56-L70

def instantiate(attributes, column_types = {})
  klass = discriminate_class_for_record(attributes)
  attributes = klass.attributes_builder.build_from_database(attributes, column_types)
  klass.allocate.init_with('attributes' => attributes, 'new_record' => false)
end

```

## initialize_dup

如果我们已经有了一个存在的AR记录，想再初始化一个的话，使用`clone`或`dup`也是可行的。关于`clone`和`dup`区别，我觉得下面这段代码描述是最清晰的：

```ruby
class Object
  def clone
    clone = self.class.allocate

    clone.copy_instance_variables(self)
    clone.copy_singleton_class(self)

    clone.initialize_clone(self)
    clone.freeze if frozen?

    clone
  end

  def dup
    dup = self.class.allocate
    dup.copy_instance_variables(self)
    dup.initialize_dup(self)
    dup
  end

  def initialize_clone(other)
    initialize_copy(other)
  end

  def initialize_dup(other)
    initialize_copy(other)
  end

  def initialize_copy(other)
    # some internal stuff (don't worry)
  end
end
```

当然，`clone`和`dup`的区别这里不是重点，想说一下 `initialize_dup` 钩子。 `initialize_dup` 是AR内置的，我们在调用 `user.dup` 时，底层调用的其实是它。注意，`user.dup` 放回对象把id重置为空了，就是说这是一个 shallow copy.

```ruby
# https://github.com/rails/rails/blob/f916aa247bddba0c58c50822886bc29e8556df76/activerecord/lib/active_record/core.rb#L325-L364

def initialize_dup(other) # :nodoc:
  @attributes = @attributes.dup
  @attributes.reset(self.class.primary_key)

  _run_initialize_callbacks

  @aggregation_cache = {}
  @association_cache = {}

  @new_record  = true
  @destroyed   = false

  super
end
```

## Question

那么问题来了:

* 为什么AR要自己实现一个 `initialize_dup`，直接dup不行么？
* 我们在缓存的时候是直接把 AR 对象塞进去（即：`Rails.cache.write user.id, user`），还是自己序列化反序列化？

## Links

* http://thekaiway.com/2013/07/26/assemble-ar-object/
* http://thekaiway.com/2013/09/08/read-write-activerecord-attribute/
* http://www.jonathanleighton.com/articles/2011/initialize_clone-initialize_dup-and-initialize_copy-in-ruby/
