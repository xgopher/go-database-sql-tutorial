---
layout: article
title: Working with NULLs
---

Nullable columns are annoying and lead to a lot of ugly code. If you can, avoid
them. If not, then you'll need to use special types from the `database/sql`
package to handle them, or define your own.

可空列很烦人，导致很多丑陋的代码。如果可以，请避免使用它们。如果没有，那么你需要使用 `database/sql` 包中的特殊类型来处理它们，或者定义你自己的类型。

There are types for nullable booleans, strings, integers, and floats. Here's how
you use them:

有可空的布尔值，字符串，整数和浮点数的类型。这是你如何使用它们：

<pre class="prettyprint lang-go">
for rows.Next() {
	var s sql.NullString
	err := rows.Scan(&amp;s)
	// check err
	if s.Valid {
	   // use s.String
	} else {
	   // NULL value
	}
}
</pre>

Limitations of the nullable types, and reasons to avoid nullable columns in case
you need more convincing:

可空类型的限制，以及在需要更有说服力的情况下避免可空列的原因：

1. There's no `sql.NullUint64` or `sql.NullYourFavoriteType`. You'd need to
	define your own for this.  
   没有 `sql.NullUint64` 或 `sql.NullYourFavoriteType`。你需要为此定义自己的。	
2. Nullability can be tricky, and not future-proof. If you think something won't
	be null, but you're wrong, your program will crash, perhaps rarely enough
	that you won't catch errors before you ship them.  
   可空性可能很棘手，而且不会出现面向未来的问题。如果您认为某些内容不会为空，但是您错了，那么您的程序将崩溃，或许很少，以至于在发送之前您不会发现错误。
3. One of the nice things about Go is having a useful default zero-value for
	every variable. This isn't the way nullable things work.  
   关于Go的一个好处是为每个变量提供了一个有用的默认零值。这不是可以为空的东西工作的方式。

If you need to define your own types to handle NULLs, you can copy the design of
`sql.NullString` to achieve that.

如果需要定义自己的类型来处理NULL，可以复制sql.NullString的设计来实现它。

If you can't avoid having NULL values in your database, there is another work around that most database systems support, namely `COALESCE()`. Something like the following might be something that you can use, without introducing a myriad of `sql.Null*` types.

如果您无法避免在数据库中使用NULL值，那么大多数数据库系统都支持另一种解决方法，即 `COALESCE()`。像下面这样的东西可能是你可以使用的东西，而不会引入无数的 `sql.Null*` 类型。

<pre class="prettyprint lang-go">
rows, err := db.Query(`
	SELECT
		name,
		COALESCE(other_field, '') as otherField
	WHERE id = ?
`, 42)

for rows.Next() {
	err := rows.Scan(&name, &otherField)
	// ..
	// If `other_field` was NULL, `otherField` is now an empty string. This works with other data types as well.
}
</pre>


**Previous: [Handling Errors](errors.html)**
**Next: [Working with Unknown Columns](varcols.html)**
