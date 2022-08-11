# PostgreSQL文本搜索（二）——表和索引

上一节的例子说明了使用简单的常量字符串进行全文匹配。本节展示了如何搜索表数据，或选择使用索引。

## 搜索表

可以在没有索引的情况下进行全文检索。一个简单的查询是打印在`body`字段中包含`friend`这个词的所有行的`title`：

```sql
SELECT title
FROM pgweb
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'friend');
```

这也会找到相关的词，如`friend`和`friendly`，因为这些词都被简化为同一个规范化的lexeme（词位）。

上面的查询指定要使用`english`(英语)配置来解析和规范化字符串。我们也可以省略该配置参数：

```sql
SELECT title
FROM pgweb
WHERE to_tsvector(body) @@ to_tsquery('friend');
```

该查询将使用`default_text_search_config`配置。

一个更复杂的例子是查询在`title`或`body`包含`create`和`table`的十个最新的文件：

```sql
SELECT title
FROM pgweb
WHERE to_tsvector(title || ' ' || body) @@ to_tsquery('create & table')
ORDER BY last_mod_date DESC
LIMIT 10;
```

为了清楚我们省略了并操作的调用，但要找到在两个字段之一中包含`NULL`的行时需要这个操作。

尽管这些查询在没有索引的情况下也可以工作，但对大多数应用会来说这种方法太慢，也许偶尔的临时自组织搜索除外。实际上使用文本搜索通常需要创建一个索引。

## 创建索引

我们可以创建一个GIN索引来加快文本搜索的速度：

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', body));
```

注意使用的是`to_tsvector`带2个参数的版本。只有指定配置名称的文本搜索函数可以在表达式索引中使用，这是因为索引内容必须不受`default_text_search_config`的影响。如果它们受到影响，索引内容可能会不一致，因为不同的入口可能包含用不同的文本搜索配置创建的`tsvectors`，而且没有办法猜测哪个是哪个。要正确地存储和恢复这样的索引是不可能的。

因为在上面的索引中使用了`to_tsvector`带2个参数的版本，所以只有使用`to_tsvector`带2个参数的版本和相同配置的查询才能使用该索引。也就是说，`WHERE to_tsvector('english', body) @@ 'a & b'`可以使用该索引，但`WHERE to_tsvector(body) @@ 'a & b'`不能。这确保了一个索引要被使用，只能用与之相同的配置。

也可以设置更复杂的表达式索引，其中配置名称由另一列指定，例如：

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector(config_name, body));
```

其中`config_name`是`pgweb`表中的一个列。这允许在同一个索引中混合配置，同时记录每个索引条目使用的是哪种配置。如果文档集包含不同语言的文档，这就很有用。同样，要使用该索引的查询必须匹配该索引的配置，例如，`WHERE to_tsvector(config_name, body) @@ 'a & b'`。

索引甚至可以将列串联起来:

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', title || ' ' || body));
```

另一种方法是创建一个单独的`tsvector`列来保存`to_tsvector`的输出。为了使这一列与它的源数据保持自动更新，存储一个生成的列。这个例子是`title`和`body`的串联，使用并操作来确保一个字段在另一个字段为空时仍然被索引。

```sql
ALTER TABLE pgweb
    ADD COLUMN textsearchable_index_col tsvector
               GENERATED ALWAYS AS (to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))) STORED;
```

然后我们创建一个GIN索引来加快搜索速度:

```sql
CREATE INDEX textsearch_idx ON pgweb USING GIN (textsearchable_index_col);
```

现在我们已经准备好进行快速的全文搜索了:

```sql
SELECT title
FROM pgweb
WHERE textsearchable_index_col @@ to_tsquery('create & table')
ORDER BY last_mod_date DESC
LIMIT 10;
```

与表达式索引相比，分离列方法的一个优点是，为了利用索引不需要在查询中明确指定文本搜索配置。如上面的例子所示，查询可以依赖于`default_text_search_config`。另一个好处是搜索会更快，因为不需要重做`to_tsvector`的调用来验证索引的匹配。(这在使用GiST索引时比使用GIN索引更重要）。然而，表达式索引的方法设置起来更简单，而且它需要更少的磁盘空间，因为`tsvector`的表示没有被存储。
