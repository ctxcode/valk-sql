
# valk-sql

One API over the SQL databases [Valk](https://valk-lang.dev) speaks: SQLite, MySQL and
Postgres. It holds what every program writes again otherwise — a connection pool, migrations, a
query builder, rows read into your own classes — and the driver packages plug into it.

Requires Valk 0.7.3 or newer, whose method chains the builder is written for. This package has
no dependencies of its own: it talks to a driver
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

// Values go in by name, whatever the database is; they are bound, never pasted in
db.exec("INSERT INTO users (name, age) VALUES (:name, :age)", .{ "name" => "Ada", "age" => 36 }) ! panic("%{E.message}")

let rows = db.all("SELECT * FROM users WHERE age > :age", .{ "age" => 18 }) ! panic("%{E.message}")
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

Numbers, text, bools, JSON values and arrays of them turn into values by themselves. A blob is
made with `sql.Value.of_blob`, and `sql.convert(x)` converts whatever a variable holds.

## Values by name

Every statement takes its values by name. A name may appear more than once, and an array becomes
a list, so an `IN` takes any number of ids from one placeholder:

```rust
let orders = db.all("SELECT * FROM orders WHERE customer = :who AND status IN (:statuses) OR referrer = :who", .{
    "who" => 7
    "statuses" => Array[String]{ "new", "paid" }
}) ! panic("%{E.message}")

let found = db.all("SELECT * FROM users WHERE id IN (:ids)", .{ "ids" => ids }) ! panic("%{E.message}")
```

An empty list becomes `NULL`, so `IN (:ids)` matches nothing and `NOT IN (:ids)` matches nothing
either. Text in quotes and Postgres casts such as `::int` are left alone, and a `?` of your own,
such as the Postgres JSON operator, stays what it is. A name without a value throws `syntax`.
`sql.named(statement, values, dialect)` does the conversion by itself, for code that passes the
statement on.

## The query builder

The builder writes the statement and keeps the values apart from it. Every method returns the
query, so a query reads as one chain:

```rust
let rows = db.select("users.name, count(posts.id) AS posts")
    .from("users")
    .join("LEFT JOIN posts ON posts.user_id = users.id")
    .where("users.active", true)
    .where("users.id", ids)
    .group_by("users.id")
    .having("count(posts.id)", ">", 2)
    .order_by("posts DESC")
    .limit(10)
    .all() ! panic("%{E.message}")
```

`where` takes a column and a value, or a column, an operator and a value:

```rust
query.where("name", "Ada")                  // name = ?
query.where("age", ">=", 18)                // =, !=, <>, <, >, <=, >=, like, not like, ilike, not ilike
query.where("deleted_at", null)             // deleted_at IS NULL
query.where("banned_at", "!=", null)        // banned_at IS NOT NULL
query.where("id", ids)                      // id IN (...)
query.where("team", "not in", teams)        // team NOT IN (...)
query.where_raw("lower(email) = :email", .{ "email" => email })
```

The column is written as it is given, so `users.id` works; it must come from your program, never
from the input it handles. What `where` cannot say goes into `where_raw`, with values by name.
A mistake, such as an operator it does not know, is thrown as `syntax` when the query runs.

## Mixing AND and OR

`where` joins with `AND` and `or_where` with `OR`. Where the two are mixed, a group puts the
brackets in, so what binds to what is never left to precedence:

```rust
// WHERE (active = ?) AND ((role = ?) OR (score > ?))
let rows = db.select()
    .from("users")
    .where("active", true)
    .where_group(fn(w: sql.Conditions) {
        w.where("role", "admin")
        w.or_where("score", ">", 100)
    })
    .all() ! panic("%{E.message}")
```

Groups nest, so the other shape — `OR` of two `AND`s — reads the same way:

```rust
// WHERE ((role = ?) AND (active = ?)) OR ((role = ?) AND ((score > ?) OR (score IS NULL)))
query.where_group(fn(w: sql.Conditions) {
    w.where("role", "admin")
    w.where("active", true)
})
query.or_where_group(fn(w: sql.Conditions) {
    w.where("role", "owner")
    w.group(fn(inner: sql.Conditions) {
        inner.where("score", ">", 50)
        inner.or_where_null("score")
    })
})
```

Inside a group the methods are `where`, `or_where`, `where_raw`, `or_where_raw`, `group`,
`or_group`, `where_in`, `where_not_in`, `where_null`, `or_where_null`, `where_not_null` and
`or_where_not_null`. Values come out in the order the placeholders take them, however deep the
nesting goes, and `HAVING` mixes the same way with `having`, `or_having`, `having_raw` and
`having_group`.

`run()` is for writes, `all()`, `one()` and `value()` for reads, and `to_sql(dialect)` returns
the statement without running it, which is what makes the builder easy to test. Writes are built
the same way:

```rust
db.insert_into("users")
    .values(.{ "name" => "Ada", "age" => 36 })
    .run() ! panic("%{E.message}")

db.update("users")
    .set("active", false)
    .set_expression("visits", "visits + :step", .{ "step" => 1 })
    .where("id", 7)
    .run() ! panic("%{E.message}")

db.delete_from("users")
    .where("id", 7)
    .run() ! panic("%{E.message}")
```

## Joins, pages and upserts

```rust
query.join_on("posts", "posts.user_id = users.id")     // JOIN posts ON ...
query.left_join("teams", "teams.id = users.team_id")   // LEFT JOIN teams ON ...
query.page(2, 20)                                      // LIMIT 20 OFFSET 20, counting from page 1
let total = query.count() ! panic("%{E.message}")      // the same query as a count(*)
```

`count` runs the query the caller built — its table, joins and conditions — without the order,
limit and offset, which is the number that belongs next to a page of rows.

An upsert is written differently by every one of these databases, and the builder writes the
right one:

```rust
db.insert_into("counters")
    .values(.{ "name" => "visits", "n" => 1 })
    .on_conflict_update(.{ "name" }, .{ "n" })
    .run() ! panic("%{E.message}")

// SQLite, Postgres: INSERT INTO counters (name, n) VALUES (?, ?)
//                   ON CONFLICT (name) DO UPDATE SET n = excluded.n
// MySQL:            INSERT INTO counters (name, n) VALUES (?, ?)
//                   ON DUPLICATE KEY UPDATE n = VALUES(n)
```

`on_conflict_nothing` keeps the row that is already there. The columns that decide what a clash
is are what SQLite and Postgres need; MySQL works that out from its own indexes and ignores them.

## Walking a large result

`all()` holds every row in memory, which is fine for a page and wrong for a table. There are
three ways to walk more rows than fit.

`each_row` streams them one at a time, straight from the connection:

```rust
db.each_row("SELECT id, email FROM users", null, fn(row: Map[sql.Value]) bool !sql.Error {
    send_email(row)
    return true                      // false stops the walk
}) ! panic("%{E.message}")
```

Nothing else may run on that connection while it walks: a statement sent in the middle takes the
connection, and the walk then throws rather than quietly stopping halfway. For a walk that
writes, take a page at a time instead.

`chunk` runs the query again per page, with a `LIMIT` and an `OFFSET`, so the connection is free
in between:

```rust
db.select().from("users").order_by("id").chunk(500, fn(rows: Array[Map[sql.Value]]) bool !sql.Error {
    each rows as row : send_email(row)
    return true
}) ! panic("%{E.message}")
```

`chunk_by_id` walks forward by a column instead of counting rows, which is the one to use on a
table that is being written:

```rust
db.select().from("users").chunk_by_id("id", 500, fn(rows: Array[Map[sql.Value]]) bool !sql.Error {
    each rows as row {
        db.exec("UPDATE users SET checked = 1 WHERE id = ?", .{ row.get("id") !? sql.Value.null() }) !>
    }
    return true
}) ! panic("%{E.message}")
```

Each page asks for `column > the last value seen`, so a row inserted or deleted while the walk
runs cannot make it skip a row or see one twice — which an `OFFSET` can — and the database uses
the index on that column instead of counting its way to a deep offset. The column must be unique
and never decrease: a primary key, or a `uuid.v7()`. The order is set by the method.

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

A transaction that fits in a closure commits when the closure returns and rolls back when it
throws:

```rust
db.transaction(fn(tx: sql.Db) !sql.Error {
    tx.exec("UPDATE accounts SET balance = balance - :amount WHERE id = :id", .{ "amount" => 10, "id" => 1 }) !>
    tx.exec("UPDATE accounts SET balance = balance + :amount WHERE id = :id", .{ "amount" => 10, "id" => 2 }) !>
}) ! panic("%{E.message}")
```

For work that does not fit in one closure, `begin_transaction` hands the transaction back
instead. `close` rolls back unless the work was committed, so a `defer` covers every way out of
the function — an early return, a thrown error, a panic:

```rust
let tx = db.begin_transaction() ! panic("%{E.message}")
defer tx.close()

tx.exec("UPDATE accounts SET balance = balance - :amount WHERE id = :id", .{ "amount" => amount, "id" => from }) !>
if (tx.value("SELECT balance FROM accounts WHERE id = :id", .{ "id" => from }) !>).to_int() < 0 {
    return "not enough money"      // the defer rolls it back
}
tx.exec("UPDATE accounts SET balance = balance + :amount WHERE id = :id", .{ "amount" => amount, "id" => to }) !>
tx.commit() !>
```

A `Tx` takes the same statements and builders as a `Db`. The plain `begin`, `commit` and
`rollback` are there as well, for a transaction that is handed around by something else.

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

Measured against SQLite in memory, 50 000 rows, best of three (`bench/` in valk-sqlite):

| | driver | through sql |
| --- | --- | --- |
| 50 000 inserts in one transaction | 13 ms | 15 ms |
| reading 50 000 rows of 3 columns | 4 ms | 5 ms |
| 50 000 single-row selects | 16 ms | 21 ms |

Both sides take their values by name. A `Db` remembers the statements it has run (256 of them),
so a statement it saw before skips reading its names again; what is left is about a tenth of a
microsecond per statement for the map of values. Reading fills the row straight from the
statement rather than building the driver's own row first. Against a database on a socket, all
of this disappears into the round trip.

The driver's own API is untouched and stays available for the paths where every allocation
counts: `sqlite.Connection` and the others work exactly as before.

## Upgrading from 0.2

- Statements take their values by name: `db.exec("... WHERE id = :id", .{ "id" => 7 })` where it
  was `db.exec("... WHERE id = ?", .{ 7 })`. `?` is no longer a placeholder.
- `where("age > ?", .{ 18 })` is `where("age", ">", 18)`, and `where("id = ?", .{ 7 })` is
  `where("id", 7)`. A condition that needs SQL goes into `where_raw` with values by name; the same
  holds for `having`, `set_expression` and the joins.
- `sql.rewrite` and `sql.placeholders` are gone: a list is an array bound to one name.

## Development

`make test` runs the suite, which needs no database: the tests drive a fake driver.
`make example` runs the example, `make lint` checks the sources and `make docs` regenerates the
API documentation.
