---
layout: article
title: Surprises, Antipatterns and Limitations
---

惊喜，反模式和限制

Although `database/sql` is simple once you're accustomed to it, you might be
surprised by the subtlety of use cases it supports. This is common to Go's core
libraries.

虽然 `database/sql` 在您习惯使用它之后很简单，但您可能会对它支持的用例的微妙之处感到惊讶。这对Go的核心库来说很常见。

Resource Exhaustion 资源枯竭
===================

As mentioned throughout this site, if you don't use `database/sql` as intended,
you can certainly cause trouble for yourself, usually by consuming some
resources or preventing them from being reused effectively:

正如本网站所述，如果你没有按预期使用 `database/sql` ，你肯定会给自己带来麻烦，通常是消耗一些资源或防止它们被有效地重用：

* Opening and closing databases can cause exhaustion of resources.  
  打开和关闭数据库可能会导致资源耗尽。
* Failing to read all rows or use `rows.Close()` reserves connections from the pool.  
  无法读取所有行或使用 `rows.Close()` 保留池中的连接。
* Using `Query()` for a statement that doesn't return rows will reserve a connection from the pool.
  对不返回行的语句使用 `Query()` 将保留池中的连接。
* Failing to be aware of how [prepared statements](prepared.html) work can lead to a lot of extra database activity.  
  如果不了解预处理语句的工作原理，可能会导致大量额外的数据库活动。

Large uint64 Values
===================

Here's a surprising error. You can't pass big unsigned integers as parameters to
statements if their high bit is set:

这是一个令人惊讶的错误。如果设置了高位，则无法将大的无符号整数作为参数传递给语句：

<pre class="prettyprint lang-go">
_, err := db.Exec("INSERT INTO users(id) VALUES", math.MaxUint64) // Error
</pre>

This will throw an error. Be careful if you use `uint64` values, as they may
start out small and work without error, but increment over time and start
throwing errors.

这会引发错误。如果使用uint64值，请小心，因为它们可能从小开始并且没有错误地工作，但随着时间的推移而增加并开始抛出错误。

Connection State Mismatch
=========================

Some things can change connection state, and that can cause problems for two
reasons:

1. Some connection state, such as whether you're in a transaction, should be
	handled through the Go types instead.  
   某些连接状态（例如您是否在事务中）应该通过Go类型来处理。	
2. You might be assuming that your queries run on a single connection when they
	don't.  
   您可能假设您的查询在单个连接上运行时不运行。	

For example, setting the current database with a `USE` statement is a typical
thing for many people to do. But in Go, it will affect only the connection that
you run it in. Unless you are in a transaction, other statements that you think
are executed on that connection may actually run on different connections gotten
from the pool, so they won't see the effects of such changes.

例如，使用 `USE` 语句设置当前数据库对于许多人来说是典型的事情。但是在Go中，它只会影响你运行它的连接。除非你在一个事务中，你认为在该连接上执行的其他语句实际上可能在从池中获取的不同连接上运行，所以它们不会看到这种变化的影响。

Additionally, after you've changed the connection, it'll return to the pool and
potentially pollute the state for some other code. This is one of the reasons
why you should never issue BEGIN or COMMIT statements as SQL commands directly,
too.

此外，在您更改连接后，它将返回池并可能污染其他代码的状态。这也是您不应该直接将BEGIN或COMMIT语句作为SQL命令发出的原因之一。

Database-Specific Syntax
========================

The `database/sql` API provides an abstraction of a row-oriented database, but
specific databases and drivers can differ in behavior and/or syntax, such as
[prepared statement placeholders](prepared.html).

`database/sql` API提供了面向行的数据库的抽象，但特定数据库和驱动程序的行为和/或语法可能不同，例如预准备语句占位符。

Multiple Result Sets
====================

