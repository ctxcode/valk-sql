
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
// What a value holds. Every database this package speaks to has the first six; a `list` is only ever an argument, which a statement expands into one value per item.
+ enum TYPE { null, bool, int, float, text, blob, list }
```

### Dialect

Which database is on the other end, and so how a statement is written.

### TYPE

What a value holds. Every database this package speaks to has the first six; a `list` is
only ever an argument, which a statement expands into one value per item.

## Functions for 'main'

```js
// Converts any supported value (integers, floats, bools, text, json values, and nullable versions of those) into a `Value`.
+ fn convert(ndata: $T) Value
// Turns the `:name` placeholders of `statement` into the placeholders of `dialect` (`?`, or `$1`, `$2`, ... for Postgres) and returns the values in their order.
+ fn named(statement: String, values: ?Map[Value], dialect: Dialect (Dialect.sqlite)) (String, Array[Value]) !Error
// Reads a row into a class or struct of your own.
+ fn row_to[T](row: Map[Value], bool_columns: ?Array[String] (null)) T !Error
// Returns the row as a JSON object, for an answer that goes straight out as JSON.
+ fn row_to_json_value(row: Map[Value], bool_columns: ?Array[String] (null)) Value
// Reads every row into a class or struct of your own, as `row_to` does for one.
+ fn rows_to[T](rows: Array[Map[Value]], bool_columns: ?Array[String] (null)) Array[T] !Error
```

### convert

Converts any supported value (integers, floats, bools, text, json values, and nullable
versions of those) into a `Value`.

### named

Turns the `:name` placeholders of `statement` into the placeholders of `dialect` (`?`, or
`$1`, `$2`, ... for Postgres) and returns the values in their order.

A name used twice takes its value twice, and a list (an array) becomes one placeholder per
item, so `IN (:ids)` works for any number of ids; an empty list becomes `NULL`, which
matches nothing. Text in quotes is left alone, and so is a Postgres cast such as `::int`.
Throws `syntax` for a name `values` does not hold; values no placeholder names are left out.

`Db` does this for every statement; this is for code that passes a statement on.

```valk
let statement, args = sql.named("SELECT * FROM users WHERE id IN (:ids)", .{ "ids" => ids }) ! panic("%{E.message}")
```

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

let row = db.one("SELECT * FROM users WHERE id = :id", .{ "id" => 1 }) ! panic("%{E.message}")
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
// Conditions joined by `AND` and `OR`, with groups where the two are mixed.
+ class Conditions {
    // Adds a group of conditions with `AND`, written in brackets.
    + fn group(build: fn(Conditions)()) Conditions
    // How many conditions this group holds.
    + get length: uint
    // Adds a group of conditions with `OR`, written in brackets.
    + fn or_group(build: fn(Conditions)()) Conditions
    // The same, joined with `OR`.
    + fn or_where(column: String, operator_or_value: ?Value, value: ?Value (not_given())) Conditions
    // The same with `OR`.
    + fn or_where_in(column: String, values: Array[Value]) Conditions
    // Adds `column IS NOT NULL` with `OR`.
    + fn or_where_not_null(column: String) Conditions
    // Adds `column IS NULL` with `OR`.
    + fn or_where_null(column: String) Conditions
    // The same, joined with `OR`.
    + fn or_where_raw(condition: String, values: ?Map[Value] (null)) Conditions
    // Adds `column = value` with `AND`, or `column <operator> value` when given three arguments.
    + fn where(column: String, operator_or_value: ?Value, value: ?Value (not_given())) Conditions
    // Adds `column IN (...)` with `AND`. An empty list matches nothing.
    + fn where_in(column: String, values: Array[Value]) Conditions
    // Adds `column NOT IN (...)` with `AND`. An empty list matches everything.
    + fn where_not_in(column: String, values: Array[Value]) Conditions
    // Adds `column IS NOT NULL` with `AND`.
    + fn where_not_null(column: String) Conditions
    // Adds `column IS NULL` with `AND`.
    + fn where_null(column: String) Conditions
    // Adds a condition written in SQL, with `AND`, for what `where` cannot say. Values go in by name, as in `Db.exec`.
    + fn where_raw(condition: String, values: ?Map[Value] (null)) Conditions
}
```

