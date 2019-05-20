---
layout: article
title: Using Prepared Statements
---

使用准备好的陈述

Prepared statements have all the usual benefits in Go: security, efficiency,
convenience. But the way they're implemented is a little different from what
you might be used to, especially with regards to how they interact with some of
the internals of `database/sql`.

准备好的陈述在Go中具有所有常见的好处：安全性，效率和便利性。但是它们的实现方式与您可能习惯的方式略有不同，特别是关于它们如何与 `database/sql` 的某些内部进行交互。

Prepared Statements And Connections
===================================

准备好的陈述和连接

At the database level, a prepared statement is bound to a single database
connection. The typical flow is that the client sends a SQL statement with
placeholders to the server for preparation, the server responds with a statement
ID, and then the client executes the statement by sending its ID and parameters.

在数据库级别，预准备语句绑定到单个数据库连接。典型的流程是客户端将带有占位符的SQL语句发送到服务器进行准备，服务器使用语句ID进行响应，然后客户端通过发送其ID和参数来执行该语句。

In Go, however, connections are not exposed directly to the user of the
`database/sql` package. You don't prepare a statement on a connection. You
prepare it on a `DB` or a `Tx`. And `database/sql` has some convenience
behaviors such as automatic retries. For these reasons, the underlying
association between prepared statements and connections, which exists at the
driver level, is hidden from your code.

但是，在Go中，连接不会直接暴露给 `database/sql` 包的用户。您没有在连接上准备语句。您在 `DB` 或 `Tx` 上准备它。并且 `database/sql` 具有一些便利行为，例如自动重试。由于这些原因，在驱动程序级别存在的预准备语句和连接之间的底层关联对代码是隐藏的。

Here's how it works:

1. When you prepare a statement, it's prepared on a connection in the pool.  
   准备语句时，它是在池中的连接上准备的。
2. The `Stmt` object remembers which connection was used.  
   Stmt对象会记住使用了哪个连接。
3. When you execute the `Stmt`, it tries to use the connection. If it's not
	available because it's closed or busy doing something else, it gets another
	connection from the pool *and re-prepares the statement with the database on
	another connection.*  
   执行Stmt时，它会尝试使用该连接。如果由于它已关闭或忙于执行其他操作而无法使用，它将从池中获取另一个连接，并在另一个连接上使用数据库重新准备语句。	

Because statements will be re-prepared as needed when their original connection
is busy, it's possible for high-concurrency usage of the database, which may
keep a lot of connections busy, to create a large number of prepared statements.
This can result in apparent leaks of statements, statements being prepared and
re-prepared more often than you think, and even running into server-side limits
on the number of statements.

因为在原始连接繁忙时将根据需要重新准备语句，所以数据库的高并发使用可能会使很多连接繁忙，从而创建大量预准备语句。这可能导致语句的明显泄漏，正在准备和重新准备的语句比您想象的更频繁，甚至在语句数量上遇到服务器端限制。

Avoiding Prepared Statements
============================

避免准备好的陈述

Go creates prepared statements for you under the covers. A simple
`db.Query(sql, param1, param2)`, for example, works by preparing the sql, then
executing it with the parameters and finally closing the statement.

Go为您创建了准备好的陈述。例如，一个简单的db.Query（sql，param1，param2）通过准备sql，然后使用参数执行它并最终关闭语句来工作。

Sometimes a prepared statement is not what you want, however. There might be
several reasons for this:

但是，有时准备好的陈述不是你想要的。可能有以下几种原因：

1. The database doesn't support prepared statements. When using the MySQL
	driver, for example, you can connect to MemSQL and Sphinx, because they
	support the MySQL wire protocol. But they don't support the "binary" protocol
	that includes prepared statements, so they can fail in confusing ways.  

   数据库不支持预准备语句。例如，当使用MySQL驱动程序时，您可以连接到MemSQL和Sphinx，因为它们支持MySQL线程协议。但是它们不支持包含预准备语句的“二进制”协议，因此它们可能会以混乱的方式失​​败。	
