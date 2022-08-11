# PostgreSQL文本搜索（三）——控制文本搜索

为了实现全文搜索，必须有一个函数来从文档中创建一个`tsvector`，从用户查询中创建一个`tsquery`。另外，我们需要以有用的顺序返回结果，所以我们需要一个函数来比较文档与查询的关联性。能够很好地显示结果也是很重要的。PostgreSQL提供了对所有这些函数的支持。

## 解析文档

PostgreSQL提供了函数`to_tsvector`，用于将一个文档转换为`tsvector`数据类型。

```sql
to_tsvector([ config regconfig, ] document text) returns tsvector
```

`to_tsvector`将一个文本文档解析为token，将token规范化为lexemes，并返回一个`tsvector`，其中列出了这些lexemes及其在文档中的位置。该文档是根据指定的或默认的文本搜索配置来处理的。下面是一个简单的例子:

```sql
SELECT to_tsvector('english', 'a fat  cat sat on a mat - it ate a fat rats');
                  to_tsvector
-----------------------------------------------------
 'ate':9 'cat':3 'fat':2,11 'mat':7 'rat':12 'sat':4
```

在上面的例子中，我们看到所产生的`tsvector`不包含单词a、on或it，单词rats变成了rat，而且标点符号`-`被忽略了。

`to_tsvector`函数在内部调用一个解析器，该解析器将文档文本分解成token，并为每个token分配一个类型。对于每个标记，都会查询一个词典列表，这个列表可以根据token的类型而变化。第一个识别该token的词典发出一个或多个规范化的lexemes来表示该token。例如，rats变成了rat，因为其中一个词典认识到rats这个词是rat的复数形式。有些词被识别为停顿词，这导致它们被忽略，因为它们出现的频率太高，在搜索中没有作用。在上面的例子中，停顿词是a、on和it。如果列表中没有词典能识别该标记，那么它也会被忽略。在这个例子中，标点符号就是这样，因为没有词典分配类型给空格token，这意味着空格永远不会被索引。解析器、词典和要索引的token类型的选择由所选择的文本搜索配置决定。在同一个数据库中，有可能有许多不同的配置，而且预定义的配置可用于各种语言。在我们的例子中使用的是`english`(y英语)的默认配置。

函数`setweight`可以用来给`tsvector`的条目标注一个给定的权重，权重是字母A、B、C或D之一。这通常用于标记来自文档不同部分的条目，如标题与正文。之后，这些信息可以用来对搜索结果进行排序。

因为`to_tsvector(NULL)`会返回NULL，所以建议在一个字段可能为空时使用并操作。下面是从结构化文档中创建`tsvector`的推荐方法：

```sql
UPDATE tt SET ti =
    setweight(to_tsvector(coalesce(title,'')), 'A')    ||
    setweight(to_tsvector(coalesce(keyword,'')), 'B')  ||
    setweight(to_tsvector(coalesce(abstract,'')), 'C') ||
    setweight(to_tsvector(coalesce(body,'')), 'D');
```

在这里，我们用`setweight`来标记完成的`tsvector`中每个lexeme的来源，然后用`tsvector`连接操作`||`来合并标记的tsvector值。