### Conditions

Conditions joined by `AND` and `OR`, with groups where the two are mixed.

A condition compares a column with a value, which is always bound, never pasted in. A group is
written in brackets, so what binds to what is never left to precedence:

```valk
query.where("active", true)
query.where_group(fn(w: sql.Conditions) {
    w.where("role", "admin")
    w.or_where("score", ">", 100)
})
// WHERE (active = ?) AND ((role = ?) OR (score > ?))
```

#### group

Adds a group of conditions with `AND`, written in brackets.

#### length

How many conditions this group holds.

#### or_group

Adds a group of conditions with `OR`, written in brackets.

#### or_where

The same, joined with `OR`.

#### or_where_in

The same with `OR`.

#### or_where_not_null

Adds `column IS NOT NULL` with `OR`.

#### or_where_null

Adds `column IS NULL` with `OR`.

#### or_where_raw

The same, joined with `OR`.

#### where

Adds `column = value` with `AND`, or `column <operator> value` when given three
arguments.

The operators are `=`, `!=`, `<>`, `<`, `>`, `<=`, `>=`, `like`, `not like`, `ilike`,
`not ilike`, `in` and `not in`. A null value becomes `IS NULL` (`IS NOT NULL` with `!=`
or `<>`), and a list (an array) becomes `IN (...)`.

```valk
w.where("name", "Ada")
w.where("age", ">=", 18)
w.where("deleted_at", null)          // deleted_at IS NULL
w.where("id", ids)                   // id IN (...)
```

The column is written as it is given, so it may be `users.id`; it must come from the
program, never from the input it is handling.

#### where_in

Adds `column IN (...)` with `AND`. An empty list matches nothing.

#### where_not_in

Adds `column NOT IN (...)` with `AND`. An empty list matches everything.

#### where_not_null

Adds `column IS NOT NULL` with `AND`.

#### where_null

Adds `column IS NULL` with `AND`.

#### where_raw

Adds a condition written in SQL, with `AND`, for what `where` cannot say. Values go in by
name, as in `Db.exec`.

```valk
w.where_raw("lower(email) = :email OR phone = :phone", .{ "email" => email, "phone" => phone })
```

```js
// A database, whichever driver is behind it.
+ class Db {
    // The driver behind this database.
    + driver: Driver

    // Runs a statement and returns every row it answered with.
    + fn all(statement: String, values: ?Map[Value] (null)) Array[Map[Value]] !Error
    // Opens a transaction.
    + fn begin(immediate: bool (false)) void !Error
    // Opens a transaction and hands it back, for work that does not fit in a closure.
    + fn begin_transaction(immediate: bool (false)) Tx !Error
    // Closes the connection.
    + fn close() void
    // Commits the open transaction.
    + fn commit() void !Error
    // Starts a `DELETE FROM` on this database.
    + fn delete_from(table: String) Query
    // Which database is on the other end.
    + get dialect: Dialect
    // Walks the rows of a statement one at a time, without holding them all in memory.
    + fn each_row(statement: String, values: ?Map[Value], handler: fn(Map[Value])(bool !Error)) uint !Error
    // Runs a statement that reads no rows, and returns how many rows it changed.
    + fn exec(statement: String, values: ?Map[Value] (null)) uint !Error
    // Starts an `INSERT INTO` on this database.
    + fn insert_into(table: String) Query
    // The id the last insert wrote. SQLite and MySQL fill this in; Postgres has no such counter, so ask it for the id with `INSERT ... RETURNING id` instead.
    + fn last_insert_id() int
    // Wraps a driver. Drivers call this; a program calls the driver's own function.
    + static fn new(driver: Driver) Db
    // Runs a statement and returns its first row, or null when it answered with none.
    + fn one(statement: String, values: ?Map[Value] (null)) ?Map[Value] !Error
    // Returns whether the connection still answers.
    + fn ping() bool
    // Runs a statement and returns its rows, to be read one at a time with `Rows.next`.
    + fn query(statement: String, values: ?Map[Value] (null)) Rows !Error
    // Rolls the open transaction back.
    + fn rollback() void !Error
    // Starts a `SELECT` on this database.
    + fn select(columns: String ("*")) Query
    // Runs `work` inside a transaction: it commits when the work returns, and rolls back when it throws.
    + fn transaction(work: fn(Db)(!Error), immediate: bool (false)) void !Error
    // Starts an `UPDATE` on this database.
    + fn update(table: String) Query
    // Runs a statement that answers with one value, and returns it. NULL when there is no row.
    + fn value(statement: String, values: ?Map[Value] (null)) Value !Error
}
```

