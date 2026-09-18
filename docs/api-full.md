
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown by every operation of this package, and by the drivers behind it.
+ error Error (connect, syntax, constraint, busy, readonly, timeout, closed, error) payload { message: String, driver_code: String (""), sql: String ("") }
```

### Error

Thrown by every operation of this package, and by the drivers behind it.

- `connect`: the database could not be opened or reached.
- `syntax`: the statement could not be prepared: a typo, an unknown table or column.
- `constraint`: a `UNIQUE`, `NOT NULL`, `CHECK` or foreign key rule was broken.
- `busy`: the database was held by someone else for longer than the driver waits.
- `readonly`: the database cannot be written.
- `timeout`: a connection did not come back from the pool in time.
- `closed`: the connection is closed.
- `error`: anything else the driver reported.

`driver_code` holds the code the driver gave, such as a SQLSTATE (`23505`) or a SQLite
result code, so a caller that needs the exact reason still has it.

## Enums for 'main'

```js
// Which database is on the other end, and so how a statement is written.
+ enum Dialect { sqlite, mysql, postgres }
// What a value holds. Every database this package speaks to has these five.
+ enum TYPE { null, bool, int, float, text, blob }
```

### Dialect

Which database is on the other end, and so how a statement is written.

### TYPE

What a value holds. Every database this package speaks to has these five.

## Functions for 'main'

```js
// Converts any supported value (integers, floats, bools, text, json values, and nullable versions of those) into a `Value`.
+ fn convert(ndata: $T) Value
// Returns `?, ?, ?` for `count` values, to write an `IN (...)` list by hand.
+ fn placeholders(count: uint) String
// Rewrites the `?` placeholders of a statement for `dialect`.
+ fn rewrite(sql: String, dialect: Dialect) String
// Reads a row into a class or struct of your own.
+ fn row_to[T](row: Map[Value], bool_columns: Array[String] (.{})) T !Error
// Returns the row as a JSON object, for an answer that goes straight out as JSON.
+ fn row_to_json_value(row: Map[Value], bool_columns: Array[String] (.{})) Value
// Reads every row into a class or struct of your own, as `row_to` does for one.
+ fn rows_to[T](rows: Array[Map[Value]], bool_columns: Array[String] (.{})) Array[T] !Error
```

### convert

Converts any supported value (integers, floats, bools, text, json values, and nullable
versions of those) into a `Value`.

### placeholders

Returns `?, ?, ?` for `count` values, to write an `IN (...)` list by hand.

### rewrite

Rewrites the `?` placeholders of a statement for `dialect`.

SQLite and MySQL take `?` as it is; Postgres wants `$1`, `$2`, ... Text in quotes is left
alone, so a `?` inside a string stays what it was.

### row_to

Reads a row into a class or struct of your own.

Every field of `T` must be a column of the row, unless the field is nullable or has a
default. Fields are matched by name: a text column reaches a `String` field and a number
column an `int` or a `float` field.

SQLite and MySQL have no boolean type and store 1 and 0, which cannot be told from any other
number, so name those columns in `bool_columns` to read them into a `bool` field. A Postgres
boolean needs no help.

This goes through the JSON decoder, so it is for readable code rather than for the hottest
loop; reading the columns by hand stays available.

```valk
class User {
    id: uint
    name: String
    nickname: ?String
}

