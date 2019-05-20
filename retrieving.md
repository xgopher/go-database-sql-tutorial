---
layout: article
title: Retrieving Result Sets
---

There are several idiomatic operations to retrieve results from the datastore.

1. Execute a query that returns rows.
1. Prepare a statement for repeated use, execute it multiple times, and destroy it.
1. Execute a statement in a once-off fashion, without preparing it for repeated use.
1. Execute a query that returns a single row. There is a shortcut for this special case.

Go's `database/sql` function names are significant. **If a function name
includes `Query`, it is designed to ask a question of the database, and will
return a set of rows**, even if it's empty. Statements that don't return rows
should not use `Query` functions; they should use `Exec()`.

Go的数据库/ sql函数名称很重要。如果函数名称包含Query，则它旨在询问数据库的问题，并返回一组行，即使它是空的。不返回行的语句不应使用Query函数;他们应该使用Exec（）。

Fetching Data from the Database
===============================

Let's take a look at an example of how to query the database, working with
results. We'll query the `users` table for a user whose `id` is 1, and print out
the user's `id` and `name`.  We will assign results to variables, a row at a
time, with `rows.Scan()`.

我们来看一个如何查询数据库，使用结果的示例。我们将在用户表中查询id为1的用户，并打印出用户的id和名称。我们将使用rows.Scan（）将结果分配给变量，一次一行。

<pre class="prettyprint lang-go">
var (
	id int
	name string
)
rows, err := db.Query("select id, name from users where id = ?", 1)
if err != nil {
	log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
	err := rows.Scan(&amp;id, &amp;name)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(id, name)
}
err = rows.Err()
if err != nil {
	log.Fatal(err)
}
</pre>

Here's what's happening in the above code:

1. We're using `db.Query()` to send the query to the database. We check the error, as usual.
2. We defer `rows.Close()`. This is very important.
3. We iterate over the rows with `rows.Next()`.
4. We read the columns in each row into variables with `rows.Scan()`.
5. We check for errors after we're done iterating over the rows.

This is pretty much the only way to do it in Go. You can't
get a row as a map, for example. That's because everything is strongly typed.
You need to create variables of the correct type and pass pointers to them, as
shown.

这几乎是在Go中实现它的唯一方法。例如，您无法将行作为地图获取。那是因为一切都是强类型的。您需要创建正确类型的变量并将指针传递给它们，如图所示。

A couple parts of this are easy to get wrong, and can have bad consequences.

* You should always check for an error at the end of the `for rows.Next()`
  loop. If there's an error during the loop, you need to know about it. Don't
  just assume that the loop iterates until you've processed all the rows.  
  您应该始终在for rows.Next（）循环结束时检查错误。如果循环期间出现错误，您需要了解它。不要只是假设循环迭代，直到您处理完所有行。
* Second, as long as there's an open result set (represented by `rows`), the
  underlying connection is busy and can't be used for any other query. That
  means it's not available in the connection pool. If you iterate over all of
  the rows with `rows.Next()`, eventually you'll read the last row, and
  `rows.Next()` will encounter an internal EOF error and call `rows.Close()` for
  you. But if for some reason you exit that loop -- an early return, or so on --
  then the `rows` doesn't get closed, and the connection remains open. (It is
  auto-closed if `rows.Next()` returns false due to an error, though). This is
  an easy way to run out of resources.  
  其次，只要有一个打开的结果集（由行表示），底层连接就会忙，不能用于任何其他查询。这意味着它在连接池中不可用。如果使用rows.Next（）迭代所有行，最终将读取最后一行，rows.Next（）将遇到内部EOF错误并为您调用rows.Close（）。但是如果由于某种原因你退出那个循环 - 早期返回，或者等等 - 那么行就不会被关闭，并且连接仍然是打开的。 （如果rows.Next（）由于错误而返回false，则会自动关闭）。这是耗尽资源的简便方法。
* `rows.Close()` is a harmless no-op if it's already closed, so you can call
  it multiple times. Notice, however, that we check the error first, and only
  call `rows.Close()` if there isn't an error, in order to avoid a runtime panic.  
  rows.Close（）是一个无害的无操作，如果它已经关闭，所以你可以多次调用它。但是请注意，我们首先检查错误，如果没有错误，只调用rows.Close（），以避免运行时出现混乱。
* You should **always `defer rows.Close()`**, even if you also call `rows.Close()`
  explicitly at the end of the loop, which isn't a bad idea. 
  你应该总是推迟rows.Close（），即使你也在循环结束时显式地调用rows.Close（），这不是一个坏主意。