### Db

A database, whichever driver is behind it.

A driver package hands one of these out (`sqlite.database(con)`, `postgres.database(con)`,
`mysql.database(con)`), and everything in this package works with it. Values go into a
statement by name, `:name`, whatever the database is; they are bound, never pasted in. An
array becomes a list, so `IN (:ids)` takes any number of ids.

```valk
let db = sqlite.database(con)
db.exec("INSERT INTO users (name, age) VALUES (:name, :age)", .{ "name" => "Ada", "age" => 36 }) ! panic("%{E.message}")
let rows = db.all("SELECT * FROM users WHERE age > :age", .{ "age" => 18 }) ! panic("%{E.message}")
```

#### driver

The driver behind this database.

#### all

Runs a statement and returns every row it answered with.

#### begin

Opens a transaction.

`immediate` takes the write lock at once where the database has one (SQLite), which a
transaction that reads a value and writes it back wants.

#### begin_transaction

Opens a transaction and hands it back, for work that does not fit in a closure.

Put `defer tx.close()` right after it: the transaction then rolls back on every way out
of the function that is not a `commit`.

#### close

Closes the connection.

#### commit

Commits the open transaction.

#### delete_from

Starts a `DELETE FROM` on this database.

#### dialect

Which database is on the other end.

#### each_row

Walks the rows of a statement one at a time, without holding them all in memory.

The handler is given each row and returns whether to carry on, so it can stop early.
Returns how many rows it saw.

The rows arrive from the connection as they are read, so **nothing else may run on that
connection while the walk is going**: a statement sent in the middle of it ends the walk.
A walk that has to write belongs in `chunk` or `chunk_by_id`, which read a page at a time
and leave the connection free in between.

```valk
db.each_row("SELECT id, email FROM users", null, fn(row: Map[sql.Value]) bool !sql.Error {
    println((row.get("email") !? sql.Value.null()).to_string())
    return true
}) ! panic("%{E.message}")
```

#### exec

Runs a statement that reads no rows, and returns how many rows it changed.

