
# valk-sql

One API over the SQL databases [Valk](https://valk-lang.dev) speaks: SQLite, MySQL and
Postgres. It holds what every program writes again otherwise — a connection pool, migrations, a
query builder, rows read into your own classes — and the driver packages plug into it.

Requires Valk 0.7.0 or newer. This package has no dependencies of its own: it talks to a driver
through an interface, and the driver packages implement it.

## Install

```
vman install github.com/ctxcode/valk-sql
```

Together with a driver, whichever you use:

```
vman install github.com/ctxcode/valk-sqlite
vman install github.com/ctxcode/valk-postgres
vman install github.com/ctxcode/valk-mysql
```

## Example

```rust
use sql
use sqlite

let db = sqlite.database(sqlite.open("app.db") ! panic("%{E.message}"))
defer db.close()

// Statements are written with `?` whatever the database is; Postgres gets $1, $2 on the way out
db.exec("INSERT INTO users (name, age) VALUES (?, ?)", .{ sql.Value.of("Ada"), sql.Value.of_int(36) }) ! panic("%{E.message}")

let rows = db.all("SELECT * FROM users WHERE age > ?", .{ sql.Value.of_int(18) }) ! panic("%{E.message}")
each rows as row {
    println((row.get("name") !? sql.Value.null()).to_string())
}

let total = (db.value("SELECT count(*) FROM users") ! panic("%{E.message}")).to_int()
```

`exec` returns the rows it changed, `all` every row, `one` the first row or null, `value` a
single value, and `query` hands back a `Rows` to read one row at a time when the result is too
large to hold in memory.

The same code runs on another database by changing the line that opens it. What differs between
them — placeholders, `RETURNING`, `LIMIT`/`OFFSET`, the id of an insert — is what this package
smooths over; SQL that is special to one database still is.

## Values

A row is a `Map[sql.Value]`, and `sql.Value` is a struct, so a row costs one allocation rather
than one per column.

```rust
value.is_null()
value.to_string()           // numbers are formatted, NULL is ""
value.to_string_or_null()   // null for NULL
value.to_int()
value.to_float()
value.to_bool()
value.to_json()
```

Values going in are made with `sql.Value.of`, `of_int`, `of_float`, `of_bool` and `of_blob`, or
with `sql.convert(x)` for whatever a variable happens to hold.

## The query builder

The builder writes the statement and keeps the values apart from it. Every method returns the
query, so short queries read as one line; longer ones are built statement by statement, since a
method chain in Valk stays on one line:

```rust
let rows = db.select().from("users").where("id = ?", .{ sql.Value.of_int(7) }).all() ! panic("%{E.message}")

let query = db.select("users.name, count(posts.id) AS posts")
query.from("users")
query.join("LEFT JOIN posts ON posts.user_id = users.id")
query.where("users.active = ?", .{ sql.Value.of_bool(true) })
query.where_in("users.id", ids)
query.group_by("users.id")
query.having("count(posts.id) > ?", .{ sql.Value.of_int(2) })
query.order_by("posts DESC")
query.limit(10)
let rows = query.all() ! panic("%{E.message}")
```

`run()` is for writes, `all()`, `one()` and `value()` for reads, and `to_sql(dialect)` returns
the statement without running it, which is what makes the builder easy to test. Writes are built
the same way:

```rust
db.insert_into("users").values(.{ "name" => sql.Value.of("Ada"), "age" => sql.Value.of_int(36) }).run() ! panic("%{E.message}")
db.update("users").set("active", sql.Value.of_bool(false)).where("id = ?", .{ sql.Value.of_int(7) }).run() ! panic("%{E.message}")
db.delete_from("users").where("id = ?", .{ sql.Value.of_int(7) }).run() ! panic("%{E.message}")
```

Conditions are text on purpose: the builder saves the placeholder bookkeeping, not SQL itself.

## Rows into your own classes

```rust
class User {
    id: uint
    name: String
    nickname: ?String
}

let users = sql.rows_to[User](db.all("SELECT id, name, nickname FROM users") ! panic("%{E.message}")) ! panic("%{E.message}")
```

Fields are matched by name; a nullable field takes a NULL column and a field with a default may
be missing. SQLite and MySQL have no boolean type and store 1 and 0, which no longer look like
a bool, so name those columns: `sql.rows_to[User](rows, .{ "active" })`. Postgres booleans need
no help. This goes through the JSON decoder, so it is for readable code rather than the hottest
loop.

## Transactions

```rust
db.transaction(fn(tx: sql.Db) !sql.Error {
    tx.exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", .{ sql.Value.of_int(10), sql.Value.of_int(1) }) !>
    tx.exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", .{ sql.Value.of_int(10), sql.Value.of_int(2) }) !>
}) ! panic("%{E.message}")
```

It commits when the work returns and rolls back when it throws. `begin`, `commit` and
`rollback` are there for a transaction that spans more than one function.

## Migrations

```rust
let migrator = sql.Migrator.new(db)
migrator.add_many(#embed_dir("migrations"))
let done = migrator.migrate() ! panic("Migration failed: %{E.message}")
each done as name : println("ran " + name)
```

`#embed_dir` puts the files inside the program, so no directory has to exist where it runs. They
run in the order of their names (`0001-users.sql`, `0002-posts.sql`, ...), each in a
transaction, and the names that ran are kept in a `schema_migrations` table so a second run does
nothing. `pending()` says what a run would do.

## Pools

```rust
global pool: sql.Pool (sql.Pool.new(fn() sql.Db !sql.Error {
    return sqlite.database(sqlite.open("app.db") ! throw .connect { message: E.message })
}, 8))

fn handler(req: http.Request) http.Response {
    let db = pool.get() ! return http.Response.text("database down", 503)
    defer pool.put(db)
    ...
}
```

A pool belongs to one thread, like the connections in it, so a server gives every worker thread
its own through a `global`. `get` waits when the pool is at `max_connections` and everything is
handed out, and the wait yields to the other coroutines on the thread.

## Errors

Every method throws `sql.Error`, whichever driver is behind it:

| code | when |
| --- | --- |
| `connect` | the database could not be opened or reached |
| `syntax` | the statement could not be prepared |
| `constraint` | a `UNIQUE`, `NOT NULL`, `CHECK` or foreign key rule was broken |
| `busy` | the database was held by someone else for too long |
| `readonly` | the database cannot be written |
| `timeout` | no connection came back from the pool in time |
| `closed` | the connection is closed |
| `error` | anything else the driver reported |

`driver_code` keeps the code the driver gave, such as a SQLSTATE (`23505`), for the cases where
the exact reason matters.

## Writing a driver

A driver implements two interfaces, `sql.Driver` and `sql.Rows`, and hands out a `sql.Db`
wrapping itself. `example/main.valk` has a working one in about forty lines. The driver packages
of this account do it for you.

## What it costs

Going through this package adds one interface call per statement and one struct per column on
top of the driver. It does not change the driver's own API, which stays available for the paths
where every allocation counts: `sqlite.Connection` and the others work exactly as before.

## Development

`make test` runs the suite, which needs no database: the tests drive a fake driver.
`make example` runs the example, `make lint` checks the sources and `make docs` regenerates the
API documentation.