The Go driver doesn't support multiple result sets from a single query in any
way, and there doesn't seem to be any plan to do that, although there is [a
feature request](https://github.com/golang/go/issues/5171) for
supporting bulk operations such as bulk copy.

Go驱动程序不以任何方式支持来自单个查询的多个结果集，并且似乎没有任何计划这样做，尽管存在支持批量操作（如批量复制）的功能请求。

This means, among other things, that a stored procedure that returns multiple
result sets will not work correctly.

这意味着，除其他外，返回多个结果集的存储过程将无法正常工作。

Invoking Stored Procedures
==========================

Invoking stored procedures is driver-specific, but in the MySQL driver it can't
be done at present. It might seem that you'd be able to call a simple
procedure that returns a single result set, by executing something like this:

调用存储过程是特定于驱动程序的，但在MySQL驱动程序中，它目前无法完成。看起来您可以通过执行以下操作来调用返回单个结果集的简单过程：

<pre class="prettyprint lang-go">
err := db.QueryRow("CALL mydb.myprocedure").Scan(&amp;result) // Error
</pre>

In fact, this won't work. You'll get the following error: _Error
1312: PROCEDURE mydb.myprocedure can't return a result set in the given
context_. This is because MySQL expects the connection to be set into
multi-statement mode, even for a single result, and the driver doesn't currently
do that (though see [this
issue](https://github.com/go-sql-driver/mysql/issues/66)).

事实上，这是行不通的。您将收到以下错误：错误1312：PROCEDURE mydb.myprocedure无法返回给定上下文中的结果集。这是因为MySQL期望将连接设置为多语句模式，即使对于单个结果也是如此，并且驱动程序当前不会这样做（尽管看到此问题）。

Multiple Statement Support
==========================

The `database/sql` doesn't explicitly have multiple statement support, which means
that the behavior of this is backend dependent:

database / sql没有显式地支持多个语句，这意味着它的行为依赖于后端：

<pre class="prettyprint lang-go">
_, err := db.Exec("DELETE FROM tbl1; DELETE FROM tbl2") // Error/unpredictable result
</pre>

The server is allowed to interpret this however it wants, which can include
returning an error, executing only the first statement, or executing both.

允许服务器解释它想要的，包括返回错误，仅执行第一个语句或执行两者。

Similarly, there is no way to batch statements in a transaction. Each statement
in a transaction must be executed serially, and the resources in the results,
such as a Row or Rows, must be scanned or closed so the underlying connection is free
for the next statement to use. This differs from the usual behavior when you're
not working with a transaction. In that scenario, it is perfectly possible to
execute a query, loop over the rows, and within the loop make a query to the
database (which will happen on a new connection):

同样，没有办法在事务中批处理语句。事务中的每个语句必须以串行方式执行，并且必须扫描或关闭结果中的资源（如行或行），以便下一个语句可以使用基础连接。这与您不使用事务时的常规行为不同。在这种情况下，完全可以执行查询，循环遍历行，并在循环内对数据库进行查询（这将在新连接上发生）：

<pre class="prettyprint lang-go">
rows, err := db.Query("select * from tbl1") // Uses connection 1
for rows.Next() {
	err = rows.Scan(&myvariable)
	// The following line will NOT use connection 1, which is already in-use
	db.Query("select * from tbl2 where id = ?", myvariable)
}
</pre>

But transactions are bound to
just one connection, so this isn't possible with a transaction:

但事务只绑定到一个连接，因此事务无法实现：

<pre class="prettyprint lang-go">
tx, err := db.Begin()
rows, err := tx.Query("select * from tbl1") // Uses tx's connection
for rows.Next() {
	err = rows.Scan(&myvariable)
	// ERROR! tx's connection is already busy!
	tx.Query("select * from tbl2 where id = ?", myvariable)
}
</pre>

Go doesn't stop you from trying, though. For that reason, you may wind up with a
corrupted connection if you attempt to perform another statement before the
first has released its resources and cleaned up after itself.  This also means
that each statement in a transaction results in a separate set of network
round-trips to the database.

不过，Go不会阻止你尝试。因此，如果您尝试在第一个语句释放其资源并在其自身之后清理之前执行另一个语句，则最终可能会出现连接已损坏的情况。这也意味着事务中的每个语句都会导致一组单独的网络往返数据库。

**Previous: [The Connection Pool](connection-pool.html)**
**Next: [Related Reading and Resources](references.html)**