`values` fill the `:name` placeholders; see `named` for the rules.

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
    tx.exec("UPDATE accounts SET balance = balance - :amount WHERE id = :id", .{ "amount" => 10, "id" => 1 }) !>
    tx.exec("UPDATE accounts SET balance = balance + :amount WHERE id = :id", .{ "amount" => 10, "id" => 2 }) !>
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
    // Opens a transaction; `immediate` takes the write lock at once where that exists.
    + fn begin(immediate: bool) void !Error
    // Closes the connection.
    + fn close() void
    + fn commit() void !Error
    // Which database this is.
    + get dialect: Dialect
    // Runs a statement that reads no rows and returns how many rows it changed.
    + fn exec(sql: String, args: Array[Value]) uint !Error
    // The id the last insert wrote, where the database has one.
    + fn last_insert_id() int
    // Returns whether the connection still answers.
    + fn ping() bool
    // Runs a statement and returns its rows.
    + fn query(sql: String, args: Array[Value]) Rows !Error
    + fn rollback() void !Error
}
```

### Driver

What a driver has to provide. The driver packages implement this; a program uses `Db`.

#### begin

Opens a transaction; `immediate` takes the write lock at once where that exists.

#### close

Closes the connection.

#### dialect

Which database this is.

#### exec

Runs a statement that reads no rows and returns how many rows it changed.

#### last_insert_id

The id the last insert wrote, where the database has one.

#### ping

Returns whether the connection still answers.

#### query

Runs a statement and returns its rows.

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
    // Runs the query a page at a time, handing each page to `handler`.
    + fn chunk(size: uint, handler: fn(Array[Map[Value]])(bool !Error)) uint !Error
    // Runs the query a page at a time, walking forward by the last value of `column`.
    + fn chunk_by_id(column: String, size: uint, handler: fn(Array[Map[Value]])(bool !Error)) uint !Error
    // Runs the same query as a count, without its order, limit and offset.
    + fn count(expression: String ("*")) uint !Error
    // Starts a `DELETE FROM`.
    + static fn delete_from(table: String) Query
    // The table to read from.
    + fn from(table: String) Query
    // Adds a `GROUP BY`.
    + fn group_by(columns: String) Query
    // Adds a `HAVING` condition with `AND`, as `where` does: `having("count(*)", ">", 5)`.
    + fn having(column: String, operator_or_value: ?Value, value: ?Value (not_given())) Query
    // Adds a group of `HAVING` conditions, written in brackets.
    + fn having_group(build: fn(Conditions)()) Query
    // Adds a `HAVING` condition written in SQL, with `AND`; values go in by name.
    + fn having_raw(condition: String, values: ?Map[Value] (null)) Query
    // Adds `INNER JOIN <table> ON <condition>`.
    + fn inner_join(table: String, condition: String, values: ?Map[Value] (null)) Query
    // Starts an `INSERT INTO`.
    + static fn insert_into(table: String) Query
    // Adds a join, written as it is: `join("LEFT JOIN posts ON posts.user_id = users.id")`. Values go in by name, as in `Db.exec`.
    + fn join(clause: String, values: ?Map[Value] (null)) Query
    // Adds `JOIN <table> ON <condition>`.
    + fn join_on(table: String, condition: String, values: ?Map[Value] (null)) Query
    // Adds `LEFT JOIN <table> ON <condition>`, which keeps the rows that match nothing.
    + fn left_join(table: String, condition: String, values: ?Map[Value] (null)) Query
    // Adds a `LIMIT`.
    + fn limit(count: uint) Query
    // Adds an `OFFSET`.
    + fn offset(count: uint) Query
    // Binds the query to a database, so that it can run itself.
    + fn on(db: Db) Query
    // On a row that clashes with one that is already there, keeps the row that is there.
    + fn on_conflict_nothing(columns: ?Array[String] (null)) Query
    // On a row that clashes with one that is already there, writes the new values over it.
    + fn on_conflict_update(columns: Array[String], update_columns: ?Array[String] (null)) Query
    // Runs the statement and returns its first row, or null.
    + fn one() ?Map[Value] !Error
    // The same, joined with `OR`.
    + fn or_having(column: String, operator_or_value: ?Value, value: ?Value (not_given())) Query
    // The same, joined with `OR`.
    + fn or_having_raw(condition: String, values: ?Map[Value] (null)) Query
    // The same, joined with `OR`.
    + fn or_where(column: String, operator_or_value: ?Value, value: ?Value (not_given())) Query
    // Adds a group of conditions with `OR`, written in brackets.
    + fn or_where_group(build: fn(Conditions)()) Query
    // The same, joined with `OR`.
    + fn or_where_raw(condition: String, values: ?Map[Value] (null)) Query
    // Adds an `ORDER BY`, written as it is: `order_by("name ASC, id DESC")`.
    + fn order_by(columns: String) Query
    // Takes one page of rows: page 1 is the first `per_page` rows, page 2 the next, and so on.
    + fn page(number: uint, per_page: uint) Query
    // Adds a `RETURNING`, which SQLite and Postgres have and MySQL does not.
    + fn returning(columns: String) Query
    // Runs the statement and returns how many rows it changed.
    + fn run() uint !Error
    // Starts a `SELECT`.
    + static fn select(columns: String ("*")) Query
    // Sets a column of an `UPDATE`.
    + fn set(column: String, value: Value) Query
    // Sets a column of an `UPDATE` to an expression, with values by name.
    + fn set_expression(column: String, expression: String, values: ?Map[Value] (null)) Query
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
    // Adds `column = value` with `AND`, or `column <operator> value` with three arguments; see `Conditions.where` for the operators, null and lists.
    + fn where(column: String, operator_or_value: ?Value, value: ?Value (not_given())) Query
    // Adds a group of conditions with `AND`, written in brackets, for a query that mixes `AND` and `OR`.
    + fn where_group(build: fn(Conditions)()) Query
    // Adds `column IN (...)` with one placeholder per value. An empty list matches nothing.
    + fn where_in(column: String, values: Array[Value]) Query
    // Adds `column NOT IN (...)`. An empty list matches everything.
    + fn where_not_in(column: String, values: Array[Value]) Query
    // Adds `column IS NOT NULL`.
    + fn where_not_null(column: String) Query
    // Adds `column IS NULL`.
    + fn where_null(column: String) Query
    // Adds a condition written in SQL, with `AND`; values go in by name.
    + fn where_raw(condition: String, values: ?Map[Value] (null)) Query
}
```

