# Rust, and why you should care

---

- Sean Griffin
- Rails committer
- Maintainer of Active Record (I'm sorry)
- Bikeshed co-host
- Developer at Shopify

---

![fit](rust.png)

^ Today I'm going to talk to you about a new programming language called Rust.  For those not familiar, Rust is a language that's been in development at Mozilla for about 10 years. Version 1.0 released in May of this year.

^ (And for any iOS devs, this is the language that the designers of Swift took the most inspiration from).

---

# What does Rust look like?

^ So let's get the first obvious question out of the way. What does a rust program look like? Well, let me answer that for you with some code that is completely representative and relevant to all modern programs. Printing "hello world" to STDOUT.

---

```rust
fn main() {
    println!("Hello, world!");
}
```

^ So this is Rust. As you can see, it's a little bit more verbose than the equivalent in Ruby

---

```ruby
puts "Hello, world!"
```

^ And a little bit less verbose than something like Java.

---

```java
class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, world!");
    }
}
```

^ (I actually had to look up the signature of main because I couldn't remember...)

^ And it's definitely less verbose than the equivalent enterprise Java, which might contain a line like this

---

```java
class AbstractGreetingStrategyFactoryBean
```

^ Anyway let's take a look at what some more realistic Rust code looks like, compared to some other languages. Sam Saffron of Discourse wrote a little gem called `fast_blank` when he noticed that an awful lot of time was spent in `String#blank?`. Here's what the Ruby version looks like (though I've changed it from a method to a function which takes a string so we can do some direct comparisons)

---

# The Ruby Version

```ruby
def blank?(string)
  /\A[[:space:]]*\z/ =~ string
end
```

^ The `blank?` method should return true if all of the characters in the string are whitespace. This isn't just as simple as comparing to the space character.  There are actually 25 characters in unicode which are considered whitespace characters.

^ Like most Ruby code, it's certainly concise. I wouldn't exactly call it expressive, but then again regular expressions rarely are. The `fast_blank` gem improves upon this by rewriting this method as a C extension.

---

# The C Version

```c
static VALUE
rb_str_blank_as(VALUE str)
{
  rb_encoding *enc;
  char *s, *e;

  enc = STR_ENC_GET(str);
  s = RSTRING_PTR(str);
  if (!s || RSTRING_LEN(str) == 0) return Qtrue;

  e = RSTRING_END(str);
  while (s < e) {
    int n;
    unsigned int cc = rb_enc_codepoint_len(s, e, &n, enc);

    switch (cc) {
      case 9:
      case 0xa:
      case 0xb:
      case 0xc:
      case 0xd:
      case 0x20:
      case 0x85:
      case 0xa0:
      case 0x1680:
      case 0x2000:
      case 0x2001:
      case 0x2002:
      case 0x2003:
      case 0x2004:
      case 0x2005:
      case 0x2006:
      case 0x2007:
      case 0x2008:
      case 0x2009:
      case 0x200a:
      case 0x2028:
      case 0x2029:
      case 0x202f:
      case 0x205f:
      case 0x3000:
          /* found */
          break;
      case 0x180e:
          if (ruby_version_before_2_2()) break;
      default:
          return Qfalse;
    }
    s += n;
  }
  return Qtrue;
}
```

^ I'm sure none of you can actually read this, so here's what each chunk is doing here.

---

# Loop through each character

```c
enc = STR_ENC_GET(str);
s = RSTRING_PTR(str);
if (!s || RSTRING_LEN(str) == 0) return Qtrue;

e = RSTRING_END(str);
while (s < e) {
  int n;
  unsigned int cc = rb_enc_codepoint_len(s, e, &n, enc);
```

^ This first block of code is a loop to iterate through each character in the string. Strings in C are just an array of char primitives. A char in C is a single byte, and is analogous to an ASCII character. Since Ruby supports UTF8, we can't just loop over each of the `char`s, we actually need to figure out how many of those bytes make up each character in the string, since they can be up to 4 bytes in unicode.

---

# Check if it's whitespace

```c
switch (cc) {
  case 9:
  case 0xa:
  case 0xb:
  case 0xc:
  case 0xd:
  case 0x20:
  case 0x85:
  case 0xa0:
  case 0x1680:
  case 0x2000:
  case 0x2001:
  case 0x2002:
  case 0x2003:
  case 0x2004:
  case 0x2005:
  case 0x2006:
  case 0x2007:
  case 0x2008:
  case 0x2009:
  case 0x200a:
  case 0x2028:
  case 0x2029:
  case 0x202f:
  case 0x205f:
  case 0x3000:
      /* found */
      break;
  case 0x180e:
      if (ruby_version_before_2_2()) break;
  default:
      return Qfalse;
}
```

^ Next we place the `char` into a giant switch statement. Each of these cases is a character that is considered to be "whitespace". If we get to the `default` case, the character is not whitespace and we return `false` from the whole function.

^ Note differences between case in C and Ruby

---

# Continue the loop

```c
  s += n;
}
return Qtrue;
```

^ Finally we increment our counter by the number of bytes used in that character, and continue the loop. If the loop completes without returning, we return true from the function since all of the characters were whitespace.

^ Now it's completely fine if that code made you feel like this

---

![fit](homer.gif)

^ Because it does that for me, too. The point is that it's very verbose, very explicit about absolutely everything that goes into determining if a character in a string contains whitespace. We lay out all of the details about how to loop through the string. We lay out exactly what whitespace means, and the instructions are set up for the computer so it doesn't do more work than it needs to. And this does pay off.

---

```
================== Test String Length: 136 ==================
Calculating -------------------------------------
          Fast Blank   123.617k i/100ms
          Slow Blank    76.362k i/100ms
-------------------------------------------------
          Fast Blank     11.952M (±11.5%) i/s -     58.594M
          Slow Blank      2.112M (± 6.9%) i/s -     10.538M

Comparison:
          Fast Blank: 12520143.0 i/s
          Slow Blank:  2112055.6 i/s - 5.93x slower
```

^ One thing that this benchmark doesn't show is that the C version also causes 0 allocations, so there will be less GC pressure overall. Now let's look at what the naive Rust version looks like.

---

# The Naive Rust Version

```rust
fn is_blank(string: String) -> bool {
    string.chars().all(|c| c.is_whitespace())
}
```

^ One thing you might notice is that this looks a lot like what you might expect the naive Ruby version to look like. If we pretend that Ruby had a char type (instead of `chars` returning an array of `String`s), and we also pretend that type had a `whitespace?` predicate method, then the code would probably look like this.

---

# The (Hypothetical) Naive Ruby Version

```ruby
fn blank?(string)
  string.chars.all?(&:whitespace?)
end
```

^ Though as you might expect, this version is significantly slower than what activesupport provides. I implemented `whitespace?` using a case statement similar to the C version to compare the two.

---

```ruby
def whitespace?
  case ord
  when 9 then true
  when 0xa then true
  when 0xb then true
  when 0xc then true
  when 0xd then true
  when 0x20 then true
  when 0x85 then true
  when 0xa0 then true
  when 0x1680 then true
  when 0x2000 then true
  when 0x2001 then true
  when 0x2002 then true
  when 0x2003 then true
  when 0x2004 then true
  when 0x2005 then true
  when 0x2006 then true
  when 0x2007 then true
  when 0x2008 then true
  when 0x2009 then true
  when 0x200a then true
  when 0x2028 then true
  when 0x2029 then true
  when 0x202f then true
  when 0x205f then true
  when 0x3000 then true
  when 0x180e then true
  else false
  end
end
```

---

## (The string tested was this)

"\u2000\u2001\u2002\u2003\u2004\u2005\u2006\u2007\u2008\u2009\u200A\u202F\u205F\u3000 \t"

---

```

Calculating -------------------------------------
ActiveSupport blank?    45.597k i/100ms
        Naive blank?    18.596k i/100ms
-------------------------------------------------
ActiveSupport blank?    680.545k (± 7.0%) i/s -      3.420M
        Naive blank?    216.477k (± 4.3%) i/s -      1.097M

Comparison:
ActiveSupport blank?:   680544.7 i/s
        Naive blank?:   216477.2 i/s - 3.14x slower
```

^ So the naive Ruby is way slower. But what if we take our Naive rust function, and set that up as a Ruby extension? How does it compare?

---

```
Calculating -------------------------------------
    Rust blank?     11.043M (± 3.5%) i/s - 54.744M
       C blank?     10.583M (± 8.5%) i/s - 52.464M
    Slow blank?    964.469k (±27.6%) i/s - 4.390M

Comparison:
    Rust blank?:    10948800 i/s
       C blank?:    10492800 i/s - 1.04x slower
    Slow blank?:      878000 i/s - 12.47x slower
```

^ Yes, you're reading that correctly. The Naive Rust version runs 4% faster than the hand rolled, super fine tuned C version. Now I could spend the entire rest of this talk going into exactly why it's faster, but that's not what I'm here to talk about (and this example was shamelessly stolen from a talk Yehuda has been giving where he does go into those details). But what we can establish is this:

---

# The ~~Naive~~ Rust Version

```rust
fn is_blank(string: String) -> bool {
    string.chars().all(|c| c.is_whitespace())
}
```

^ This isn't the naive version at all. This is just the Rust version. You'll be hard pressed to find a way to write this function any faster. And this is ultimately a big part of what Rust is about. Abstraction without costs. Feeling high level, without compromising safety or performance.

---

# So why should I care?

^ Really though, all I've done is shown you this function and said: "look, it looks kinda like Ruby but it runs way faster". That's not exactly a compelling argument to me, and I don't expect it to be one for you, either. So let's look at Rust's key selling features. There's two main things you'll usually hear people say when they talk about Rust.

---

# Memory Safety

^ The biggest selling point that you'll usually hear from Rust is that it's memory safe. This is a big deal for a manually memory managed language (others include C, C++, Objective-C, and others). Roughly speaking, memory safety means protecting you from accidentally accessing memory that you aren't supposed to.  You probably remember a big memory unsafety bug called Heartbleed that happened not too long ago.

---

# ~~Memory Safety~~

## DON'T CARE

^ And I couldn't care less. Coming from the world that I live in, and I assume a lot of you live in, I pretty much only work with garbage collected languages.  This means that memory safety is just something I assume in any language I look at.

---

# Performance

^ The second one you'll hear a lot about Rust is that it's extremely performant.  I've already shown one example, where Rust is even competitive with C. And this is something that's getting better all of the time.

---

# ~~Performance~~

## DON'T CARE

^ And again, I don't know about you, but I just don't care about this that much.  I write Ruby. Ruby is a lot faster than it used to be, but it's not exactly a language that you choose because of its blazing fast performance.

---

# ~~Performance~~

## DON'T CARE
## (Okay maybe a little)

^ Well, maybe I do care a little. And that gets into one of its other key features.

---

# Zero Cost C FFI

^ Rust can call C functions, and can be called from C, for free. You can mark a rust function as "extern 'C'", and when you compile it as a library, things can call you and have no idea that the function wasn't originally written in C. This is also part of why it's important to note that Rust has no runtime. Calling code doesn't have to Rust installed at all.

^ This might not seem like a big deal if you're not a C programmer, but the thing you have to remember is that *everything* knows how to speak C. And since Rust can pretend to be C, this means that Rust has an automatic C FFI to pretty much everything. This means you can even write Ruby extensions in Rust.

^ Next slide has more notes

---

# Zero Cost C FFI

^ And this is really cool. Because it means when your app has that one performance critical thing that really needs to move out of Ruby, you don't need to rearchitect everything into micro-services just so you can rewrite that one little piece in another language. You can just write some Rust code and expose an API to Ruby.

^ And since Rust has no garbage collection, you won't run into a lot of issues with memory contention you might have with other languages. I won't go into too much detail on that, but trust me -- having two garbage collectors running in the same process is a bad idea.

---

# So this is kinda neat but...

^ And I'm hoping there's a few people who are really excited about that aspect of it, because it's a killer feature and is really easy to accomplish in your apps to get some free performance. But like I said, as a Rubyist, performance just isn't a huge selling point to me. So let's start talking about the thing that really excites me about Rust.

---

# The Type System

^ The thing that excites me about Rust is it's really expressive type system.  I know a lot of people have a bad taste about static types after working with Java. But modern type systems don't exist to bind you or just to help the compiler, they actually exist to help you express your ideas. It's the kind of thing where when you're working on a problem, sometimes just writing out the type signatures will show you the solution.

---

# Ownership

^ One of the main concepts you'll see when talking about Rust's type system is the concept of ownership. Basically the compiler keeps track of who owns a value, and makes sure that there's only one owner at a given time. This is important to understand for how it achieves memory safety.

^ Loosely, owning a value means that you have the right to destroy it. At its simplest, destroying something means freeing the memory it was using. This is what the garbage collector does for you when it reaps an object, but garbage collection can be slow, and actually has some limiting factors. Let's look at a simple example.

---

```rust
fn main() {
    let x = "Hello!".to_string();
    // ^--- main "owns" the string x.

} // <--- main returns, and all variables it owns are destroyed. The memory for
//        x is freed.
```

^ explain the code

^ Now let's look at how ownership works with other functions

---

```rust
fn foo(y: String) {
    //    ^--- foo takes ownership of its argument
    // we'd do stuff with y here
}
// when foo returns, y will be destroyed

fn main() {
    let x = "Hello!".to_string();
    // ^--- main owns x

    foo(x);
    // x takes ownership of it's argument, so ownership of
    // x is transfered to foo

    println!("{}", x);
    // ^--- this line will error, because x has been moved. We can no longer use
    // it.
}
```

^ Explain the code

^ Now this wouldn't be very useful if you could ever only use a value once. This is why rust has the concept of borrowing

---

```rust
fn foo(y: &String) {
    //    ^--- foo borrows its argument.
    // we'd do stuff with y here
}

fn main() {
    let x = "Hello!".to_string();
    // ^--- main owns x

    foo(x);
    // x borrows it's argument, main still owns x.

    println!("{}", x);
    // ^--- this line will now work.
}
// main returns, x is destroyed
```

^ Explain the code

^ When you borrow a value, that's called a reference. There's two types of references in Rust. Shared references, and unique references. This is also sometimes referred to as an immutable borrow and a mutable borrow. In order to mutate something in Rust, you have to be the only one who can see that value.

---

![fit](homer.gif)

^ Hopefully that didn't have this affect again. Ownership is one of the concepts in Rust that can seem really daunting at first. But in practice, what it comes down to is this. When you take arguments, you borrow them. Most of the time this is the right thing to do. If the compiler yells at you, then you think about what you're actually trying to accomplish. Usually the change is trivial. If it isn't, the compiler is probably pointing out to you that you might be doing something complex or dangerous (like sharing mutable state across threads).

^ Those are the cases where you do have to spend a bit of time thinking about the right way to accomplish what you want. But if there is a time when you should be doing that, it's when you're doing things that are complex or dangerous.

---

# Sharing and mutability

^ This is one of the first things that I find actually helps me when writing my programs. If you know any Haskell or Clojure programmers, you've probably heard a lot about how immutibility helps you to reason about your programs. And they're right. When anything can mutate any value at any time, you often have to be extremely defensive in how you program.

^ The reasons that Rust does this are actually related to memory safety, not functional purity or anything like that, but it also helps make your programs easier to reason about. You can see at a glance who can and cannot mutate something. And if you own a value, or have a shared reference, you are guaranteed that it won't be mutated from underneath you unless you explicitly allow it.

---

# Ownership isn't unique to Rust

^ And this isn't some new concept in Rust. You already have ownership in your Ruby programs. We just don't usually give it a name. But it's a way of codifying patterns that we use to clean up our applications, like "am I allowed to save this Active Record model". Rust just gives you a way to express it explicitly.

---

# Resource management

^ At its core, ownership is about resource management. Memory just happens to be one kind of resource, but there are others, too. Have you ever forgotten to close a file in Ruby? Or had issues with database connections in Active Record when you tried to set up a threaded web server? Me too. Those issues don't exist in Rust.

^ The same mechanisms that are used to clean up memory in Rust are used for all kinds of things. If the string in the previous examples had been a file object, the file handle would be closed when the value was destroyed. And the same mechanisms that are used to reason about sharing are used for things like thread safety.

---

# Thread Safety

^ Yes, Rust has compiler guaranteed thread safety. There's actually nothing
special about threads in the language. It simply uses its type system to
accomplish this. This is a slightly simplified version of what creating a new
thread looks like

---

```rust
// in the module thread

fn spawn<F>(func: F) where F: FnOnce() + Send
```

^ So this syntax is probably a little unfamiliar. At a high level, the function spawn is going to take a block, and presumably we'd run that block on another thread. It's very similar to how `Thread::new` works in Ruby.

^ Syntactically we first state the name of the function. Then in the angle brackets, we're taking "type variables". This means that we have what's called a generic function, where we don't actually know the type of what we're taking other than the constraints we place on it. In this case we have one type variable, called F.

^ spawn takes a single argument, called func, and then we state our constraints about F. F must be a function that can be called once, and must also implement a thing called Send. Both of these are traits.

---

# Traits

^ Traits are how we share behavior in Rust. They're kinda like interfaces in other languages, or modules in Ruby. If you've done any Haskell, they're pretty much literally Haskell's type classes, minus higher kinded types. If that last sentence made no sense to you, just ignore it. It's not important.

^ The closest thing in Ruby to a trait in Rust that you're likely familiar with is the Enumerable module. Enumerable is special because you don't just mix it in, you're expected to define some behavior for it as well. This is how traits work in Rust. We have one very similar to Enumerable, and it's called `Iterator`. You'd implement it like this:

---

```rust
impl Iterator for MyType {
    type Item = i32;

    fn next(&mut self) -> Option<i32> {
        // give the next item in the list
    }
}
```

^ Iterator wants two things from you. First it wants to know the type of the item you're iterating over, and then it wants a single function called `next`, where you return the next item in the sequence if there is any, or `None` if you're at the end of the list.

^ There were two traits we saw earlier. `FnOnce`, and `Send`. `FnOnce` is something you'd never implement manually, it's a thing the compiler does automatically for blocks in Rust. `Send` is what's called a marker trait. A marker trait is what we call a trait with no behavior. We only use it to mark types. In the case of `Send`, it's used to mark a type as safe to give to another thread.

---

# Sharing between threads

^ There's a second type called `Sync`, which marks a type as safe to share between threads. You'll notice that the function to spawn threads didn't mention `Sync` at all. And it doesn't have to. That's because to share something is with a reference. And inside of Rust's standard library, you'll find this

---

```rust
impl<T> Send for &T where T: Sync
```

^ This means that a reference to a thing isn't `Send` unless the thing being referenced is `Sync`.

---

# Rust Uses its Type System to Enforce Best Practices

^ Now this is a lot of information, I know. I don't want to get too bogged down in the specifics. They're a lot easier to understand in context once you try to use the language. But the point I want to hammer here is that basically by default, sharing something across threads isn't allowed by the compiler.

^ And if you do want to share something (particularly if you want to share something and be able to mutate it), the only way to do that in Rust is to use a mutex, which is a way to ensure only one thread can access a value at a time.

---

# Why GC isn't enough

^ And this leads into an interesting point as well. You may be wondering why we don't just have File objects close themselves when they get garbage collected in Ruby. And you can accomplish this if you want to, using a thing called a finalizer. However, the issue with finalizers is that you don't know when they'll be run, if they get run at all. And this means that when you're dealing with a resource that is limited, you have to go back to remembering to clean up after yourself.

^ One particularly important case is a Mutex. When you want to access the data a mutex is protecting, you need to lock it. When you're done, you need to unlock it. And you absolutely cannot wait for GC to run to unlock the Mutex. Once again, in Rust, this isn't an issue.

---

```rust
fn do_shared_stuff(mutex: Mutex<MyType>) {
    let value = mutex.borrow_mut()
    // this will block until the mutex is unlocked, and lock it.
    // the value will be wrapped in a `LockGuard` but you don't need to care
    // in your code

    value.do_thread_unsafe_stuff()
    // we can be guaranteed no other thread is accessing the value here
}
// the lock is destroyed, so the mutex is automatically unlocked
```

---

# You can use these concepts in your own code

^ And there's nothing special about these constructs for thread safety. They're just clever uses of the type system. You can enforce all kinds of correctness in your own. For example, I'm currently working on an ORM for Rust. Tables and columns are represented as types, rather than strings, and I use this to ensure that you don't write queries that would error at runtime. For example, the signature of select is this:

---

```rust
fn select<S>(&self, s: S) -> SelectStatement
  where S: SelectableExpression<Self>
```

^ What this is saying is that you can only call `select` with arguments that can be selected from `Self`. The `SelectableExpression` trait gets implemented for all columns of a table, saying they can be selected from that table, and things like joins between tables can have anything selected that is able to be selected on either side. It even goes so far as to enforce things like if it's a left outer join, columns on the right side are now nullable.

^ The usage feels remarkably like a high level ORM like Active Record, as well. For example, this is a simple usage

---

```rust
users.select((id, name))
```

^ And while usage still feels high level, behind the scenes I'm giving you lots for free. For example, if you look at the final machine code that this compiles to, this will just compile to a string literal. There's no runtime AST walking or anything expensive like we do in Active Record. And the compiler is now verifying the correctness of that query. These for example, won't compile.

---

```rust
users.select(posts::id)
// can't select a column from another table

users.select((id, sum(post_count))
// can't select both aggregate and non-aggregate without a group by clause
```

^ And also one of my favorite features of the tooling is that I have test cases verifying that those will continue to fail to compile.

---

# Rust helps me write better Ruby

^ And I think this is the biggest reason everyone in this room should care about Rust. Rust has a truly novel way of representing ownership. Once you start to get into that world, you start to really see those concepts when writing code in other languages. And I think my Ruby code has improved as a result of learning Rust.

---

# Information overload!

^ Anyway I know this is a ton of information, and I think this is a good place to cut it off. Rust is an incredibly powerful and expressive language. I hope I've convinced you to take a look at it. Feel free to come talk to me if you'd like me to go more in depth on anything.

---

# The End

## (Please ask me questions now)

---

# Sean Griffin

- Twitter: [@sgrif](http://twitter.com/sgrif)
- Github: [sgrif](http://github.com/sgrif)
- Email: [sean@seantheprogrammer.com](mailto:sean@seantheprogrammer.com)
- Podcast: [bikeshed.fm](http://bikeshed.fm)
- Rust ORM: [github.com/sgrif/yaqb](github.com/sgrif/yaqb)
