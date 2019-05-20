---
layout: article
title: Overview
---

To access databases in Go, you use a `sql.DB`. You use this type to create
statements and transactions, execute queries, and fetch results.

要在Go中访问数据库，请使用sql.DB.您可以使用此类型来创建语句和事务，执行查询以及获取结果。

The first thing you should know is that **a `sql.DB` isn't a database
connection**. It also doesn't map to any particular database software's notion
of a "database" or "schema." It's an abstraction of the interface and existence
of a database, which might be as varied as a local file, accessed through a network
connection, or in-memory and in-process.

您应该知道的第一件事是sql.DB不是数据库连接。它也没有映射到任何特定的数据库软件的“数据库”或“模式”的概念。它是接口的抽象和数据库的存在，它可以像通过网络连接访问的本地文件一样多样化，或者在内存中和进程中。

`sql.DB` performs some important tasks for you behind the scenes:

* It opens and closes connections to the actual underlying database, via the driver.
* It manages a pool of connections as needed, which may be a variety of things as mentioned.

The `sql.DB` abstraction is designed to keep you from worrying about how to
manage concurrent access to the underlying datastore.  A connection is marked
in-use when you use it to perform a task, and then returned to the available
pool when it's not in use anymore. One consequence of this is that **if you fail
to release connections back to the pool, you can cause `sql.DB` to open a lot of
connections**, potentially running out of resources (too many connections, too
many open file handles, lack of available network ports, etc). We'll discuss
more about this later.

sql.DB抽象旨在避免担心如何管理对底层数据存储的并发访问。当您使用连接执行任务时，连接将被标记为正在使用，然后在不再使用时返回到可用池。这样做的一个结果是，如果您无法将连接释放回池，则可能导致sql.DB打开大量连接，可能耗尽资源（连接太多，打开文件句柄太多，缺少可用网络）港口等）。我们稍后会详细讨论这个问题。

After creating a `sql.DB`, you can use it to query the database that it
represents, as well as creating statements and transactions.

创建sql.DB之后，您可以使用它来查询它所代表的数据库，以及创建语句和事务。

**Next: [Importing a Database Driver](importing.html)**