### Query

A statement built from its parts, with the values kept apart from the text.

Values are always bound, never pasted in; `to_sql` writes the placeholders the dialect
wants. A mistake in a condition, such as an unknown operator, is thrown as `syntax` when the
query runs.

```valk
let rows = db.select("id, name")
    .from("users")
    .where("age", ">", 18)
    .where("active", true)
    .order_by("name")
    .limit(10)
    .all() ! panic("%{E.message}")
```

#### all

Runs the statement and returns every row.

#### args

Returns the values of the statement, in the order its placeholders take them.

#### chunk

Runs the query a page at a time, handing each page to `handler`.

Every page is a query of its own, with a `LIMIT` and an `OFFSET`, so the connection is
free between pages: the handler may write, and may use the same database. Returns how
many rows were handed over in total, and stops early when the handler returns false.

The query needs an `order_by` to mean anything, since a database is free to return rows
in another order each time. Rows that are inserted or deleted while this runs shift the
pages under it, and a deep offset gets slower the further it goes: `chunk_by_id` has
neither problem and is the one to use on a table that is being written.

```valk
db.select().from("users").order_by("id").chunk(500, fn(rows: Array[Map[sql.Value]]) bool !sql.Error {
    each rows as row : send_email(row)
    return true
}) ! panic("%{E.message}")
```

#### chunk_by_id

Runs the query a page at a time, walking forward by the last value of `column`.

Each page asks for the rows after the last one of the page before it
(`WHERE column > last ORDER BY column LIMIT size`), which is what makes this safe on a
table that is being written: no row is skipped or seen twice because rows were inserted
or deleted in between, and the database can use the index on `column` instead of counting
its way to a deep offset.

`column` must be unique and never decrease — a primary key or a `uuid.v7()` is exactly
that. The order of the query is set by this method; any `order_by` on it is replaced.

```valk
db.select().from("users").chunk_by_id("id", 500, fn(rows: Array[Map[sql.Value]]) bool !sql.Error {
    each rows as row {
        db.exec("UPDATE users SET checked = 1 WHERE id = :id", .{ "id" => row.get("id") !? sql.Value.null() }) !>
    }
    return true
}) ! panic("%{E.message}")
```

#### count

Runs the same query as a count, without its order, limit and offset.

This is the count that belongs next to a page of rows: the table, the joins and the
conditions are the ones of this query.

#### delete_from

Starts a `DELETE FROM`.

#### from

The table to read from.

#### group_by

Adds a `GROUP BY`.

#### having

