# `ormlite`

**`ormlite` is an ORM in Rust for developers that love SQL.**

`ormlite` gives structs an ergonomic interface to communicate with the database (e.g. `insert`, `delete`, `update`), 
but stays close to SQL for all query building, both in syntax and performance.

The objectives of this project:

* **fast**: We aim for minimal to no measurable overhead for using the ORM.
* **true to SQL**: We eschew any custom query syntax, and stay true to SQL as much as possible while providing ORM and query builder functionality.
* **`async`-first**: We built on top of the great foundation of `sqlx`, providing `async` interfaces.

> **NOTE**: This is alpha-quality and being actively developed. In usage so far, the software is functional, performant, and correct, but until it undergoes more rigorous battle testing, we recommend vetting the code yourself before using the crate in production environments.

# Usage

```rust

use ormlite::model::*;

#[derive(ormlite::Model)]
pub struct Person {
    pub id: u32,
    pub name: String,
    pub age: u8,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// Start by making a sqlx connection.
    let mut conn = sqlx::SqliteConnection::connect_with(&sqlx::sqlite::SqliteConnectOptions::from_str("sqlite://:memory:").unwrap()).await?;
    
    /// You can insert the model directly.
    let mut john = Person {
        id: 1,
        name: "John".to_string(),
        age: 99,
    }.insert(&mut conn).await;
    
    /// After modifying the object, you can update all its fields.
    john.age += 1;
    john.update_all_fields(&mut conn).await?;
    
    /// Lastly, you can delete the object.
    john.delete(&mut conn).await?;
    
    /// You can use builder syntax to do insertion with only certain fields.
    let john = Person::build()
        .name("John".to_string())
        .age(99)
        .insert(&mut conn).await?;
    
    /// You can also use builder syntax to update only certain fields.
    let john = john.update_partial()
        .age(100)
        .update(&mut conn).await?;
 
    /// You can get a single user.
    let john = Person::get_one(1, &mut conn).await?;
  
    /// You can create a query builder.
    let people = Person::select()
            .filter("age > ?").bind(50)
            .fetch_all(&mut conn).await?;
  
    /// You can also fall back to raw sql.
    let people = Person::query("SELECT * FROM person WHERE age > ?")
            .bind(50)
            .fetch_all(&mut conn).await?;
    Ok(())
}
```

### Partial Structs

If, instead of builder syntax, you prefer to create partial structs to statically enforce which columns are affected for insertions, use the following:

```rust
use ormlite::model::*;

#[derive(ormlite::Model)]
#[ormlite(table_name = "person")]
pub struct InsertPerson {
    pub name: String,
    pub age: u8,
}

async fn do_partial_insertion() {
    let mut john = InsertPerson {
        name: "John".to_string(),
        age: 99,
    }.insert(&mut conn).await;
}
```

# Installation

For postgres:

    [dependencies]
    ormlite = { version = "0.1.0", features = ["postgres", "runtime-tokio-rustls"]

For sqlite:

    [dependencies]
    ormlite = { version = "0.1.0", features = ["sqlite", "runtime-tokio-rustls"]
    
    
Other databases (mysql) and runtimes should work smoothly, but might not be 100% wired up yet. Please submit an issue if you encounter issues.

# Roadmap
- [x] insert, update, delete directly on model instances
- [x] builder for partial update and insertions
- [x] user can create insert models that ignore default values
- [x] select query builder
- [x] build the derive macro
- [x] get() function for fetching a single entity.
- [x] ability to specify the name of a table and name of primary column
- [ ] macro option to auto adjust columns like updated_at
- [ ] upsert functionality
- [ ] joins
- [ ] bulk insertions
- [ ] query builder for bulk update
- [ ] automatically generate insert models
- [ ] benchmarks against raw sql, sqlx, ormx, seaorm, sqlite3-sys, pg, diesel
- [ ] macro option to delete with deleted_at rather than `DELETE`
- [ ] support for patch records, i.e. update with static fields.
```rust
/// You can also create objects to statically enforce update fields.
#[derive(TODO)]
struct UpdatePerson {
    age: u8,
}
fn main() {
    john.update_from(UpdatePerson {
        age: 100,
    }, &mut conn).await?;    
}
```

# Documentation

### Logging

You can log queries using sqlx's logger: `RUST_LOG=sqlx=info`