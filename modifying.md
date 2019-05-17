---
layout: article
title: Modifying Data and Using Transactions
---

Now we're ready to see how to modify data and work with transactions. The
distinction might seem artificial if you're used to programming languages that
use a "statement" object for fetching rows as well as updating data, but in Go,
there's an important reason for the difference.

Statements that Modify Data
===========================

Use `Exec()`, preferably with a prepared statement, to accomplish an `INSERT`,
`UPDATE`, `DELETE`, or another statement that doesn't return rows. The following
example shows how to insert a row and inspect metadata about the operation:

<pre class="prettyprint lang-go">
stmt, err := db.Prepare("INSERT INTO users(name) VALUES(?)")
if err != nil {
	log.Fatal(err)
}
res, err := stmt.Exec("Dolly")
if err != nil {
	log.Fatal(err)
}
lastId, err := res.LastInsertId()
if err != nil {
	log.Fatal(err)
}
rowCnt, err := res.RowsAffected()
if err != nil {
	log.Fatal(err)
}
log.Printf("ID = %d, affected = %d\n", lastId, rowCnt)
</pre>

Executing the statement produces a `sql.Result` that gives access to statement
metadata: the last inserted ID and the number of rows affected.

What if you don't care about the result? What if you just want to execute a
statement and check if there were any errors, but ignore the result? Wouldn't
the following two statements do the same thing?

<pre class="prettyprint lang-go">
_, err := db.Exec("DELETE FROM users")  // OK
_, err := db.Query("DELETE FROM users") // BAD
</pre>

The answer is no. They do **not** do the same thing, and **you should never use
`Query()` like this.** The `Query()` will return a `sql.Rows`, which reserves a
database connection until the `sql.Rows` is closed.
Since there might be unread data (e.g. more data rows), the connection can not
be used. In the example above, the connection will *never* be released again.
The garbage collector will eventually close the underlying `net.Conn` for you,
but this might take a long time. Moreover the database/sql package keeps
tracking the connection in its pool, hoping that you release it at some point,
so that the connection can be used again.
This anti-pattern is therefore a good way to run out of resources (too many
connections, for example).

Working with Transactions
=========================

In Go, a transaction is essentially an object that reserves a connection to the
datastore. It lets you do all of the operations we've seen thus far, but
guarantees that they'll be executed on the same connection.

在Go中，事务本质上是一个保留与数据存储区连接的对象。它允许您执行我们迄今为止看到的所有操作，但保证它们将在同一连接上执行。

You begin a transaction with a call to `db.Begin()`, and close it with a
`Commit()` or `Rollback()` method on the resulting `Tx` variable. Under the
covers, the `Tx` gets a connection from the pool, and reserves it for use only
with that transaction. The methods on the `Tx` map one-for-one to methods you
can call on the database itself, such as `Query()` and so forth.

您通过调用 `db.Begin()` 开始事务，并在生成的Tx变量上使用 `Commit()` 或 `Rollback()` 方法将其关闭。在封面下，`Tx` 从池中获取连接，并保留它仅用于该事务。 Tx上的方法可以一对一地映射到可以在数据库本身上调用的方法，例如 `Query()` 等。

Prepared statements that are created in a transaction are bound exclusively to
that transaction. See [prepared statements](prepared.html) for more.

在事务中创建的预准备语句专门绑定到该事务。有关更多信息，请参

You should not mingle the use of transaction-related functions such as `Begin()`
and `Commit()` with SQL statements such as `BEGIN` and `COMMIT` in your SQL
code. Bad things might result:

您不应该在SQL代码中使用与事务相关的函数（如Begin（）和Commit（））与SQL语句（如BEGIN和COMMIT）混合使用。可能导致不好的事情：

* The `Tx` objects could remain open, reserving a connection from the pool and not returning it.  
  Tx对象可以保持打开状态，保留池中的连接而不返回它。
* The state of the database could get out of sync with the state of the Go variables representing it.  
  数据库的状态可能与表示它的Go变量的状态不同步。
* You could believe you're executing queries on a single connection, inside of a transaction, when in reality Go has created several connections for you invisibly and some statements aren't part of the transaction.  
  你可以相信你在一个事务中的单个连接上执行查询，而实际上Go已经为你创建了几个连接而且一些语句不是事务的一部分。

While you are working inside a transaction you should be careful not to make
calls to the `db` variable. Make all of your calls to the `Tx` variable that you
created with `db.Begin()`. `db` is not in a transaction, only the `Tx` object is.
If you make further calls to `db.Exec()` or similar, those will happen outside
the scope of your transaction, on other connections.

If you need to work with multiple statements that modify connection state, you
need a `Tx` even if you don't want a transaction per se. For example:

* Creating temporary tables, which are only visible to one connection.
* Setting variables, such as MySQL's `SET @var := somevalue` syntax.
* Changing connection options, such as character sets or timeouts.

If you need to do any of these things, you need to bind your activity to a
single connection, and the only way to do that in Go is to use a `Tx`.

**Previous: [Retrieving Result Sets](retrieving.html)**
**Next: [Using Prepared Statements](prepared.html)**