Adds a `HAVING` condition with `AND`, as `where` does: `having("count(*)", ">", 5)`.

#### having_group

Adds a group of `HAVING` conditions, written in brackets.

#### having_raw

Adds a `HAVING` condition written in SQL, with `AND`; values go in by name.

#### inner_join

Adds `INNER JOIN <table> ON <condition>`.

#### insert_into

Starts an `INSERT INTO`.

#### join

Adds a join, written as it is: `join("LEFT JOIN posts ON posts.user_id = users.id")`.
Values go in by name, as in `Db.exec`.

#### join_on

Adds `JOIN <table> ON <condition>`.

```valk
query.join_on("posts", "posts.user_id = users.id")
```

#### left_join

Adds `LEFT JOIN <table> ON <condition>`, which keeps the rows that match nothing.

#### limit

Adds a `LIMIT`.

#### offset

Adds an `OFFSET`.

#### on

Binds the query to a database, so that it can run itself.

#### on_conflict_nothing

On a row that clashes with one that is already there, keeps the row that is there.

#### on_conflict_update

On a row that clashes with one that is already there, writes the new values over it.

`columns` are the ones that decide what a clash is, which SQLite and Postgres need (the
columns of the unique index); MySQL finds that out by itself and ignores them.
`update_columns` are the ones to overwrite, and empty means every column of the insert.

```valk
db.insert_into("counters")
    .values(.{ "name" => "visits", "n" => 1 })
    .on_conflict_update(.{ "name" }, .{ "n" })
    .run() ! panic("%{E.message}")
```

#### one

Runs the statement and returns its first row, or null.

#### or_having

The same, joined with `OR`.

#### or_having_raw

The same, joined with `OR`.

#### or_where

The same, joined with `OR`.

```valk
query.where("role", "admin").or_where("score", ">", 100)
// WHERE (role = ?) OR (score > ?)
```

#### or_where_group

Adds a group of conditions with `OR`, written in brackets.

#### or_where_raw

The same, joined with `OR`.

#### order_by

Adds an `ORDER BY`, written as it is: `order_by("name ASC, id DESC")`.

#### page

Takes one page of rows: page 1 is the first `per_page` rows, page 2 the next, and so on.

```valk
let rows = db.select().from("posts").order_by("id").page(2, 20).all() ! panic("%{E.message}")
```

#### returning

Adds a `RETURNING`, which SQLite and Postgres have and MySQL does not.

#### run

Runs the statement and returns how many rows it changed.

#### select

Starts a `SELECT`.

#### set

Sets a column of an `UPDATE`.

#### set_expression

Sets a column of an `UPDATE` to an expression, with values by name.

```valk
query.set_expression("balance", "balance + :amount", .{ "amount" => 10 })
```

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

Adds `column = value` with `AND`, or `column <operator> value` with three arguments; see
`Conditions.where` for the operators, null and lists.

```valk
query.where("role", "admin")
query.where("age", ">=", 18)
query.where("id", ids)              // id IN (...)
```

#### where_group

Adds a group of conditions with `AND`, written in brackets, for a query that mixes `AND`
and `OR`.

```valk
query.where("active", true)
query.where_group(fn(w: sql.Conditions) {
    w.where("role", "admin")
    w.or_where("score", ">", 100)
})
// WHERE (active = ?) AND ((role = ?) OR (score > ?))
```

#### where_in

Adds `column IN (...)` with one placeholder per value. An empty list matches nothing.

#### where_not_in

Adds `column NOT IN (...)`. An empty list matches everything.

#### where_not_null

Adds `column IS NOT NULL`.

#### where_null

Adds `column IS NULL`.

#### where_raw

Adds a condition written in SQL, with `AND`; values go in by name.

```valk
query.where_raw("lower(email) = :email", .{ "email" => email })
```

```js
// The rows of one query, read one at a time.
+ interface Rows {
    // Releases what the query still holds. Reading to the end does this as well.
    + fn close() void
    // Reads the next row into `row` and returns whether there was one. The map is cleared first, so one map can serve a whole result.
    + fn next(row: Map[Value]) bool !Error
}
```

