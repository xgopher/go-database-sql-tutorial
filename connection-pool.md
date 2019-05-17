---
layout: article
title: The Connection Pool
---

There is a basic connection pool in the `database/sql` package. There isn't a
lot of ability to control or inspect it, but here are some things you might find
useful to know:

`database/sql` 包中有一个基本连接池。没有很多控制或检查它的能力，但这里有一些你可能会发现有用的知识：

* Connection pooling means that executing two consecutive statements on a single database might open two connections and execute them separately. It is fairly common for programmers to be confused as to why their code misbehaves. For example, `LOCK TABLES` followed by an `INSERT` can block because the `INSERT` is on a connection that does not hold the table lock.  
  连接池意味着在单个数据库上执行两个连续语句可能会打开两个连接并分别执行它们。对于程序员来说，为什么他们的代码行为不端是很常见的。例如，后跟INSERT的LOCK TABLES可以阻塞，因为INSERT位于不保存表锁的连接上。
* Connections are created when needed and there isn't a free connection in the pool.  
  在需要时创建连接，并且池中没有空闲连接。
* By default, there's no limit on the number of connections. If you try to do a lot of things at once, you can create an arbitrary number of connections. This can cause the database to return an error such as "too many connections."  
  默认情况下，连接数没有限制。如果您尝试同时执行大量操作，则可以创建任意数量的连接。这可能导致数据库返回错误，例如“连接太多”。
* In Go 1.1 or newer, you can use `db.SetMaxIdleConns(N)` to limit the number of *idle* connections in the pool. This doesn't limit the pool size, though.
* In Go 1.2.1 or newer, you can use `db.SetMaxOpenConns(N)` to limit the number of *total* open connections to the database. Unfortunately, a [deadlock bug](https://groups.google.com/d/msg/golang-dev/jOTqHxI09ns/x79ajll-ab4J) ([fix](https://code.google.com/p/go/source/detail?r=8a7ac002f840)) prevents `db.SetMaxOpenConns(N)` from safely being used in 1.2.
* Connections are recycled rather fast. Setting a high number of idle connections with `db.SetMaxIdleConns(N)` can reduce this churn, and help keep connections around for reuse.  
  连接回收速度相当快。使用db.SetMaxIdleConns（N）设置大量空闲连接可以减少此流失，并有助于保持连接以便重用。
* Keeping a connection idle for a long time can cause problems (like in [this issue](https://github.com/go-sql-driver/mysql/issues/257) with MySQL on Microsoft Azure). Try `db.SetMaxIdleConns(0)` if you get connection timeouts because a connection is idle for too long.  
  长时间保持连接空闲可能会导致问题（例如Microsoft Azure上的MySQL问题）。如果连接超时，请尝试使用db.SetMaxIdleConns（0），因为连接空闲时间过长。
* You can also specify the maximum amount of time a connection may be reused by setting `db.SetConnMaxLifetime(duration)` since reusing long lived connections may cause network issues. This closes the unused connections lazily i.e. closing expired connection may be deferred.  
  您还可以通过设置db.SetConnMaxLifetime（duration）来指定可以重用连接的最长时间，因为重用长期连接可能会导致网络问题。这懒惰地关闭未使用的连接，即可以延迟关闭过期的连接。

**Previous: [Working with Unknown Columns](varcols.html)**
**Next: [Surprises, Antipatterns and Limitations](surprises.html)**
