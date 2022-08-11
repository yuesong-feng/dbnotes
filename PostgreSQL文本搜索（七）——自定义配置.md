# PostgreSQL文本搜索（七）——自定义配置

文本搜索配置指定了将文档转化为`tsvector`所需的所有选项：使用词法分析器将文本分解为token，以及使用词典将每个token转化为lexeme。每一次对`to_tsvector`或`to_tsquery`的调用都需要一个文本搜索配置来执行其处理。配置参数`default_text_search_config`指定了默认配置的名称，如果省略了一个明确的配置参数，文本搜索函数就会使用这个配置。它可以在`postgresql.conf`中设置，或者使用`SET`命令为单个会话设置。

有几个预定义的文本搜索配置可用，你也可以很容易地创建自定义配置。为了便于管理文本搜索对象，有一组SQL命令可用，还有几个psql命令可以显示文本搜索对象的信息。

下面是一个例子，我们将创建一个配置`pg`，首先是复制内置的`english`配置:

```sql
CREATE TEXT SEARCH CONFIGURATION public.pg ( COPY = pg_catalog.english );
```

我们将使用一个PostgreSQL专用的同义词列表，并将其存储在`$SHAREDIR/tsearch_data/pg_dict.syn`中。文件内容如下：

```sql
postgres    pg
pgsql       pg
postgresql  pg
```

然后这样定义同义词词典:

```sql
CREATE TEXT SEARCH DICTIONARY pg_dict (
    TEMPLATE = synonym,
    SYNONYMS = pg_dict
);
```

接下来我们注册Ispell词典`english_ispell`，它有自己的配置文件：

```sql
CREATE TEXT SEARCH DICTIONARY english_ispell (
    TEMPLATE = ispell,
    DictFile = english,
    AffFile = english,
    StopWords = english
);
```

现在我们可以在配置`pg`中设置单词的映射。

```sql
ALTER TEXT SEARCH CONFIGURATION pg
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
    WITH pg_dict, english_ispell, english_stem;
```

我们选择不索引或搜索一些内置配置处理的token类型：

```sql
ALTER TEXT SEARCH CONFIGURATION pg
    DROP MAPPING FOR email, url, url_path, sfloat, float;
```

现在可以测试我们的自定义配置：

```sql
SELECT * FROM ts_debug('public.pg', '
PostgreSQL, the highly scalable, SQL compliant, open source object-relational
database management system, is now undergoing beta testing of the next
version of our software.
');
```

下一步是设置会话使用新的配置，该配置是在`public`模式中创建的：

```sql
=> \dF
   List of text search configurations
 Schema  | Name | Description
---------+------+-------------
 public  | pg   |

SET default_text_search_config = 'public.pg';
SET

SHOW default_text_search_config;
 default_text_search_config
----------------------------
 public.pg
```