### Rows

The rows of one query, read one at a time.

#### close

Releases what the query still holds. Reading to the end does this as well.

#### next

Reads the next row into `row` and returns whether there was one. The map is cleared
first, so one map can serve a whole result.

```js
// A transaction that is not held by a closure.
+ class Tx {
    // The database this transaction runs on. Statements may go through it as well.
    + db: Db
    // Whether the transaction is still open: neither committed nor rolled back.
    ~ open: bool

    // Runs a statement and returns every row.
    + fn all(statement: String, values: ?Map[Value] (null)) Array[Map[Value]] !Error
    // Rolls the work back unless it was committed. Made for `defer`, so it throws nothing.
    + fn close() void
    // Commits the work. The transaction is closed afterwards.
    + fn commit() void !Error
    // Starts a `DELETE FROM` on this transaction.
    + fn delete_from(table: String) Query
    // Runs a statement that reads no rows, and returns how many rows it changed.
    + fn exec(statement: String, values: ?Map[Value] (null)) uint !Error
    // Starts an `INSERT INTO` on this transaction.
    + fn insert_into(table: String) Query
    // The id the last insert wrote, where the database has one.
    + fn last_insert_id() int
    // Runs a statement and returns its first row, or null.
    + fn one(statement: String, values: ?Map[Value] (null)) ?Map[Value] !Error
    // Runs a statement and returns its rows, to be read one at a time.
    + fn query(statement: String, values: ?Map[Value] (null)) Rows !Error
    // Rolls the work back. The transaction is closed afterwards.
    + fn rollback() void !Error
    // Starts a `SELECT` on this transaction.
    + fn select(columns: String ("*")) Query
    // Starts an `UPDATE` on this transaction.
    + fn update(table: String) Query
    // Runs a statement that answers with one value, and returns it.
    + fn value(statement: String, values: ?Map[Value] (null)) Value !Error
}
```

### Tx

A transaction that is not held by a closure.

`close` rolls back unless the work was committed, so a `defer` right after the `begin` makes
every way out of the function safe: an early return, a thrown error, a panic in the middle.

```valk
let tx = db.begin_transaction() ! panic("%{E.message}")
defer tx.close()

tx.exec("UPDATE accounts SET balance = balance - :amount WHERE id = :id", .{ "amount" => amount, "id" => from }) !>
if (tx.value("SELECT balance FROM accounts WHERE id = :id", .{ "id" => from }) !>).to_int() < 0 {
    return "not enough money"      // the defer rolls it back
}
tx.exec("UPDATE accounts SET balance = balance + :amount WHERE id = :id", .{ "amount" => amount, "id" => to }) !>
tx.commit() !>
```

`Db.transaction` does the same with a closure, for work that fits in one.

#### db

The database this transaction runs on. Statements may go through it as well.

#### open

Whether the transaction is still open: neither committed nor rolled back.

#### all

Runs a statement and returns every row.

#### close

Rolls the work back unless it was committed. Made for `defer`, so it throws nothing.

#### commit

Commits the work. The transaction is closed afterwards.

#### delete_from

Starts a `DELETE FROM` on this transaction.

#### exec

Runs a statement that reads no rows, and returns how many rows it changed.

#### insert_into

Starts an `INSERT INTO` on this transaction.

#### last_insert_id

The id the last insert wrote, where the database has one.

#### one

Runs a statement and returns its first row, or null.

#### query

Runs a statement and returns its rows, to be read one at a time.

#### rollback

Rolls the work back. The transaction is closed afterwards.

#### select

Starts a `SELECT` on this transaction.

#### update

Starts an `UPDATE` on this transaction.

#### value

Runs a statement that answers with one value, and returns it.

```js
// One value: a cell of a row, or an argument of a statement.
+ struct Value {
    + bool_value: bool
    + float_value: float
    + int_value: int
    // The items of a list.
    + items: ?Array[Value]
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

#### items

The items of a list.

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
