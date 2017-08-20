# Diesel

## A safe and extensible ORM and Query Builder for Rust

---

```rust





pub struct TestCode {
    fn can_you_read_this_color_scheme() -> bool {
        // ???
    }
}
```

---

# Who am I?

- Sean Griffin
- 10x Hacker Ninja Guru at Shopify
- Rails Committer
- Maintainer of Active Record
- Creator of Diesel
- Bikeshed co-host

---

# Why Diesel?

![fit](diesel.png)

---

```rust





let stmt = try!(conn.prepare("SELECT * FROM users \
                              WHERE users.name = $1 LIMIT 1"));
let rows = try!(stmt.query(&[&name]));
let row = rows.iter().next();
Ok(User::from_row(&row))
```

^ The first reason is purely convenience. I don't want to have to write code that looks like this.

---

```rust






users.filter(name.eq(&name))
    .first(&connection)
```

^ I want to write code that looks like this. And I'm leaving out the boilerplate code to build a struct for mapping the database rows into a struct, but in Diesel it's just...

---

# `#[derive(Queryable)]`

---

```sql






SELECT * FROM users WHERE users.name = $1
```

^ One of the questions I get sometimes is: "Why is the diesel equivalent of this SQL any better". There's many reasons, but one of the key points that's missing here is that a SQL string isn't something that we can statically check. And I have fat fingers, so most likely I won't have written this, I'll have written something more like...

---

```sql






SELECT * FROM users WHERE usres.nme = $3
```

^ And that means that I'll get a runtime error from the database.

---

> thread '<main>' panicked at 'Sean you are a fucking moron. Give up on life'
-- What I see when I get database errors

^ My computer and I have a bit of a troubled relationship...

---

# I hate runtime errors

---

# I hate runtime errors
# (Especially in Rust)

^ RIP stack traces

---

```





thread '<main>' panicked at 'called `Result::unwrap()`
on an `Err` value: "LOL STACKTRACES"',
../src/libcore/result.rs:741
```

---

# Abstractibility

^ Which I'm pretty sure is a word that I just made up.

---

```rust
let versions = Version::belonging_to(krate)
    .select(id)
    .order(num.desc())
    .limit(5);
let downloads = try!(version_downloads
    .filter(date.gt(now - 90.days()))
    .filter(version_id.eq(any(versions)))
    .order(date)
    .load::<Download>(&conn));
```

^ So this is some code that comes out of one of the more involved queries in crates.io when ported to Diesel. I really like this example, as it's a fairly complex query that involves subselects, the `= ANY` operator from postgres, SQL functions, subtracting intervals from dates, etc.

