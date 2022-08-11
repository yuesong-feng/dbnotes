# PostgreSQL文本搜索（五）——词法分析器

文本搜索词法分析器负责将原始文档文本分割成token，并确定每个token的类型，其中可能的类型集合由词法分析器本身定义。注意分析器根本不修改文本，它只是识别合理的词的边界。由于这种有限的范围，对特定应用的自定义分析器的需求比对自定义字典的需求要少。目前PostgreSQL只提供了一个内置的分析器，它已经被发现对广泛的应用很有用。

内置的分析器被命名为`pg_catalog.default`。它可以识别23种标记类型，如下表所示。

|Alias	|Description	|Example|
|--|--|--|
|asciiword	|Word, all ASCII letters|	elephant
|word	|Word, all letters|	mañana
|numword|	Word, letters and digits	|beta1
|asciihword|	Hyphenated word, all ASCII	|up-to-date
|hword	|Hyphenated word, all letters	|lógico-matemática
|numhword	|Hyphenated word, letters and digits	|postgresql-beta1
|hword_asciipart	|Hyphenated word part, all ASCII	|postgresql in the context postgresql-beta1
|hword_part	|Hyphenated word part, all letters	|lógico or matemática in the context lógico-matemática
|hword_numpart|	Hyphenated word part, letters and digits	|beta1 in the context postgresql-beta1
|email	|Email address	|foo@example.com
|protocol	|Protocol head	|http://
|url	|URL	|example.com/stuff/index.html
|host	|Host	|example.com
|url_path	|URL path	|/stuff/index.html, in the context of a URL
|file	|File or path name	|/usr/local/foo.txt, if not within a URL
|sfloat	|Scientific notation	|-1.234e56
|float	|Decimal notation|	-1.234
|int	|Signed integer	|-1234
|uint	|Unsigned integer	|1234
|version|	Version number	|8.3.0
|tag	|XML tag	| `<a href="dictionaries.html">`
|entity	|XML entity	| `&amp;`
|blank	|Space symbols|	(any whitespace or punctuation not otherwise recognized)

> 分析器对"字母"的概念是由数据库的区域设置决定的，特别是`lc_ctype`。仅包含基本ASCII字母的词被报告为一个单独的token类型，因为有时区分它们是很有用的。在大多数欧洲语言中，token类型`word`和`asciiword`应该被同等对待。

> `email`不全部支持RFC 5322所定义的有效的电子邮件字符。具体来说，电子邮件用户名支持的非字母数字字符只有句号、破折号和下划线。

分析器有可能从同一段文本中产生重叠的标记。例如，一个连字符的单词将被报告为整个单词和每个组成部分：

```sql
SELECT alias, description, token FROM ts_debug('foo-bar-beta1');
      alias      |               description                |     token     
-----------------+------------------------------------------+---------------
 numhword        | Hyphenated word, letters and digits      | foo-bar-beta1
 hword_asciipart | Hyphenated word part, all ASCII          | foo
 blank           | Space symbols                            | -
 hword_asciipart | Hyphenated word part, all ASCII          | bar
 blank           | Space symbols                            | -
 hword_numpart   | Hyphenated word part, letters and digits | beta1
```

这种行为允许同时对整个复合词和某个组成部分进行搜索。下面是另一个有启发性的例子：

```sql
SELECT alias, description, token FROM ts_debug('http://example.com/stuff/index.html');
  alias   |  description  |            token             
----------+---------------+------------------------------
 protocol | Protocol head | http://
 url      | URL           | example.com/stuff/index.html
 host     | Host          | example.com
 url_path | URL path      | /stuff/index.html
```
