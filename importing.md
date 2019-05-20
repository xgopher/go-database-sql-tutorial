---
layout: article
title: Importing a Database Driver
---

To use `database/sql` you'll need the package itself, as well as a driver for
the specific database you want to use.

要使用database / sql，您需要包本身，以及您要使用的特定数据库的驱动程序。

You generally shouldn't use driver packages directly, although some drivers
encourage you to do so. (In our opinion, it's usually a bad idea.) Instead, your
code should only refer to types defined in `database/sql`, if possible. This
helps avoid making your code dependent on the driver, so that you can change the
underlying driver (and thus the database you're accessing) with minimal code
changes. It also forces you to use the Go idioms instead of ad-hoc idioms that a
particular driver author may have provided.

您通常不应该直接使用驱动程序包，但有些驱动程序会鼓励您这样做。 （在我们看来，这通常是一个坏主意。）相反，您的代码应该只引用database / sql中定义的类型，如果可能的话。这有助于避免使代码依赖于驱动程序，因此您可以使用最少的代码更改来更改底层驱动程序（以及您正在访问的数据库）。它还会强制您使用Go习语而不是特定驱动程序作者可能提供的特殊习语。

In this documentation, we'll use the excellent [MySQL
drivers](https://github.com/go-sql-driver/mysql) from @julienschmidt and @arnehormann
 for examples.

在本文档中，我们将使用@julienschmidt和@arnehormann的优秀MySQL驱动程序作为示例。

Add the following to the top of your Go source file:

<pre class="prettyprint lang-go">
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
</pre>

Notice that we're loading the driver anonymously, aliasing its package qualifier
to `_` so none of its exported names are visible to our code. Under the hood,
the driver registers itself as being available to the `database/sql` package,
but in general nothing else happens with the exception that the init function is run.

请注意，我们正在匿名加载驱动程序，将其包限定符别名为_，因此我们的代码不会看到其导出的名称。在引擎盖下，驱动程序将自己注册为可用于database / sql包，但一般情况下，除了运行init函数之外没有其他任何事情发生。

Now you're ready to access a database.

**Previous: [Overview of Go's database/sql Package](overview.html)**
**Next: [Accessing the Database](accessing.html)**
