---
layout: article
title: Analyzing Prepared Statement Performance With VividCortex
---

Posted by [John Potocny](https://www.vividcortex.com/blog/author/john-potocny) on Nov 19, 2014 4:30:00 AM

Optimizing MySQL performance requires the ability to inspect production query traffic. If you’re not seeing your application’s production workload, you’re missing a vital part of the picture. In particular, there are lots of performance optimizations your systems might be doing that you’re not aware of. One of these is using prepared statements for queries.

优化MySQL性能需要能够检查生产查询流量。如果您没有看到应用程序的生产工作量，那么您就错过了图片的重要部分。特别是，您的系统可能正在进行大量性能优化，而您却不知道。其中之一是使用预准备语句进行查询。

## What Are Prepared Statements?

A prepared statement is a SQL statement with parameter placeholders which is sent to the database server and prepared for repeated execution. It’s a performance optimization as well as a security measure; it protects against attacks such as SQL injection, where an attacker hijacks unguarded string concatenation to produce malicious queries.

准备好的语句是带有参数占位符的SQL语句，它被发送到数据库服务器并准备重复执行。这是一种性能优化以及安全措施;它可以防止攻击，例如SQL注入，攻击者劫持无人看守的字符串连接以产生恶意查询。

In MySQL, as well as in most databases, you first send the SQL to the server and ask for it to be prepared with placeholders for bind parameters. The server responds with a statement ID. You then send an `execute` command to the server, passing it the statement ID and the parameters.

在MySQL以及大多数数据库中，首先将SQL发送到服务器，并要求使用占位符来准备绑定参数。服务器以语句ID响应。然后，您将执行命令发送到服务器，并向其传递语句ID和参数。

Go, which we use heavily at VividCortex, transparently prepares, executes and closes prepared statements behind the scenes for you in some circumstances. Sometimes this isn’t obvious to the programmer.

Go，我们在VividCortex中大量使用，在某些情况下为您透明地准备，执行和关闭幕后准备好的语句。有时这对程序员来说并不明显。

## A Real-Life Production Use Case

When prepared statements are being used as designed, they’re a win. Prepare once, execute many times. But if you’re preparing, executing once, and then closing the statement, you’re making three network round-trips to the server, which is a performance reduction, not an improvement.

当准备好的陈述按设计使用时，它们就是胜利。准备一次，执行多次。但是，如果您正在准备，执行一次，然后关闭该语句，那么您将对服务器进行三次网络往返，这是性能降低，而不是改进。

Yesterday I found just such an example in our central shard lookup database, which is a hotspot for all of our API traffic. Every API access first has to find the database server that stores the data for the customer environment being accessed. Here’s a screenshot of the query:

昨天我在我们的中央分片查找数据库中找到了这样一个例子，它是我们所有API流量的热点。每个API访问首先必须找到存储正在访问的客户环境的数据的数据库服务器。这是查询的屏幕截图：

![Production_total_time](https://www.vividcortex.com/hubfs/Blog/Production_total_time.png)

I’m ranking Top Queries by total time consumed, and limiting to the top two queries. You can see that the top two queries appear to be the same query! But if you look at the right-hand column, you’ll notice that the action – execute and prepare – is different.

我按总消耗时间对Top Queries进行排名，并限制为前两个查询。您可以看到前两个查询看起来是同一个查询！但是如果你看一下右栏，你会发现行动 - 执行和准备 - 是不同的。

The implication is clear. The top query on this database server is being essentially doubled in impact! Not only that, but if you look at the count, you can see that it’s actually prepared more times than it’s executed, which means sometimes it’s prepared and then not even executed! This is an artifact of Go’s database/sql package and the way it treats prepared statments behind the scenes.

言外之意很明显。此数据库服务器上的顶级查询实际上是影响加倍！不仅如此，如果你看一下计数，你可以看到它实际上准备的次数超过它的执行次数，这意味着有时它已经准备好了，甚至没有执行！这是Go的数据库/ sql包的工件以及它在幕后处理准备好的语句的方式。

If we change from a prepared statement to a plaintext query here, we’ll reduce round-trips on the network and free up a lot of resources in the database server. Simple queries like this don’t benefit from being prepared. We’ll handle the security aspect of this query by validating input (which our API framework already does for us).

如果我们在这里从准备好的语句更改为明文查询，我们将减少网络上的往返次数并释放数据库服务器中的大量资源。像这样的简单查询不会从准备中受益。我们将通过验证输入（我们的API框架已经为我们做过）来处理此查询的安全性方面。

You can see the results of replacing the prepared statement below. Making this change resulted in nearly a 30% drop in load on our shard lookup server! The savings really cannot be overstated here; this query is run every time we read or write data from a customer’s environment, so this optimization will make a big difference for us.

您可以在下面看到替换准备好的声明的结果。进行此更改导致我们的分片查找服务器上的负载下降了近30％！这里的节省真的不容小觑;每次我们从客户的环境中读取或写入数据时都会运行此查询，因此这种优化将对我们产生重大影响。

![Production_total_time_after_changes](https://www.vividcortex.com/hubfs/Blog/Production_total_time_after_changes.png)

![MYSQL_Activity](https://www.vividcortex.com/hubfs/Blog/MYSQL_Activity.png)

## Making Prepared Statements Visible

Prepared statement usage can be difficult to see in MySQL, making MySQL performance monitoring harder. There’s no internal instrumentation for it; you can’t see activity such as prepare, execute, and close distinct from just querying by sending plaintext SQL. There’s no way to get a list of all the prepared statements. In current versions of MySQL, you can’t even see prepared statement usage with the Performance Schema statement tables. The slow query log also doesn’t show preparing or closing; it only shows executed statements, and it doesn’t give any indication that they were really prepared statement executions. They look just like any other query in the log.

在MySQL中很难看到准备好的语句用法，这使得MySQL性能监控变得更加困难。它没有内部仪器;通过发送明文SQL，您无法看到准备，执行和关闭等活动与查询不同。没有办法获得所有准备好的陈述的清单。在当前版本的MySQL中，您甚至看不到使用Performance Schema语句表的预准备语句用法。慢查询日志也不显示准备或关闭;它只显示已执行的语句，并且没有任何迹象表明它们是真正准备好的语句执行。它们看起来就像日志中的任何其他查询一样。

In other words, database performance management is hard partially because databases don’t give you all the information needed. At VividCortex, we measure what matters, regardless of whether the system in question provides the data we need or not. We get the data by any means necessary (safely and at low overhead, of course).

换句话说，数据库性能管理很难，部分原因是数据库没有为您提供所需的所有信息。在VividCortex，无论相关系统是否提供我们需要的数据，我们都会测量重要的事项。我们以任何必要的方式获取数据（当然，安全且低开销）。

The only way to see MySQL performance at the prepared statement level is deep packet inspection of the type that VividCortex does. Only VividCortex performs this deep inspection of production traffic and makes it instantly visible to everyone on the team – ops, development, and management.

在预处理语句级别查看MySQL性能的唯一方法是对VividCortex所做类型的深度数据包检查。只有VividCortex对生产流量进行深入检查，并使团队中的每个人都能立即看到 - 运营，开发和管理。

## Conclusion 结论

VividCortex captures network traffic and performs deep packet inspection to pull out the finest-grained MySQL performance insight available on the market. By passively capturing and measuring every query in-flight, we enable you to drill down into your production workload at an incredible level of detail, in 1-second granularity and with microsecond precision. With this visibility, you can see and fix performance problems no other solution can even observe.

VividCortex可捕获网络流量并执行深度数据包检测，以获得市场上最精细的MySQL性能洞察力。通过被动捕获和测量飞行中的每个查询，我们使您能够以1秒的粒度和微秒精度深入了解您的生产工作负载。通过这种可见性，您可以查看并修复其他解决方案甚至无法观察到的性能问题。