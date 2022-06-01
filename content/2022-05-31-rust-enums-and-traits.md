+++
title="sql-press: On Rust Traits and Enums"
description="sql-press is a library crate, which describes database migrations as code and transforms it to your favorite sql dialect. sql-press is written in Rust and uses Traits as essential building block. This article explains the use of Enums vs Traits for such use-cases and discusses the respective up- and downsides."
date=2022-05-31


[taxonomies]
tags = ["rust", "trait", "enum", "sql", "ddl"]

[extra]
ToC = true
+++

# sql-press Introduction
[sql-press][0] is a Rust library crate, which I use to generate my database
migration changes. The excellent crate [refinery][1] is used as executor and
tracker of those individual changes. An example might illustrate it better:

```rust
use sql_press::{
    change::ChangeSet,
    column::{integer, varchar},
    sql_dialect::Postgres,
};

pub fn migration() -> String {
    let mut cs = ChangeSet::new();

    cs.create_table("my_table", |t| {
        t.add_column(integer("id").primary(true).build());
        t.add_column(varchar("name", Some(255)).not_null(true).build());
    });

    cs.get_ddl(Postgres::new_rc());
}
```

The above script would create the following Postgres change:

```sql
CREATE TABLE public."my_table" (
"id" integer PRIMARY KEY,
"name" VARCHAR(255) NOT NULL
);
```

Refinery searches for `.rs` files in a configurable directory, containing a
function with signature `pub fn migration() -> String` and takes it's return value and
applies it to the configured database. Refinery also supports reading plain
`.sql` files. For more information, please see the docs of refinery's
[embed_migrations()! macro][2].

The sql-press user-central data structure is a `ChangeSet`. A user creates a new
mutable `ChangeSet` and then runs functions on it, which record changes. At the
end, the user calls `get_ddl()` with the desired SQL dialect (currently only
`PostgreSQL` is supported - contributions are welcome) and generates a simple
String which contains all recorded changes.

sql-press supports other operations like:
- alter_table:
  ```rust
  cs.alter_table("my_table", |t| {
      t.rename_column("name", "slug");
      t.alter_column("slug", ColumnType::TEXT, None);
  });
  ```
- rename_table or drop_table:
  ```rust
  cs.rename_table("my_table", "my_renamed_table");
  cs.drop_table("my_renamed_table");
  ```

# Design Goals
When using sql-press, I wanted it to be as ergonomic as possible to the user.
This results in the following basic requirements:
1. provide different operations for `create_table` and `alter_table`,
2. support different sql dialects (e.g. postgres, mysql, mssql).

In the following sections, I will discuss the currently implemented Trait
approach and will compare it to my first attempt of using Enums. 

# Traits vs. Enums
Dislaimer: I want to mention that I am neither a Rust professional, nor are my
opinions a non-plus ultra. I am still learning Rust and am open to discuss
opinions which help me and - in the best case - also the Rust community
(especially the newer members) to learn and improve.

Most of my professional Software Engineer life, I was working
(programming-language wise) with PHP (yes, this is how I started), Python, Java
and a tiny bit of C++. For me, using traits was (and is) substantially different
to any kind of object-oriented programming I did before. This holds especially
true because of Rusts special requirements (e.g. [object safety][3]) on what can
be implemented as trait functions and what not.

## Providing context-dependent operations
Let's start with some example code on the different ergonomics of
`create_table` and `alter_table`.

```rust
let mut cs = ChangeSet::new();

cs.create_table("new_table", |table| {
    table.add_column(integer("id").primary(true).build());
    table.rename_column("id", "id2"); // -> fails in the create_table scope
});

cs.alter_table(|table| {
    table.rename_column("id", "id2"); // -> works in the alter_table scope
});
```

### Traits
The above behaviour is implemented with trait bounds in the function signature
of the `{create|alter}_table` functions.

Let's have a look at both definitions:

**create_table**
```rust
pub fn create_table<H>(&mut self, name: &str, handler: H)
where
    H: FnOnce(&mut dyn ColumnCreate),
{ ... }
```

**alter_table**
```rust
pub fn alter_table<H>(&mut self, name: &str, handler: H)
where
    H: FnOnce(&mut dyn ColumnAlter),
{ ... }
```

The `handler` argument of both functions is a generic argument `H`, which must be an
anonymous function which takes exactly one parameter which is a mutable
dynamically dispatched trait object of either `ColumnCreate` or `ColumnAlter`.

Let's look at the definitions of those two:

**ColumnCreate**

`ColumnCreate` has two so-called [Supertraits][4] which it automatically
"inherits".
```rust
pub trait ColumnCreate: ColumnAdd + IndexAdd {}

pub trait ColumnAdd {
    fn add_column(&mut self, column: ColumnAddChange);
}

pub trait IndexAdd {
    fn add_foreign_index(
        &mut self,
        column_name: &str,
        foreign_table_name: &str,
        foreign_column_name: &str,
        idx_name: Option<String>,
    );

    fn add_primary_index(&mut self, columns: Vec<&str>);

    fn add_unique_constraint(&mut self, constraint_name: &str, columns: Vec<&str>);
}
```
This means, that the `ColumnCreate` trait bound in the `create_table` function
automatically "provides" the functions of both traits `ColumnAdd` and `IndexAdd` to be
used. 

The same behaviour is applicable for the below described `ColumnAlter` trait and
it's supertraits.

**ColumnAlter**

`ColumnAlter` also has two [Supertraits][4] which it automatically "inherits".
```rust
pub trait ColumnAlter: ColumnDrop + IndexAlter {
    fn add_column(&mut self, column: ColumnAddChange);

    fn rename_column(&mut self, column_name: &str, new_column_name: &str);

    fn alter_column(
        &mut self,
        column_name: &str,
        new_column_type: ColumnType,
        conversion_method: Option<String>,
    );
}

pub trait ColumnDrop {
    fn drop_column(&mut self, name: &str);
    fn drop_column_if_exists(&mut self, name: &str);
}

pub trait IndexAlter {
    fn add_foreign_index(
        &mut self,
        column_name: &str,
        foreign_table_name: &str,
        foreign_column_name: &str,
        idx_name: Option<String>,
    );

    fn add_primary_index(&mut self, columns: Vec<&str>);

    fn add_unique_constraint(&mut self, constraint_name: &str, columns: Vec<&str>);

    // todo:
    // fn drop_unique_constraint();
    // fn drop_foreign_index();
    // ...
}
```

The combination of trait bounds and 

### Enums

## Supporting different SQL dialects
### Traits

### Enums

# Final thoughts

[0]: https://github.com/elbart/sql-press
[1]: https://crates.io/crates/refinery
[2]: https://docs.rs/refinery/0.8.4/refinery/macro.embed_migrations.html
[3]: https://doc.rust-lang.org/reference/items/traits.html#object-safety
[4]: https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#using-supertraits-to-require-one-traits-functionality-within-another-trait