2. The statements aren't reused enough to make them worthwhile, and security
	issues are handled in other ways, so performance overhead is undesired. An
	example of this can be seen at the
	[VividCortex blog](https://vividcortex.com/blog/2014/11/19/analyzing-prepared-statement-performance-with-vividcortex/).  

   这些语句的重用不足以使它们变得有价值，并且安全问题以其他方式处理，因此不希望出现性能开销。可以在VividCortex博客上看到这方面的一个例子。	

If you don't want to use a prepared statement, you need to use `fmt.Sprint()` or
similar to assemble the SQL, and pass this as the only argument to `db.Query()`
or `db.QueryRow()`. And your driver needs to support plaintext query execution,
which is added in Go 1.1 via the `Execer` and `Queryer` interfaces,
[documented here](http://golang.org/pkg/database/sql/driver/#Execer).

如果您不想使用预准备语句，则需要使用 `fmt.Sprint()` 或类似语法来组装SQL，并将其作为 `db.Query()` 或 `db.QueryRow()` 的唯一参数传递。并且您的驱动程序需要支持明文查询执行，这是通过此处记录的 `Execer` 和 `Queryer` 接口在Go 1.1中添加的。

Prepared Statements in Transactions
===================================

Prepared statements that are created in a `Tx` are bound exclusively to
it, so the earlier cautions about repreparing do not apply. When
you operate on a `Tx` object, your actions map directly to the one and only one
connection underlying it.

在Tx中创建的预处理语句仅与其绑定，因此之前关于重新表示的警告不适用。当您对Tx对象进行操作时，您的操作将直接映射到其下的唯一连接。

This also means that prepared statements created inside a `Tx` can't be used
separately from it. Likewise, prepared statements created on a `DB` can't be
used within a transaction, because they will be bound to a different connection.

这也意味着在Tx中创建的预处理语句不能与它分开使用。同样，在DB上创建的预准备语句不能在事务中使用，因为它们将绑定到不同的连接。

To use a prepared statement prepared outside the transaction in a `Tx`, you can use
`Tx.Stmt()`, which will create a new transaction-specific statement from the one
prepared outside the transaction. It does this by taking an existing prepared statement,
setting the connection to that of the transaction and repreparing all statements every
time they are executed. This behavior and its implementation are undesirable and there's
even a TODO in the `database/sql` source code to improve it; we advise against using this.

要在Tx中使用在事务外部准备的预准备语句，可以使用Tx.Stmt（），它将从事务外部准备的语句创建新的特定于事务的语句。它通过获取现有的预准备语句，将连接设置为事务的连接并在每次执行时重新表示所有语句来完成此操作。这种行为及其实现是不可取的，在数据​​库/ sql源代码中甚至还有一个TODO来改进它;我们建议不要使用它。

Caution must be exercised when working with prepared statements in
transactions. Consider the following example:

在交易中处理准备好的报表时必须谨慎行事。请考虑以下示例：

<pre class="prettyprint lang-go">
tx, err := db.Begin()
if err != nil {
	log.Fatal(err)
}
defer tx.Rollback()
stmt, err := tx.Prepare("INSERT INTO foo VALUES (?)")
if err != nil {
	log.Fatal(err)
}
defer stmt.Close() // danger!
for i := 0; i < 10; i++ {
	_, err = stmt.Exec(i)
	if err != nil {
		log.Fatal(err)
	}
}
err = tx.Commit()
if err != nil {
	log.Fatal(err)
}
// stmt.Close() runs here!
</pre>


Before Go 1.4 closing a `*sql.Tx` released the connection associated with it back into the
pool, but the deferred call to Close on the prepared statement was executed
**after** that has happened, which could lead to concurrent access to the
underlying connection, rendering the connection state inconsistent.
If you use Go 1.4 or older, you should make sure the statement is always closed before the transaction is
committed or rolled back. [This issue](https://github.com/golang/go/issues/4459) was fixed in Go 1.4 by [CR 131650043](https://codereview.appspot.com/131650043).

在Go 1.4关闭之前，* sql.Tx将与之关联的连接释放回池中，但是在已经发生的情况下执行了对准备好的语句的延迟调用Close，这可能导致并发访问底层连接，从而呈现连接状态不一致。如果使用Go 1.4或更早版本，则应确保在提交或回滚事务之前始终关闭该语句。 CR 131650043在Go 1.4中修复了此问题。

Parameter Placeholder Syntax
============================

The syntax for placeholder parameters in prepared statements is
database-specific. For example, comparing MySQL, PostgreSQL, and Oracle:

	MySQL               PostgreSQL            Oracle
	=====               ==========            ======
	WHERE col = ?       WHERE col = $1        WHERE col = :col
	VALUES(?, ?, ?)     VALUES($1, $2, $3)    VALUES(:val1, :val2, :val3)

**Previous: [Modifying Data and Using Transactions](modifying.html)**
**Next: [Handling Errors](errors.html)**
