# Designing a Great Ruby API

---

- Sean Griffin
- Software Developer at thoughtbot
- Rails committer
- Bikeshed co-host

![inline](thoughtbot.png)

^ **Brief** intro about yourself

^ Today we're going to talk about API design. I'd like to take a look at a new API that is being introduced in Rails 5 as an example. It's called the Attributes API, and it allows you to hook into the type casting system in Active Record. We're going to talk about the process that went into developing it.

---

# Type Casting

^ Type casting is the process of explicitly converting a value from one type to another. Here's an example.

---

# Type Casting

```ruby
x = "1"
x.class # => String

x.to_i # => 1
x.to_i.class # => Fixnum
```

^ Details about code example

^ In Active Record, we do something called type coercion. Type coercion is the same process, but done automatically instead of explicitly.

---

# Type Coercion

```ruby
user = User.new
user.age = "30" # => "30"

user.age # => 30
user.age.class # => Fixnum
```

^ Details about code example

^ You might be wondering why we coerce values in the first place. Active Record was designed to work easily with web forms. Form input will always be a string, and having to cast manually would be cumbersome. This is especially true for more complicated types like dates. The type system has grown more complex than just handling strings, but the path it's taken can be traced back to that original constraint.

^ In Rails 4 and earlier, the only way to have a coerced attribute is if it's backed by a database column. We wanted to be able to hook into this behavior, and be able to modify it and use it independently of the database. This is what it looks like in the most simple case.

---

```ruby
class Product < ActiveRecord::Base
  attribute :name, Type::String.new
  attribute :price, Type::Integer.new
end
```

---
