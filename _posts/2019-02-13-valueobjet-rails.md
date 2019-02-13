---
layout: post
title: valueobject-in-rails
date:  2019-02-13 17:12:49 +0800
categories: programming
tags: [programming, DDD, Rails, Ruby]
---
最近在看领域驱动设计有关的书（Doman Driven Design，简称DDD），它把对象分成两类，一类是Entity，一类是Value Object，关于Value Object的好处以及DDD的具体的细节这里不多讲，有兴趣的可以看一下有关的书和文章。

看到Value Object的时候，随手搜了一下Value Object 在 Rails 中的应用，发现以前一直没用过的 `composed_of`方法。它是用来在 ActiveRecord 里面方便的实现Value Object的，Rails 的有关文档讲的非常详细，不过在实际项目中用的人好像不是太多，这是比较可惜的。

这篇文章会讲一下如何使用普通的accessor（不推荐）以及composed_of(推荐)分别实现Value Object的继承。

## Address Value Object 的定义
```ruby
class Address
  attr_reader :street, :city

  def initialize(street, city)
    @street, @city = street, city
  end

  def ==(other_address)
    city == other_address.city && street == other_address.street
  end
end
```

Value Object 就是普通的Ruby Object，如果逻辑（特别是业务有关的逻辑）复杂的时候，Value Object的好处就更能体现。

## 前期准备
一个顾客有一个送货地址，地址由城市(city)和街道(street)组成，和上面的 `Address`保持一致

```ruby
  create_table "customers", force: :cascade do |t|
    t.string "name"
    t.string "shipping_street"
    t.string "shipping_city"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end
```

## 普通的 accessor 方法集成 Value Object
```ruby
class Customer < ApplicationRecord
  def shipping_address
    Address.new(shipping_street, shipping_city)
  end

  def shipping_address=(address)
    self.shipping_address = address.address
    self.shipping_street = address.street
  end
end
```

使用：
```ruby
c = Customer.first
c.shipping_address ##<Address:0x00007fafae091bf8 @street="pudong", @city="shanghai">

ad = Address.new("beijing", "beijing")
c.shipping_address = ad
c.shipping_city #"beijing"
c.shipping_street "beijign"
```

## 使用 ActiveRecord 的 composed_of
Rails 提供了 composed_of 方法，就是为了方便集成 value object：
> Active Record implements aggregation through a macro-like class method called composed_of for representing attributes as value objects.

数据库 schema 里定义相应的column，使用的时候使用 value object 作为属性。

用法非常简单：
```ruby
class Customer < ApplicationRecord
  composed_of :shipping_address, class_name: "Address", mapping: [ %w(shipping_street street), %w(shipping_city city) ]
end
```

注意 mapping 表示 Customer 的 shipping_street 对应 Address 的 street 属性，city类似，mapping 的顺序是调用 Address 构造方法的传参顺序。

定义好后就可以使用了:
```ruby
c = Customer.first
c.shipping_address ##<Address:0x00007fafae141170 @street="beijing", @city="beijing">
c.shipping_address = Address.new("shanghai", "shanghai")
c.save #UPDATE "customers" SET "shipping_street" = ?, "shipping_city" = ?, "updated_at" = ? WHERE "customers"."id" = ?  [["shipping_street", "shanghai"], ["shipping_city", "shanghai"], ["updated_at", "2019-02-13 09:03:20.895132"], ["id", 1]]
```

另外还可以直接在通过value object查找:
```ruby
ad = Address.new("pudong", "shanghai")
Customer.where(shipping_address: ad)
# SELECT "customers".* FROM "customers" WHERE "customers"."shipping_street" = ? AND "customers"."shipping_city" = ? LIMIT ?  [["shipping_street", "pudong"], ["shipping_city", "shanghai"], ["LIMIT", 11]]
```
和普通的查询一样方便。

## 总结
Value Object 在领域驱动开发中占有很重要的地位，其实 Rails 也可以很方便的实现Value Object。