let row = db.one("SELECT * FROM users WHERE id = ?", .{ 1 }) ! panic("%{E.message}")
if isset(row) {
    let user = sql.row_to[User](row) ! panic("%{E.message}")
    println(user.name)
}
```

### row_to_json_value

Returns the row as a JSON object, for an answer that goes straight out as JSON.

### rows_to

Reads every row into a class or struct of your own, as `row_to` does for one.

## Classes for 'main'

```js
// A database, whichever driver is behind it.
+ class Db {
    // The driver behind this database.
    + driver: Driver

    // Runs a statement and returns every row it answered with.
    + fn all(sql: String, args: Array[Value] (.{})) Array[Map[Value]] !Error
    // Opens a transaction.
    + fn begin(immediate: bool (false)) void !Error
    // Closes the connection.
    + fn close() void
    // Commits the open transaction.
    + fn commit() void !Error
    // Starts a `DELETE FROM` on this database.
    + fn delete_from(table: String) Query
    // Which database is on the other end.
    + get dialect: Dialect
    // Runs a statement that reads no rows, and returns how many rows it changed.
    + fn exec(sql: String, args: Array[Value] (.{})) uint !Error
    // Starts an `INSERT INTO` on this database.
    + fn insert_into(table: String) Query
    // The id the last insert wrote. SQLite and MySQL fill this in; Postgres has no such counter, so ask it for the id with `INSERT ... RETURNING id` instead.
    + fn last_insert_id() int
    // Wraps a driver. Drivers call this; a program calls the driver's own function.
    + static fn new(driver: Driver) Db
    // Runs a statement and returns its first row, or null when it answered with none.
    + fn one(sql: String, args: Array[Value] (.{})) ?Map[Value] !Error
    // Returns whether the connection still answers.
    + fn ping() bool
    // Runs a statement and returns its rows, to be read one at a time with `Rows.next`.
    + fn query(sql: String, args: Array[Value] (.{})) Rows !Error
    // Rolls the open transaction back.
    + fn rollback() void !Error
    // Starts a `SELECT` on this database.
    + fn select(columns: String ("*")) Query
    // Runs `work` inside a transaction: it commits when the work returns, and rolls back when it throws.
    + fn transaction(work: fn(Db)(!Error), immediate: bool (false)) void !Error
    // Starts an `UPDATE` on this database.
    + fn update(table: String) Query
    // Runs a statement that answers with one value, and returns it. NULL when there is no row.
    + fn value(sql: String, args: Array[Value] (.{})) Value !Error
}
```

### Db

A database, whichever driver is behind it.

A driver package hands one of these out (`sqlite.database(con)`, `postgres.database(con)`,
`mysql.database(con)`), and everything in this package works with it. Statements are written
with `?` placeholders whatever the database is; the values are bound, never pasted in.

```valk
let db = sqlite.database(con)
db.exec("INSERT INTO users (name, age) VALUES (?, ?)", .{ "Ada", 36 }) ! panic("%{E.message}")
let rows = db.all("SELECT * FROM users WHERE age > ?", .{ 18 }) ! panic("%{E.message}")
```

#### driver

The driver behind this database.

#### all

Runs a statement and returns every row it answered with.

#### begin

Opens a transaction.

`immediate` takes the write lock at once where the database has one (SQLite), which a
transaction that reads a value and writes it back wants.

#### close

Closes the connection.

#### commit

Commits the open transaction.

#### delete_from

Starts a `DELETE FROM` on this database.

#### dialect

Which database is on the other end.

#### exec

Runs a statement that reads no rows, and returns how many rows it changed.

#### insert_into

Starts an `INSERT INTO` on this database.

#### last_insert_id

The id the last insert wrote. SQLite and MySQL fill this in; Postgres has no such
counter, so ask it for the id with `INSERT ... RETURNING id` instead.

#### new

Wraps a driver. Drivers call this; a program calls the driver's own function.

#### one

Runs a statement and returns its first row, or null when it answered with none.

#### ping

Returns whether the connection still answers.

#### query

Runs a statement and returns its rows, to be read one at a time with `Rows.next`.

#### rollback

Rolls the open transaction back.

#### select

Starts a `SELECT` on this database.

#### transaction

Runs `work` inside a transaction: it commits when the work returns, and rolls back when
it throws.

```valk
db.transaction(fn(tx: sql.Db) !sql.Error {
    tx.exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", .{ 10, 1 }) !>
    tx.exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", .{ 10, 2 }) !>
}) ! panic("%{E.message}")
```

#### update

Starts an `UPDATE` on this database.

#### value

Runs a statement that answers with one value, and returns it. NULL when there is no row.

```valk
let total = (db.value("SELECT count(*) FROM users") ! panic("%{E.message}")).to_int()
```

```js
// What a driver has to provide. The driver packages implement this; a program uses `Db`.
+ interface Driver {
}
```

### Driver

What a driver has to provide. The driver packages implement this; a program uses `Db`.

```js
// One migration: a name and the statements it runs.
+ class Migration {
    // The name, which is also the order they run in. A timestamp in front keeps them in order, as in `0001-users.sql` or `20260918-add-index.sql`.
    + name: String
    // The statements, separated by `;`.
    + sql: String
}
```

### Migration

One migration: a name and the statements it runs.

#### name

The name, which is also the order they run in. A timestamp in front keeps them in order,
as in `0001-users.sql` or `20260918-add-index.sql`.

#### sql

The statements, separated by `;`.

```js
// Runs the migrations a database still needs, and remembers which it has run.
+ class Migrator {
    // The database to migrate.
    + db: Db
    // The migrations, in the order they run.
    ~ migrations: Array[Migration]
    // The table the names of the migrations that ran are kept in.
    + table: String

    // Adds one migration.
    + fn add(name: String, sql: String) Migrator
    // Adds every file of a map of name to contents, sorted by name.
    + fn add_many(files: Map[String]) Migrator
    // Returns the names that have run, oldest first.
    + fn applied() Array[String] !Error
    // Runs every migration that has not run yet, and returns their names.
    + fn migrate() Array[String] !Error
    // Creates a migrator for a database.
    + static fn new(db: Db, table: String ("schema_migrations")) Migrator
    // Returns the migrations that have not run yet, in the order they would run.
    + fn pending() Array[String] !Error
}
```

### Migrator

Runs the migrations a database still needs, and remembers which it has run.

The names that ran are kept in a table (`schema_migrations` by default), so running the
migrator again only runs what is new. Every migration runs inside a transaction, so one that
fails halfway leaves nothing behind on a database with transactional DDL (SQLite and
Postgres; MySQL commits each schema change on its own).

```valk
let migrator = sql.Migrator.new(db)
migrator.add_many(#embed_dir("migrations"))
let done = migrator.migrate() ! panic("Migration failed: %{E.message}")
each done as name : println("ran " + name)
```

#### db

The database to migrate.

#### migrations

The migrations, in the order they run.

#### table

The table the names of the migrations that ran are kept in.

#### add

Adds one migration.

#### add_many

Adds every file of a map of name to contents, sorted by name.

`#embed_dir("migrations")` gives exactly that map, so the files travel inside the
program and no directory has to exist where it runs.

#### applied

Returns the names that have run, oldest first.

#### migrate

Runs every migration that has not run yet, and returns their names.

A migration that fails stops the run and throws; the ones before it stay applied.

#### new

Creates a migrator for a database.

#### pending

Returns the migrations that have not run yet, in the order they would run.

```js
// A set of connections that are opened once and handed out as they are needed.
+ class Pool {
    // Whether `get` checks an idle connection before handing it out, which costs a round trip and catches a connection the database dropped in the meantime.
    + check_on_get: bool
    // How many connections are handed out at this moment.
    ~ in_use: uint
    // The greatest number of connections that may exist at one time, idle and handed out together. 0 is no limit.
    + max_connections: uint
    // How many idle connections are kept. Connections given back beyond this are closed.
    + max_idle: uint
    // Opens a new connection. The pool calls it whenever it needs one.
    + open: fn()(Db !Error)
    // How many connections the pool has: idle plus handed out.
    ~ size: uint
    // How long `get` waits for a connection to come back when the pool is at its limit, in milliseconds. It throws `timeout` after that; 0 waits forever.
    + wait_timeout_ms: uint

    // Closes every idle connection. Connections that are handed out are left alone.
    + fn close_idle() void
    // Takes a connection out of the pool, opening one when none is idle.
    + fn get() Db !Error
    // Creates a pool. No connection is opened until the first `get`.
    + static fn new(open: fn()(Db !Error), max_connections: uint (8), max_idle: uint (4)) Pool
    // Gives a connection back. One beyond `max_idle` is closed instead of kept.
    + fn put(db: Db) void
}
```

### Pool

A set of connections that are opened once and handed out as they are needed.

A pool belongs to one thread, like the connections in it: a server that answers requests on
several worker threads gives every thread its own through a `global`, since every thread
runs the global initializers.

```valk
global pool: sql.Pool (sql.Pool.new(fn() sql.Db !sql.Error {
    return sqlite.database(sqlite.open("app.db") ! throw .connect { message: E.message })
}, 8))

fn handler(req: http.Request) http.Response {
    let db = pool.get() ! return http.Response.text("database down", 503)
    defer pool.put(db)
    ...
}
```

#### check_on_get

Whether `get` checks an idle connection before handing it out, which costs a round trip
and catches a connection the database dropped in the meantime.

#### in_use

How many connections are handed out at this moment.

#### max_connections

The greatest number of connections that may exist at one time, idle and handed out
together. 0 is no limit.

#### max_idle

How many idle connections are kept. Connections given back beyond this are closed.

#### open

Opens a new connection. The pool calls it whenever it needs one.

#### size

How many connections the pool has: idle plus handed out.

#### wait_timeout_ms

How long `get` waits for a connection to come back when the pool is at its limit, in
milliseconds. It throws `timeout` after that; 0 waits forever.

#### close_idle

Closes every idle connection. Connections that are handed out are left alone.

#### get

Takes a connection out of the pool, opening one when none is idle.

Waits when the pool is at `max_connections` and every connection is handed out, and
throws `timeout` when none comes back in time. Give it back with `put`, which belongs in
a `defer` right after the `get`.

#### new

Creates a pool. No connection is opened until the first `get`.

#### put

Gives a connection back. One beyond `max_idle` is closed instead of kept.

```js
// A statement built from its parts, with the values kept apart from the text.
+ class Query {
    // Runs the statement and returns every row.
    + fn all() Array[Map[Value]] !Error
    // Returns the values of the statement, in the order its placeholders take them.
    + fn args() Array[Value]
    // Starts a `DELETE FROM`.
    + static fn delete_from(table: String) Query
    // The table to read from.
    + fn from(table: String) Query
    // Adds a `GROUP BY`.
    + fn group_by(columns: String) Query
    // Adds a `HAVING` condition.
    + fn having(condition: String, args: Array[Value] (.{})) Query
    // Starts an `INSERT INTO`.
    + static fn insert_into(table: String) Query
    // Adds a join, written as it is: `join("JOIN posts ON posts.user_id = users.id")`.
    + fn join(clause: String, args: Array[Value] (.{})) Query
    // Adds a `LIMIT`.
    + fn limit(count: uint) Query
    // Adds an `OFFSET`.
    + fn offset(count: uint) Query
    // Binds the query to a database, so that it can run itself.
    + fn on(db: Db) Query
    // Runs the statement and returns its first row, or null.
    + fn one() ?Map[Value] !Error
    // Adds an `ORDER BY`, written as it is: `order_by("name ASC, id DESC")`.
    + fn order_by(columns: String) Query
    // Adds a `RETURNING`, which SQLite and Postgres have and MySQL does not.
    + fn returning(columns: String) Query
    // Runs the statement and returns how many rows it changed.
    + fn run() uint !Error
    // Starts a `SELECT`.
    + static fn select(columns: String ("*")) Query
    // Sets a column of an `UPDATE`.
    + fn set(column: String, value: Value) Query
    // Sets a column of an `UPDATE` to an expression, such as `balance + ?`.
    + fn set_expression(column: String, expression: String, args: Array[Value] (.{})) Query
    // Sets several columns of an `UPDATE`.
    + fn set_many(values: Map[Value]) Query
    // Writes the statement for `dialect`.
    + fn to_sql(dialect: Dialect (Dialect.sqlite)) String
    // Starts an `UPDATE`.
    + static fn update(table: String) Query
    // Runs the statement and returns the first value of its first row.
    + fn value() Value !Error
    // Adds a row to an `INSERT`. Every row must have the same columns.
    + fn values(row: Map[Value]) Query
    // Adds a condition. Several conditions are joined with `AND`.
    + fn where(condition: String, args: Array[Value] (.{})) Query
    // Adds `column IN (?, ?, …)` with one placeholder per value.
    + fn where_in(column: String, values: Array[Value]) Query
    // Adds `column IS NOT NULL`.
    + fn where_not_null(column: String) Query
    // Adds `column IS NULL`.
    + fn where_null(column: String) Query
}
```

### Query

A statement built from its parts, with the values kept apart from the text.

Conditions are written with `?` placeholders and their values passed next to them, so the
values are always bound. `to_sql` writes the placeholders the dialect wants.

```valk
let rows = db.select("id, name")
    .from("users")
    .where("age > ?", .{ 18 })
    .where("active = ?", .{ true })
    .order_by("name")
    .limit(10)
    .all() ! panic("%{E.message}")
```

#### all

Runs the statement and returns every row.

#### args

Returns the values of the statement, in the order its placeholders take them.

#### delete_from

Starts a `DELETE FROM`.

#### from

The table to read from.

#### group_by

Adds a `GROUP BY`.

#### having

Adds a `HAVING` condition.

#### insert_into

Starts an `INSERT INTO`.

#### join

Adds a join, written as it is: `join("JOIN posts ON posts.user_id = users.id")`.

#### limit

Adds a `LIMIT`.

#### offset

Adds an `OFFSET`.

#### on

Binds the query to a database, so that it can run itself.

#### one

Runs the statement and returns its first row, or null.

#### order_by

Adds an `ORDER BY`, written as it is: `order_by("name ASC, id DESC")`.

#### returning

Adds a `RETURNING`, which SQLite and Postgres have and MySQL does not.

#### run

Runs the statement and returns how many rows it changed.

#### select

Starts a `SELECT`.

#### set

Sets a column of an `UPDATE`.

#### set_expression

Sets a column of an `UPDATE` to an expression, such as `balance + ?`.

#### set_many

Sets several columns of an `UPDATE`.

#### to_sql

Writes the statement for `dialect`.

#### update

Starts an `UPDATE`.

#### value

Runs the statement and returns the first value of its first row.

#### values

Adds a row to an `INSERT`. Every row must have the same columns.

#### where

Adds a condition. Several conditions are joined with `AND`.

Each `?` in the condition takes the next value from `args`.

#### where_in

Adds `column IN (?, ?, …)` with one placeholder per value.

#### where_not_null

Adds `column IS NOT NULL`.

#### where_null

Adds `column IS NULL`.

```js
// The rows of one query, read one at a time.
+ interface Rows {
}
```

### Rows

The rows of one query, read one at a time.

```js
// One value: a cell of a row, or an argument of a statement.
+ struct Value {
    + bool_value: bool
    + float_value: float
    + int_value: int
    // The bytes of a text or blob value.
    + text: String
    // Which of the kinds this value is.
    + type: TYPE

    // Returns whether the value is a blob.
    + fn is_blob() bool
    // Returns whether the value is a real boolean, which only Postgres reports.
    + fn is_bool() bool
    // Returns whether the value is NULL.
    + fn is_null() bool
    // Returns a NULL value.
    + static fn null() Value
    // Returns a text value.
    + static fn of(text: String ("")) Value
    // Returns a blob value; the bytes are stored as they are.
    + static fn of_blob(data: String ("")) Value
    // Returns a bool value. A driver whose database has no boolean type sends it as 1 or 0.
    + static fn of_bool(value: bool (false)) Value
    // Returns a floating point value.
    + static fn of_float(number: float (0)) Value
    // Returns an integer value.
    + static fn of_int(number: int (0)) Value
    // Returns the value as a bool: any number other than 0, and the texts `1`, `t`, `true`, `yes` and `on`.
    + fn to_bool() bool
    // Returns the value as a float; text is parsed, NULL is 0.
    + fn to_float() float
    // Returns the value as an integer; text is parsed, floats are truncated, NULL is 0.
    + fn to_int() int
    // Parses the value as JSON. Returns json null when the text is not valid JSON.
    + fn to_json() Value
    // Returns the value as text; numbers are formatted and NULL becomes "".
    + fn to_string() String
    // Like `to_string`, but null for NULL, which tells an empty value apart from a missing one.
    + fn to_string_or_null() ?String
    // Returns the value as an unsigned integer; negative numbers become 0.
    + fn to_uint() uint
}
```

### Value

One value: a cell of a row, or an argument of a statement.

This is a struct, so a row of values costs one allocation rather than one per column.

#### text

The bytes of a text or blob value.

#### type

Which of the kinds this value is.

#### is_blob

Returns whether the value is a blob.

#### is_bool

Returns whether the value is a real boolean, which only Postgres reports.

#### is_null

Returns whether the value is NULL.

#### null

Returns a NULL value.

#### of

Returns a text value.

#### of_blob

Returns a blob value; the bytes are stored as they are.

#### of_bool

Returns a bool value. A driver whose database has no boolean type sends it as 1 or 0.

#### of_float

Returns a floating point value.

#### of_int

Returns an integer value.

#### to_bool

Returns the value as a bool: any number other than 0, and the texts `1`, `t`, `true`,
`yes` and `on`.

#### to_float

Returns the value as a float; text is parsed, NULL is 0.

#### to_int

Returns the value as an integer; text is parsed, floats are truncated, NULL is 0.

#### to_json

Parses the value as JSON. Returns json null when the text is not valid JSON.

#### to_string

Returns the value as text; numbers are formatted and NULL becomes "".

#### to_string_or_null

Like `to_string`, but null for NULL, which tells an empty value apart from a missing one.

#### to_uint

Returns the value as an unsigned integer; negative numbers become 0.