^ Compared to the original code (which won't fit nicely on a slide), it is more terse and has static checking, but we can do better.

^ Talk about SQL strings not being composeable

---

```rust
let versions = krate.latest_versions()
    .limit(5);
let downloads = try!(version_downloads
    .filter(date.gt(now - 90.days()))
    .filter(version_id.eq(any(versions)))
    .order(date)
    .load::<Download>(&conn));
```

^ We can pull out portions of this into another function. For example, getting the versions for a crate ordered by number in descending order is probably pretty common. We can call it `latest_versions()`. But one of the important things to note is that we don't have to pull that whole expression out. We can still modify the query later (within reason).

^ Just to drive home how non-composeable SQL strings are, I'd like to show what that first bit looks like in the actual crates.io code

---

```rust





let mut versions = try!(krate.versions(tx));
versions.sort_by(|a, b| b.num.cmp(&a.num));
let to_show = &versions[..cmp::min(5, versions.len())];
let ids = to_show.iter().map(|i| i.id).collect::<Vec<_>>();
```

^ There was already a versions function on Crate, which gets the versions belonging to it. But this function actually executes the query. And because it's so hard to abstract that in a composeable way when using SQL directly, the code actually does the sorting and limiting in Rust instead of SQL.

^ The point of this isn't to rag on the code, it's what I would probably do in that situation, as it's just much easier than the alternative.

---

```rust
let versions = krate.latest_versions()
    .limit(5);
let downloads = try!(version_downloads
    .filter(date.gt(now - 90.days()))
    .filter(version_id.eq(any(versions)))
    .order(date)
    .load::<Download>(&conn));
```

^ Back to our original example, we can still pull stuff out. For example, that `now - 90.days()` line is something that has a better meaning which we can give a name to, and put it somewhere that we can define it separately from our actual query.

---

```rust
let versions = krate.latest_versions()
    .limit(5);
let downloads = try!(version_downloads
    .filter(Download::recent())
    .filter(version_id.eq(any(versions)))
    .order(date)
    .load::<Download>(&conn));
```

^ So we can pull out that predicate into a function called `recent`. We can go farther if we wanted to, but we'll stop here in the interest of time as this is just to make a point.

---

# Abstractions are cool

^ Abstractions aren't something just for library authors. It's something that you build in your own code bases as they get larger in size.

---

# I want to think about my problem domain

^ And since my app is in Rust, that presumably means that my problem domain is in Rust, not SQL.

---

# How does it all work?

^ So that's the intro to Diesel. I'll stop pitching now, and let's take a look at how it works.

---

```rust






users.filter(name.eq("Sean"))
```

---

# Macros and codegen

```rust
table! {
    users {
        id -> Serial,
        name -> String,
        hair_color -> Nullable<String>,
    }
}
```

---

```rust







infer_table_from_schema!("users", env!("DATABASE_URL"))
```

---

```rust







infer_schema!(env!("DATABASE_URL"))
```

---

```rust



pub mod users {
    pub struct table;
    pub struct id;
    pub struct name;
    pub struct hair_color;

    /* Boilerplate impls for all of the things */
}
```

---

```rust






impl Expression for users::name {
    type SqlType = VarChar;
}
```

^ Don't assume that everyone writes compilers or AST builders and knows what an
expression is. Explain it

---

```rust







impl SelectableExpression<users::table> for users::name {
}
```

---

```rust






fn eq<T>(other: T) -> Eq<Self, T::Expression> where
    T: AsExpression<Self::SqlType>,
```

---

```rust





name.eq("Sean");
```

---

```rust





name.eq("Sean");
name.eq(String::from("Sean"));
```

---

```rust





name.eq("Sean");
name.eq(String::from("Sean"));
name.eq(hair_color);
```

---

```rust





name.eq("Sean");
name.eq(String::from("Sean"));
name.eq(hair_color);
name.eq(lower(coalesce(hair_color, "Jim")));
```

---

```rust




impl<'a> AsExpression<VarChar> for &'a str {
    type Expression = Bound<&'a str, VarChar>;

    fn as_expression(self) -> Self::Expression {
        // ...
    }
}
```

---

# Things weren't always the cleanest

---

```rust







fn find<T, U, PK>(&self, source: T, id: PK) -> QueryResult<U> where
    T: Table + FilterDsl<FindPredicate<T, PK>>,
    FindBy<T, T::PrimaryKey, PK>: LimitDsl,
    Limit<FindBy<T, T::PrimaryKey, PK>>: QueryFragment<Self::Backend>,
    U: Queryable<<Limit<FindBy<T, T::PrimaryKey, PK>> as Query>::SqlType, Self::Backend>,
    Self::Backend: HasSqlType<<Limit<FindBy<T, T::PrimaryKey, PK>> as Query>::SqlType>,
    PK: AsExpression<PkType<T>>,
    AsExpr<PK, T::PrimaryKey>: NonAggregate,
```

---

## (Don't worry, that where clause is gone now. It was horrible)

---

```rust









pub trait FindDsl<PK> where
    Self: Table + FilterDsl<Eq<<Self as Table>::PrimaryKey, PK>>,
    PK: AsExpression<<Self::PrimaryKey as Expression>::SqlType>,
    Eq<Self::PrimaryKey, PK>: SelectableExpression<Self, SqlType=Bool> + NonAggregate,
```

---

# The end result is safety

---

```rust






fn main() {
    let _ = users::table.filter(posts::id.eq(1));
    //~^ ERROR SelectableExpression
}
```

---

```rust



fn main() {
    use self::users::dsl::*;

    let pred = id.eq("string");
    //~^ ERROR E0277
    let pred = id.eq(name);
    //~^ ERROR type mismatch
}
```

---

# Safety allows speed

---

# Types are fucking cool

---

# http://diesel.rs

![fit](diesel.png)

---

# I have stickers!

![fit](diesel.png)

---

![fit](diesel.png)

---

# http://diesel.rs
# @dieselframework

![fit](diesel.png)
