# Owning Ownership

---

# Who am I?

- Sean Griffin
- Rails Committer
- Maintainer of Active Record
- Creator of Diesel
- Bikeshed co-host
- @sgrif on Twitter

^ I'd like to start this talk with a very controversial statement.

---

# Ruby, JavaScript, and Python are dynamically typed

^ Hopefully we didn't make anybody do a spit take.

^ The important point I'd like to make about this though, is because Ruby is dynamically typed, it does not give you a way to express types or verify that your code is well typed.

^ But here's the thing...

---

# Your program still has types

^ It does not matter if you're working in a statically typed language or a dynamically typed language. You still have types. You still expect a variable to adhere to a certain interface. You still reason about your program based on the types of your objects.

^ Note: By type I don't mean the specific class of an object. "Thing that responds to `#to_s`" is still a type. When you think about it this way, of course your program has types. If an object didn't have a type, you wouldn't be able to do anything with it.

---

# Your program still has types

^ In fact, when I'm mentoring Turing students and they start to get lost, one of the questions I ask them to get them back on track is "What are the types here? What do you expect? What do you need? How do we get from A to B".

^ Types are a lovely thing, and thinking about them in the back of your mind prevents a lot of bugs.

^ The point to all this is...

---

## Semantics still exist, regardless of whether the language gives you a way to express or enforce them

^ Now let's talk about another type of semantics that Ruby "doesn't have".

---

# Let's talk about ownership

^ Ownership is something that was explored a bit in C++ in 2011, but was truly introduced to the world by a new programming language called Rust.

---

# Let's talk about ownership

![fit](rust.png)

^ All values have an owner. You either own an object, or you're borrowing it.

---

## What does it mean to own an object?

^ In the simplest sense, owning an object means that you have the right to destroy it.

^ Now you might be asking...

---

## What does it mean to destroy an object?

^ It actually doesn't matter what specifically happens, other than the rule that after an object has been destroyed, you aren't allowed to use it anymore.

^ You might ask...

---

## What's an example of an object that can no longer be used at some point?

^ You might also ask...

---

## When will you stop answering questions with more questions?

---

# Code time!

^ Let's look at some examples. We'll show it in Rust, since it has ways to let us express ownership. We'll work with a `File` object, since destruction has very clear semantics in Ruby. You can't call any methods on `File` once you've called `close`.

---

```rust
fn do_stuff_with_file(f: File) {
    // ...
    // f will be destroyed when this function returns
}

fn main() {
    let file = File::open("hello.txt");
    do_stuff_with_file(file);
    // ownership of file is moved into `do_stuff_with_file`

    do_more_stuff_with_file(file);
    // ERROR: Use of moved value `file`
}
```

^ It wouldn't be terribly useful if we could only ever use a variable once. This is why we are able to "borrow" objects.

---

```rust
fn do_stuff_with_file(f: &File) {
    //                   ^~~~~ Note the ampersand.
    // ...
    // f is borrowed, and therefore not destroyed
}

fn main() {
    let file = File::open("hello.txt");
    do_stuff_with_file(&file);
    // do_stuff_with_file borrows `file`, ownership is not moved.

    do_more_stuff_with_file(file);
    // This works fine now.
}
```

^ Explain what a reference is. Explain it in terms of pointers for those who have worked in a c-like language. For those who haven't, I'm jealous. Pray that you never have to.

^ Briefly cover mutability as well

---

# Objects can own things as well as functions

^ This applies to fields on objects as well. If you store something in an instance variable, you almost always own it.

^ You'll also tend to note that "god objects" in your system tend to own a lot of stuff. Because of this, I'd like to propose that we rename "god object" to...

---

![fit](trump.jpg)

---

![fit](trump.jpg)

## (I don't know if you've heard, but he's very rich.)

^ Make Ruby great again

---

# Caveat: Some types don't have owners

^ These are usually immutable values where it doesn't matter how many copies of it you make (we actually call these types `Copy` in Rust). Things like numbers and booleans fall into this category. Nobody owns `3`.

---

# Ownership recap

- Owning an object means you have the right to destroy it
- Values have one owner
- If you need an object temporarily, you borrow it
- Objects can be borrowed either mutably or immutably

^ So now let's go back to my original point...

---

# Ruby, JavaScript, and Python are dynamically typed

---

## Your code still has to reason about types

---

## These languages do not have an explicit concept of ownership

---

## Your code still has to reason about ownership

---

## Semantics still exist, regardless of whether the language gives you a way to express or enforce them

---

## Enough of this Rust nonsense. Let's look at some examples.

---

# Ownership in Active Record

```ruby
class User < ActiveRecord::Base
  attribute :name, Type::String.new
end

name = "Sean"
user = User.new(name: name)
# Active Record has taken ownership of `name`

name << " Griffin"
# ^-- This would cause bugs!
```

---

# Ownership in Active Record

