
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown by every operation of this package, and by the drivers behind it.
+ error Error (connect, syntax, constraint, busy, readonly, timeout, closed, error) payload { message: String, driver_code: String (""), sql: String ("") }
```

## Enums for 'main'

```js
// Which database is on the other end, and so how a statement is written.
+ enum Dialect { sqlite, mysql, postgres }
// What a value holds. Every database this package speaks to has these five.
+ enum TYPE { null, bool, int, float, text, blob }
```

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
    // Adds a condition with `OR`.
    + fn or_where(condition: String, args: Array[Value] (.{})) Conditions
    // The same with `OR`.
    + fn or_where_in(column: String, values: Array[Value]) Conditions
    // Adds `column IS NOT NULL` with `OR`.
    + fn or_where_not_null(column: String) Conditions
    // Adds `column IS NULL` with `OR`.
    + fn or_where_null(column: String) Conditions
    // Adds a condition with `AND`.
    + fn where(condition: String, args: Array[Value] (.{})) Conditions
    // Adds `column IN (?, ?, …)` with `AND`. An empty list matches nothing.
    + fn where_in(column: String, values: Array[Value]) Conditions
    // Adds `column NOT IN (?, ?, …)` with `AND`. An empty list matches everything.
    + fn where_not_in(column: String, values: Array[Value]) Conditions
    // Adds `column IS NOT NULL` with `AND`.
    + fn where_not_null(column: String) Conditions
    // Adds `column IS NULL` with `AND`.
    + fn where_null(column: String) Conditions
}
```

```js
// A database, whichever driver is behind it.
+ class Db {
    // The driver behind this database.
    + driver: Driver

    // Runs a statement and returns every row it answered with.
    + fn all(sql: String, args: Array[Value] (.{})) Array[Map[Value]] !Error
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
    + fn each_row(sql: String, args: Array[Value] (.{}), handler: fn(Map[Value])(bool !Error)) uint !Error
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

```js
// One migration: a name and the statements it runs.
+ class Migration {
    // The name, which is also the order they run in. A timestamp in front keeps them in order, as in `0001-users.sql` or `20260918-add-index.sql`.
    + name: String
    // The statements, separated by `;`.
    + sql: String
}
```

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
    // Adds a `HAVING` condition with `AND`.
    + fn having(condition: String, args: Array[Value] (.{})) Query
    // Adds a group of `HAVING` conditions, written in brackets.
    + fn having_group(build: fn(Conditions)()) Query
    // Adds `INNER JOIN <table> ON <condition>`.
    + fn inner_join(table: String, condition: String, args: Array[Value] (.{})) Query
    // Starts an `INSERT INTO`.
    + static fn insert_into(table: String) Query
    // Adds a join, written as it is: `join("LEFT JOIN posts ON posts.user_id = users.id")`.
    + fn join(clause: String, args: Array[Value] (.{})) Query
    // Adds `JOIN <table> ON <condition>`.
    + fn join_on(table: String, condition: String, args: Array[Value] (.{})) Query
    // Adds `LEFT JOIN <table> ON <condition>`, which keeps the rows that match nothing.
    + fn left_join(table: String, condition: String, args: Array[Value] (.{})) Query
    // Adds a `LIMIT`.
    + fn limit(count: uint) Query
    // Adds an `OFFSET`.
    + fn offset(count: uint) Query
    // Binds the query to a database, so that it can run itself.
    + fn on(db: Db) Query
    // On a row that clashes with one that is already there, keeps the row that is there.
    + fn on_conflict_nothing(columns: Array[String] (.{})) Query
    // On a row that clashes with one that is already there, writes the new values over it.
    + fn on_conflict_update(columns: Array[String], update_columns: Array[String] (.{})) Query
    // Runs the statement and returns its first row, or null.
    + fn one() ?Map[Value] !Error
    // Adds a `HAVING` condition with `OR`.
    + fn or_having(condition: String, args: Array[Value] (.{})) Query
    // Adds a condition with `OR`.
    + fn or_where(condition: String, args: Array[Value] (.{})) Query
    // Adds a group of conditions with `OR`, written in brackets.
    + fn or_where_group(build: fn(Conditions)()) Query
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
    // Adds a condition with `AND`.
    + fn where(condition: String, args: Array[Value] (.{})) Query
    // Adds a group of conditions with `AND`, written in brackets, for a query that mixes `AND` and `OR`.
    + fn where_group(build: fn(Conditions)()) Query
    // Adds `column IN (?, ?, …)` with one placeholder per value. An empty list matches nothing.
    + fn where_in(column: String, values: Array[Value]) Query
    // Adds `column NOT IN (?, ?, …)`. An empty list matches everything.
    + fn where_not_in(column: String, values: Array[Value]) Query
    // Adds `column IS NOT NULL`.
    + fn where_not_null(column: String) Query
    // Adds `column IS NULL`.
    + fn where_null(column: String) Query
}
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

```js
// A transaction that is not held by a closure.
+ class Tx {
    // The database this transaction runs on. Statements may go through it as well.
    + db: Db
    // Whether the transaction is still open: neither committed nor rolled back.
    ~ open: bool

    // Runs a statement and returns every row.
    + fn all(sql: String, args: Array[Value] (.{})) Array[Map[Value]] !Error
    // Rolls the work back unless it was committed. Made for `defer`, so it throws nothing.
    + fn close() void
    // Commits the work. The transaction is closed afterwards.
    + fn commit() void !Error
    // Starts a `DELETE FROM` on this transaction.
    + fn delete_from(table: String) Query
    // Runs a statement that reads no rows, and returns how many rows it changed.
    + fn exec(sql: String, args: Array[Value] (.{})) uint !Error
    // Starts an `INSERT INTO` on this transaction.
    + fn insert_into(table: String) Query
    // The id the last insert wrote, where the database has one.
    + fn last_insert_id() int
    // Runs a statement and returns its first row, or null.
    + fn one(sql: String, args: Array[Value] (.{})) ?Map[Value] !Error
    // Runs a statement and returns its rows, to be read one at a time.
    + fn query(sql: String, args: Array[Value] (.{})) Rows !Error
    // Rolls the work back. The transaction is closed afterwards.
    + fn rollback() void !Error
    // Starts a `SELECT` on this transaction.
    + fn select(columns: String ("*")) Query
    // Starts an `UPDATE` on this transaction.
    + fn update(table: String) Query
    // Runs a statement that answers with one value, and returns it.
    + fn value(sql: String, args: Array[Value] (.{})) Value !Error
}
```

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
