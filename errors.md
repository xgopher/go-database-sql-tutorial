---
layout: article
title: Handling Errors
---

Almost all operations with `database/sql` types return an error as the last
value. You should always check these errors, never ignore them.

几乎所有使用 `database/sql` 类型的操作都会返回错误作为最后一个值。您应该始终检查这些错误，不要忽略它们。

There are a few places where error behavior is special-case, or there's
something additional you might need to know.

在某些地方，错误行为是特殊情况，或者您可能需要了解其他一些内容。

Errors From Iterating Resultsets
================================

Consider the following code:

<pre class="prettyprint lang-go">
for rows.Next() {
	// ...
}
if err = rows.Err(); err != nil {
	// handle the error here
}
</pre>

The error from `rows.Err()` could be the result of a variety of errors in the
`rows.Next()` loop. The loop
might exit for some reason other than finishing the loop normally, so you always
need to check whether the loop terminated normally or not. An abnormal
termination automatically calls `rows.Close()`, although it's harmless to call it
multiple times.

`rows.Err()` 中的错误可能是 `rows.Next()` 循环中的各种错误的结果。除了正常完成循环之外，循环可能由于某种原因而退出，因此您始终需要检查循环是否正常终止。异常终止会自动调用 `rows.Close()` ，尽管多次调用它是无害的。

Errors From Closing Resultsets
==============================

You should always explicitly close a `sql.Rows` if you exit the loop
prematurely, as previously mentioned. It's auto-closed if the loop exits
normally or through an error, but you might mistakenly do this:

如前所述，如果过早地退出循环，则应始终显式关闭sql.Rows。如果循环正常退出或通过错误退出，它会自动关闭，但您可能会错误地执行此操作：

<pre class="prettyprint lang-go">
for rows.Next() {
	// ...
	break; // whoops, rows is not closed! memory leak...
}
// do the usual "if err = rows.Err()" [omitted here]...
// it's always safe to [re?]close here:
if err = rows.Close(); err != nil {
	// but what should we do if there's an error?
	log.Println(err)
}
</pre>

The error returned by `rows.Close()` is the only exception to the general rule
that it's best to capture and check for errors in all database operations. If
`rows.Close()` returns an error, it's unclear what you should do.
Logging the error message or panicing might be the only sensible thing,
and if that's not sensible, then perhaps you should just ignore the error.

`rows.Close()` 返回的错误是一般规则的唯一例外，它最好捕获并检查所有数据库操作中的错误。如果rows.Close（）返回错误，则不清楚应该怎么做。记录错误消息或panicing可能是唯一明智的事情，如果这不合理，那么也许你应该忽略错误。

Errors From QueryRow()
======================

Consider the following code to fetch a single row:

<pre class="prettyprint lang-go">
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&amp;name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
</pre>

What if there was no user with `id = 1`? Then there would be no row in the
result, and `.Scan()` would not scan a value into `name`. What happens then?

如果没有id = 1的用户怎么办？然后结果中没有行，并且.Scan（）不会将值扫描到名称中。那么会发生什么？

Go defines a special error constant, called `sql.ErrNoRows`, which is returned
from `QueryRow()` when the result is empty. This needs to be handled as a
special case in most circumstances. An empty result is often not considered an
error by application code, and if you don't check whether an error is this
special constant, you'll cause application-code errors you didn't expect.

Go定义了一个特殊的错误常量，称为sql.ErrNoRows，当结果为空时，它从QueryRow（）返回。在大多数情况下，这需要作为特殊情况处理。应用程序代码通常不会将空结果视为错误，如果不检查错误是否为此特殊常量，则会导致您没有预料到的应用程序代码错误。

Errors from the query are deferred until `Scan()` is called, and then are
returned from that. The above code is better written like this instead:

查询中的错误将被推迟，直到调用 `Scan()` ，然后从中返回。上面的代码更好地编写如下：

<pre class="prettyprint lang-go">
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&amp;name)
if err != nil {
	if err == sql.ErrNoRows {
		// there were no rows, but otherwise no error occurred
	} else {
		log.Fatal(err)
	}
}
fmt.Println(name)
</pre>

One might ask why an empty result set is considered an error. There's nothing
erroneous about an empty set. The reason is that the `QueryRow()` method needs
to use this special-case in order to let the caller distinguish whether
`QueryRow()` in fact found a row; without it, `Scan()` wouldn't do anything and
you might not realize that your variable didn't get any value from the database
after all.

有人可能会问为什么空结果集被认为是错误。空集没有任何错误。原因是QueryRow（）方法需要使用这种特殊情况才能让调用者区分QueryRow（）实际上是否找到了一行;没有它，Scan（）不会做任何事情，你可能没有意识到你的变量毕竟没有从数据库中获得任何值。

You should only run into this error when you're using `QueryRow()`. If you
encounter this error elsewhere, you're doing something wrong.

Identifying Specific Database Errors
====================================

It can be tempting to write code like the following:

<pre class="prettyprint lang-go">
rows, err := db.Query("SELECT someval FROM sometable")
// err contains:
// ERROR 1045 (28000): Access denied for user 'foo'@'::1' (using password: NO)
if strings.Contains(err.Error(), "Access denied") {
	// Handle the permission-denied error
}
</pre>

This is not the best way to do it, though. For example, the string value might
vary depending on what language the server uses to send error messages.  It's
much better to compare error numbers to identify what a specific error is.

The mechanism to do this varies between drivers, however, because this isn't
part of `database/sql` itself. In the MySQL driver that this tutorial focuses
on, you could write the following code:

<pre class="prettyprint lang-go">
if driverErr, ok := err.(*mysql.MySQLError); ok { // Now the error number is accessible directly
	if driverErr.Number == 1045 {
		// Handle the permission-denied error
	}
}
</pre>

Again, the `MySQLError` type here is provided by this specific driver, and the
`.Number` field may differ between drivers. The value of the number, however,
is taken from MySQL's error message, and is therefore database specific, not
driver specific.

This code is still ugly. Comparing to 1045, a magic number, is a code smell.
Some drivers (though not the MySQL one, for reasons that are off-topic here)
provide a list of error identifiers. The Postgres `pq` driver does, for example, in
[error.go](https://github.com/lib/pq/blob/master/error.go). And there's an
external package of [MySQL error numbers maintained by
VividCortex](https://github.com/VividCortex/mysqlerr). Using such a list, the
above code is better written thus:

<pre class="prettyprint lang-go">
if driverErr, ok := err.(*mysql.MySQLError); ok {
	if driverErr.Number == mysqlerr.ER_ACCESS_DENIED_ERROR {
		// Handle the permission-denied error
	}
}
</pre>

Handling Connection Errors
==========================

What if your connection to the database is dropped, killed, or has an error?

You don't need to implement any logic to retry failed statements when this
happens. As part of the [connection pooling](connection-pool.html) in
`database/sql`, handling failed connections is built-in. If you execute a query
or other statement and the underlying connection has a failure, Go will reopen a
new connection (or just get another from the connection pool) and retry, up to
10 times.

There can be some unintended consequences, however. Some types of errors may be
retried when other error conditions happen. This might also be driver-specific.
One example that has occurred with the MySQL driver is that using `KILL` to
cancel an undesired statement (such as a long-running query) results in the
statement being retried up to 10 times.

**Previous: [Using Prepared Statements](prepared.html)**
**Next: [Working with NULLs](nulls.html)**
