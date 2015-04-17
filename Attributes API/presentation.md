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

# Identifying Holes in Your API

^ Before you can design a great API, you need to find the API that you're missing. In any large or legacy code base (and make no mistake, Rails is just a large legacy code base), you can find plenty of concepts that exist in your domain, but don't have an explicit name. Common signs of this can include structurally similar or duplicated code, methods with the same prefix, or multiple classes/modules frequently overriding the same method.

^ Here's some examples of code that you might find in Rails 4.1. The common concept will quickly become apparent.

---

# Time Zone Conversion

```ruby
# lib/active_record/attribute_methods/time_zone_conversion.rb

if create_time_zone_conversion_attribute?(attr_name, columns_hash[attr_name])
  method_body, line = <<-EOV, __LINE__ + 1
    def #{attr_name}=(time)
      time_with_zone = convert_value_to_time_zone("#{attr_name}", time)
      previous_time = attribute_changed?("#{attr_name}") ? changed_attributes["#{attr_name}"] : read_attribute(:#{attr_name})
      write_attribute(:#{attr_name}, time)
      #{attr_name}_will_change! if previous_time != time_with_zone
      @attributes_cache["#{attr_name}"] = time_with_zone
    end
  EOV
  generated_attribute_methods.module_eval(method_body, __FILE__, line)
end
```

^ Don't worry too much about the specifics of the code. This code will override the attribute writer for an attribute, so that times are converted to the current time zone. The implementation jumps through some hoops to maintain the "before type cast" version of the attribute, and has to redefine dirty tracking in this method as well.

---

# Time Zone Conversion

- Overriding an attribute reader/writer
- Duplicates code from other parts of Active Record
- Jumps through significant hoops for a relatively minor behavior change

---

# Time Zone Conversion

- Overriding an attribute reader/writer :white_check_mark:
- Duplicates code from other parts of Active Record
- Jumps through significant hoops for a relatively minor behavior change

---

# Time Zone Conversion

- Overriding an attribute reader/writer :white_check_mark:
- Duplicates code from other parts of Active Record :white_check_mark:
- Jumps through significant hoops for a relatively minor behavior change

---

# Time Zone Conversion

- Overriding an attribute reader/writer :white_check_mark:
- Duplicates code from other parts of Active Record :white_check_mark:
- Jumps through significant hoops for a relatively minor behavior change :white_check_mark:

---

# Time Zone Conversion

- Overriding an attribute reader/writer :white_check_mark:
- Duplicates code from other parts of Active Record :white_check_mark:
- Jumps through significant hoops for a relatively minor behavior change :white_check_mark:
- Introduces large number of subtle bugs :bomb:

^ Another thing to note about this style of code is that it introduces a large amount of subtle bugs, which are hard to detect since the concept is scattered over so many places.

---

# Serialized Attributes

```ruby
# lib/active_record/attribute_methods/serialization.rb

def typecasted_attribute_value(name)
  if self.class.serialized_attributes.include?(name)
    @attributes[name].serialized_value
  else
    super
  end
end
```

^ Here's another example, from the code for the `serialize` macro. In this case, we're just overriding the method that ultimately gets called to perform typecasting, rather than the reader/writer explicitly. Unfortunately it's not nearly this simple. Here's a look at some more of the code from that file

---

```ruby
module ClassMethods # :nodoc:
  def initialize_attributes(attributes, options = {})
    serialized = (options.delete(:serialized) { true }) ? :serialized : :unserialized
    super(attributes, options)

    serialized_attributes.each do |key, coder|
      if attributes.key?(key)
        attributes[key] = Attribute.new(coder, attributes[key], serialized)
      end
    end

    attributes
  end
end

def should_record_timestamps?
  super || (self.record_timestamps && (attributes.keys & self.class.serialized_attributes.keys).present?)
end

def keys_for_partial_write
  super | (attributes.keys & self.class.serialized_attributes.keys)
end
```