```ruby
class User < ActiveRecord::Base
  attribute :name, Type::String.new
end

name = "Sean"
user = User.new(name: name)
user.name << " Griffin"
```

---

# Ownership in Active Record

```ruby
class User < ActiveRecord::Base
  attribute :name, Type::String.new
end

user = User.new(name: "Sean")
user.name << " Griffin"
```

---

# Let's look at a Rails bug!

---

![fit](tendertroll.jpg)

---

![fit](tendertroll.jpg)

# (I kid, I kid)

---

```ruby
# Returns a hash with the \parameters used to form the \path of the request.
# Returned hash keys are strings:
#
#   {'action' => 'my_action', 'controller' => 'my_controller'}
def path_parameters
  @env[PARAMETERS_KEY] ||= {}
end
```

---

```patch
index 277207a..4defb7f 100644
--- a/actionpack/lib/action_dispatch/http/parameters.rb
+++ b/actionpack/lib/action_dispatch/http/parameters.rb
@@ -32,7 +29,7 @@ module ActionDispatch
       #
       #   {'action' => 'my_action', 'controller' => 'my_controller'}
       def path_parameters
-        @env[PARAMETERS_KEY] ||= {}
+        get_header(PARAMETERS_KEY) || {}
       end
 
     private
```

---

> use get / set header to avoid depending on the `env` ivar

---

```ruby





controller.request.path_parameters[:controller] =
  _controller_path
```

---

#### `fn path_parameters(&mut self) -> &mut HashMap<String, String>;`

^ One thing to note: That signature implies that the return value is owned by
`self`. Since we're returning a reference, not an owned value. `self` is the
only argument, so it must be owned by self. We can't pull a reference out of
thin air.

---

```ruby
# Returns a hash with the \parameters used to form the \path of the request.
# Returned hash keys are strings:
#
#   {'action' => 'my_action', 'controller' => 'my_controller'}
def path_parameters
  get_header(PARAMETERS_KEY) || {}
end
```

---

# Who owns the empty hash?

---

## What do we expect the caller to do with the result of this method?

---

## What do we expect the caller to do with the result of this method?

- Are they allowed to mutate it?

---

## What do we expect the caller to do with the result of this method?

- Are they allowed to mutate it?
- Are they allowed to keep a strong reference to it?

---

## What do we expect the caller to do with the result of this method?

- Are they allowed to mutate it?
- Are they allowed to keep a strong reference to it?
- Are we allowed to mutate it out from under them?

---

## What do we expect the caller to do with the result of this method?

- Are they allowed to mutate it?
- Are they allowed to keep a strong reference to it?
- Are we allowed to mutate it out from under them?
- Are we allowed to replace the hash on ourselves? Does the caller expect this
  hash to always be associated with `self`?

---

## What was the intention of the change?

---

> call the `path_parameters=` setter rather than rely on mutations

---

#### `fn path_parameters(&mut self) -> &mut HashMap<String, String>;`

---

#### `fn path_parameters(&self) -> &HashMap<String, String>;`

---

## How can we make our ownership contract more explicit?

---

```ruby




def changed_attributes
  @changed_attributes ||=
    ActiveSupport::HashWithIndifferentAccess.new
end
```

---

```ruby





def changed_attributes
  super.reverse_merge(mutation_tracker.changed_values)
end
```

---

## The return value isn't owned by Active Record, your mutations won't be seen by it

---

```patch
index 0bcfa5f..d40f352 100644
--- a/activerecord/lib/active_record/attribute_methods/dirty.rb
+++ b/activerecord/lib/active_record/attribute_methods/dirty.rb
@@ -80,7 +80,7 @@ module ActiveRecord
       def changed_attributes
         super.reverse_merge(mutation_tracker.changed_values)
+          .freeze
       end
```

---

## Our contract becomes explicit

---

# The bug was eventually resolved

```patch
index c9df787..cca7376 100644
--- a/actionpack/lib/action_dispatch/http/parameters.rb
+++ b/actionpack/lib/action_dispatch/http/parameters.rb
@@ -43,7 +43,7 @@ module ActionDispatch
       #
       #   {'action' => 'my_action', 'controller' => 'my_controller'}
       def path_parameters
-        get_header(PARAMETERS_KEY) || {}
+        get_header(PARAMETERS_KEY) || set_header(PARAMETERS_KEY, {})
       end
 
       private
```

---

> Remember the parameter hash we return. Callers expect to be able to manipulate it.
-- The commit message resolving the bug

^ Or to put it another way, callers expect to borrow a value that we own.

---

## Let's start thinking about ownership in our code

---

![fit](diesel.png)

---

![fit](diesel.png)

# http://diesel.rs

---

# Thank you!

- Email: sean@seantheprogrammer.com
- Twitter: @sgrif
- Github: @sgrif
- http://diesel.rs
- http://bikeshed.fm
