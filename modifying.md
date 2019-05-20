---
layout: article
title: Modifying Data and Using Transactions
---

Now we're ready to see how to modify data and work with transactions. The
distinction might seem artificial if you're used to programming languages that
use a "statement" object for fetching rows as well as updating data, but in Go,
there's an important reason for the difference.

现在我们已经准备好了解如何修改数据和处理事务。如果你习惯于编写使用“语句”对象来获取行以及更新数据的语言，那么区别似乎是假的，但在Go中，存在差异的重要原因。

Statements that Modify Data
===========================

Use `Exec()`, preferably with a prepared statement, to accomplish an `INSERT`,
`UPDATE`, `DELETE`, or another statement that doesn't return rows. The following
example shows how to insert a row and inspect metadata about the operation:

使用Exec（），最好使用预准备语句来完成INSERT，UPDATE，DELETE或其他不返回行的语句。以下示例显示如何插入行并检查有关操作的元数据：

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

执行该语句会生成一个sql.Result，它提供对语句元数据的访问：最后插入的ID和受影响的行数。

What if you don't care about the result? What if you just want to execute a
statement and check if there were any errors, but ignore the result? Wouldn't
the following two statements do the same thing?

如果你不关心结果怎么办？如果您只想执行一个语句并检查是否有错误，但忽略结果怎么办？以下两个陈述不会做同样的事情吗？

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

答案是不。他们不做同样的事情，你不应该像这样使用Query（）。 Query（）将返回一个sql.Rows，它保留数据库连接，直到sql.Rows关闭。由于可能存在未读数据（例如，更多数据行），因此不能使用该连接。在上面的示例中，永远不会再次释放连接。垃圾收集器最终将为您关闭底层的net.Conn，但这可能需要很长时间。此外，database / sql包会一直跟踪其池中的连接，希望您在某个时刻释放它，以便可以再次使用该连接。因此，这种反模式是耗尽资源的好方法（例如，连接太多）。

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

当您在事务内部工作时，应该注意不要调用db变量。对使用db.Begin（）创建的Tx变量进行所有调用。 db不在事务中，只有Tx对象。如果进一步调用db.Exec（）或类似函数，那么这些调用将发生在事务范围之外，在其他连接上。

If you need to work with multiple statements that modify connection state, you
need a `Tx` even if you don't want a transaction per se. For example:

如果您需要使用多个修改连接状态的语句，即使您不想要事务本身，也需要Tx。例如：

* Creating temporary tables, which are only visible to one connection.  
  创建仅对一个连接可见的临时表。
* Setting variables, such as MySQL's `SET @var := somevalue` syntax.  
  设置变量，例如MySQL的SET @var：= somevalue语法。
* Changing connection options, such as character sets or timeouts.  
  更改连接选项，例如字符集或超时。

If you need to do any of these things, you need to bind your activity to a
single connection, and the only way to do that in Go is to use a `Tx`.

如果您需要执行上述任何操作，则需要将活动绑定到单个连接，而在Go中执行此操作的唯一方法是使用Tx。

**Previous: [Retrieving Result Sets](retrieving.html)**
**Next: [Using Prepared Statements](prepared.html)**