^ But wait, there's more!

---

```ruby
def type_cast_attribute_for_write(column, value)
  if column && coder = self.class.serialized_attributes[column.name]
    Attribute.new(coder, value, :unserialized)
  else
    super
  end
end

def raw_type_cast_attribute_for_write(column, value)
  if column && coder = self.class.serialized_attributes[column.name]
    Attribute.new(coder, value, :serialized)
  else
    super
  end
end

def _field_changed?(attr, old, value)
  if self.class.serialized_attributes.include?(attr)
    old != value
  else
    super
  end
end
```

^ And more

---

```ruby
def read_attribute_before_type_cast(attr_name)
  if self.class.serialized_attributes.include?(attr_name)
    super.unserialized_value
  else
    super
  end
end

def attributes_before_type_cast
  super.dup.tap do |attributes|
    self.class.serialized_attributes.each_key do |key|
      if attributes.key?(key)
        attributes[key] = attributes[key].unserialized_value
      end
    end
  end
end

def attributes_for_coder
  attribute_names.each_with_object({}) do |name, attrs|
    attrs[name] = if self.class.serialized_attributes.include?(name)
                    @attributes[name].serialized_value
                  else
                    read_attribute(name)
                  end
  end
end
```

^ And I think you get the picture. I left out several more pages for brevity. This one overrides almost every method in Active Record that contains the word "attribute"

---

# Serialized Attributes

- Overriding an attribute reader/writer
- Duplicates code from other parts of Active Record
- Jumps through significant hoops for a relatively minor behavior change
- Overrides literally everything
- Introduces large number of subtle bugs

---

# Serialized Attributes

- Overriding an attribute reader/writer :x:
- Duplicates code from other parts of Active Record :white_check_mark:
- Jumps through significant hoops for a relatively minor behavior change :white_check_mark:
- Overrides literally everything :white_check_mark:
- Introduces large number of subtle bugs :bomb: :bomb: :bomb: :bomb:

^ In this case we don't ever explicitly override the reader and writer, just the generic read/write attribute methods in Active Record. We see less direct copy/pasting of code, but this file ends up knowing about the entire structure of attribute assignment. And oh my god this thing caused so many bugs.

^ A couple of things I didn't show is that this macro ends up modifying the columns hash, which is something that we'll talk about a lot later.

---

# Enum

```ruby
# def status=(value) self[:status] = statuses[value] end
klass.send(:detect_enum_conflict!, name, "#{name}=")
define_method("#{name}=") { |value|
  if enum_values.has_key?(value) || value.blank?
    self[name] = enum_values[value]
  elsif enum_values.has_value?(value)
    # Assigning a value directly is not a end-user feature, hence it's not documented.
    # This is used internally to make building objects from the generated scopes work
    # as expected, i.e. +Conversation.archived.build.archived?+ should be true.
    self[name] = value
  else
    raise ArgumentError, "'#{value}' is not a valid #{name}"
  end
}

# def status() statuses.key self[:status] end
klass.send(:detect_enum_conflict!, name, name)
define_method(name) { enum_values.key self[name] }

# def status_before_type_cast() statuses.key self[:status] end
klass.send(:detect_enum_conflict!, name, "#{name}_before_type_cast")
define_method("#{name}_before_type_cast") { enum_values.key self[name] }
```

^ Same deal. Overriding various methods which exist at different points in the lifecycle.

---

# Enum

- Overriding an attribute reader/writer :white_check_mark:
- Duplicates code from other parts of Active Record :white_check_mark: :white_check_mark: :white_check_mark:
- Jumps through significant hoops for a relatively minor behavior change :white_check_mark:
- Introduces large number of subtle bugs :bomb: :bomb: :bomb: :bomb:

---

# We've Found a Missing Concept

Typed attributes are found and overridden *all over the place*. If we want to do this so much, maybe others do as well.

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
