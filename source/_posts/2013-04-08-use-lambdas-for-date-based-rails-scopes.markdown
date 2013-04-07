---
layout: post
title: "Use Lambdas for Date-based Rails Scopes"
date: 2013-04-08 00:26
comments: true
categories: [rails, scope]
---

A scope allows you to specify an ARel query that can be used as a method call to the model (or association objects).

```ruby
class Item
  scope :delivered, where(delivered: true)
end

Item.delivered.to_sql                     # SELECT "items".* FROM "items"  WHERE "items"."delivered" = 't'
Item.where(price: 2000).delivered.to_sql  # SELECT "items".* FROM "items"  WHERE "items"."price" = 2000 AND "items"."delivered" = 't' 
```

There's a problem if we try using a scope for a relative date, though.

```ruby
class Item
  scope :expired, where("expiry_date < ?", Date.today)
end
```

This code gets evaluated when the server is started, and the *output* of `Date.today` is stored in the scope.

That scope is equivalent to the following:

```ruby
class Item
  def self.expired
    where("expiry_date < ?", "2013-04-01")
  end
end
```

The date is hardcoded in there, and will not be changed until the scope is re-evaluated.  
This typically happens only when the server is restarted.


To get around this problem, use a lambda when defining date (or time) based scopes. This will force the evaluation of the scope each time it is *called*.


```ruby
class Item
  scope :expired, lambda { where("expiry_date < ?", Date.today) }
end
```