* Don't `defer` within a loop. A deferred statement doesn't get executed until
  the function exits, so a long-running function shouldn't use it. If you do,
  you will slowly accumulate memory. If you are repeatedly querying and
  consuming result sets within a loop, you should explicitly call `rows.Close()`
  when you're done with each result, and not use `defer`.  
  不要在循环中推迟。在函数退出之前，延迟语句不会执行，因此长时间运行的函数不应该使用它。如果你这样做，你会慢慢积累记忆。如果您在循环中反复查询和使用结果集，则应在完成每个结果时显式调用rows.Close（），而不是使用defer。

How Scan() Works
================

When you iterate over rows and scan them into destination variables, Go performs data
type conversions work for you, behind the scenes. It is based on the type of the
destination variable. Being aware of this can clean up your code and help avoid
repetitive work.

迭代行并将它们扫描到目标变量时，Go会在后台执行数据类型转换。它基于目标变量的类型。意识到这一点可以清理代码并帮助避免重复性工作。

For example, suppose you select some rows from a table that is defined with
string columns, such as `VARCHAR(45)` or similar. You happen to know, however,
that the table always contains numbers. If you pass a pointer to a string, Go
will copy the bytes into the string. Now you can use `strconv.ParseInt()` or
similar to convert the value to a number. You'll have to check for errors in the
SQL operations, as well as errors parsing the integer. This is messy and
tedious.

例如，假设您从使用字符串列定义的表中选择某些行，例如VARCHAR（45）或类似的行。但是，您恰好知道该表始终包含数字。如果将指针传递给字符串，Go会将字节复制到字符串中。现在，您可以使用strconv.ParseInt（）或类似方法将值转换为数字。您必须检查SQL操作中的错误，以及解析整数的错误。这是混乱和乏味的。

Or, you can just pass `Scan()` a pointer to an integer. Go will detect that and
call `strconv.ParseInt()` for you. If there's an error in conversion, the call
to `Scan()` will return it. Your code is neater and smaller now. This is the
recommended way to use `database/sql`.

或者，您可以将Scan（）传递给整数。 Go将检测到并为您调用strconv.ParseInt（）。如果转换时出错，则调用Scan（）将返回它。您的代码现在更整洁更小。这是使用database / sql的推荐方法。

Preparing Queries
=================

You should, in general, always prepare queries to be used multiple times. The
result of preparing the query is a prepared statement, which can have
placeholders (a.k.a. bind values) for parameters that you'll provide when you
execute the statement.  This is much better than concatenating strings, for all
the usual reasons (avoiding SQL injection attacks, for example).

通常，您应该始终准备多次使用的查询。准备查询的结果是一个预准备语句，它可以为您在执行语句时提供的参数提供占位符（a.k.a.绑定值）。由于所有常见原因（例如，避免SQL注入攻击），这比串联字符串要好得多。

In MySQL, the parameter placeholder is `?`, and in PostgreSQL it is `$N`, where
N is a number. SQLite accepts either of these.  In Oracle placeholders begin with
a colon and are named, like `:param1`. We'll use `?` because we're using MySQL
as our example.

在MySQL中，参数占位符是？，而在PostgreSQL中它是$ N，其中N是数字。 SQLite接受其中任何一个。在Oracle中，占位符以冒号开头并命名，如：param1。我们会用吗？因为我们以MySQL为例。

<pre class="prettyprint lang-go">
stmt, err := db.Prepare("select id, name from users where id = ?")
if err != nil {
	log.Fatal(err)
}
defer stmt.Close()
rows, err := stmt.Query(1)
if err != nil {
	log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
	// ...
}
if err = rows.Err(); err != nil {
	log.Fatal(err)
}
</pre>

Under the hood, `db.Query()` actually prepares, executes, and closes a prepared
statement. That's three round-trips to the database. If you're not careful, you
can triple the number of database interactions your application makes! Some
drivers can avoid this in specific cases,

在引擎盖下，db.Query（）实际上准备，执行和关闭准备好的语句。这是数据库的三次往返。如果您不小心，您可以将应用程序的数据库交互次数增加三倍！某些驱动程序可以在特定情况下避免这种情况，但并非所有驱有关更多信息，请参

but not all drivers do. See [prepared statements](prepared.html) for more.

Single-Row Queries
==================

If a query returns at most one row, you can use a shortcut around some of the
lengthy boilerplate code:

<pre class="prettyprint lang-go">
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&amp;name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
</pre>

Errors from the query are deferred until `Scan()` is called, and then are
returned from that. You can also call `QueryRow()` on a prepared statement:

<pre class="prettyprint lang-go">
stmt, err := db.Prepare("select name from users where id = ?")
if err != nil {
	log.Fatal(err)
}
defer stmt.Close()
var name string
err = stmt.QueryRow(1).Scan(&amp;name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
</pre>

**Previous: [Accessing the Database](accessing.html)**
**Next: [Modifying Data and Using Transactions](modifying.html)**
