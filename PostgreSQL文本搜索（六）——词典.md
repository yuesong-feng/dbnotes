# PostgreSQL文本搜索（六）——词典

词典用于消除在搜索中不应考虑的词（停顿词），并使词规范化，以便同一词的不同派生形式能够匹配。一个成功规范化的词被称为lexeme（词位、词素）。除了提高搜索质量之外，规范化和删除停顿词可以减少文档的`tsvector`的大小，从而提高性能。规范化并不总是具有语言学意义，通常取决于实际应用决定的语义。

一些规范化的例子：

- 语言学--Ispell词典试图将输入的单词简化到一个规范化的形式；stemmer（词干分析器）词典去除单词的词尾
- URL位置可以被规范化，以使相等的URL相匹配。
  - `http://www.pgsql.ru/db/mw/index.html`
  - `http://www.pgsql.ru/db/mw/`
  - `http://www.pgsql.ru/db/../db/mw/index.html`
- 颜色名称可以用它们的十六进制值代替，例如， red, green, blue, magenta -> FF0000, 00FF00, 0000FF, FF00FF
- 如果对数字进行索引，我们可以去掉一些小数位，以减少可能的数字范围，因此，例如3.14159265359，3.1415926，3.14如果只保留小数点后的两个数字，那么规范化后的数字将是一样的。

词典是一个程序，它接受一个token作为输入并返回：

- 如果输入的token是词典中已知的，则是一个lexeme数组（注意，一个token可以产生一个以上的lexeme）。
- 设置了`TSL_FILTER`标志的单个lexeme，它将作为一个新的token替换原有的token，以传递给后续的词典（这样做的词典被称为过滤词典）
- 如果词典知道该token但它是一个停顿词，返回一个空数组
- 如果词典不能识别输入的token，则返回NULL

PostgreSQL为许多语言提供预定义的词典。也有几个预定义的模板，可以用来创建带有自定义参数的新词典。下文描述了每个预定义的词典模板。如果没有现成的模板适合，也可以创建新的模板，PostgreSQL发行版的`contrib/`目录中有相关的例子。

文本搜索配置将一个分析器与一组词典绑定在一起，以处理分析器的输出token。对于分析器可以返回的每一种token类型，配置都会指定一个单独的词典列表。当分析器发现该类型的token时，依次查询列表中的每一个词典，直到某个词典将其识别为一个已知的词。如果它被识别为一个停顿词，或者没有字典识别该标记，它将被丢弃，不被索引或搜索。通常情况下，第一个返回非NULL输出的词典决定结果，其余的词典都不会被查询；但是过滤词典可以用一个修改过的词来替换给定的词，然后再传递给后面的词典。

配置词典列表的一般规则是，首先放置最细粒度、最具体的字典，然后是更通用的字典，最后是非常通用的字典，如Snowball词干分析器或simple，它可以识别所有的东西。例如，对于一个特定的天文学搜索（`astro_en`配置），可以将toekn类型`asciiword`（ASCII词）绑定到一个天文学术语的同义词词典，一个一般的英语词典和一个Snowball英语词干分析器：

```sql
ALTER TEXT SEARCH CONFIGURATION astro_en
    ADD MAPPING FOR asciiword WITH astrosyn, english_ispell, english_stem;
```

过滤词典可以放在列表的任何地方，但放在最后是没用的。过滤词典对于部分规范化的单词很有用，可以简化后面的词典的任务。例如，过滤词典可以用来去除重音字母的重音，例如`unaccent`模块。

## 停顿词

停顿词是非常常见的词，几乎出现在每份文件中，并且没有任何区分价值。因此，在全文搜索的背景下，它们可以被忽略。例如，每个英文文本都包含像a和the这样的词，所以在索引中存储它们是没有用的。然而，停顿词会影响到`tsvector`中的位置信息，这反过来又会影响排序：

```sql
SELECT to_tsvector('english', 'in the list of stop words');
        to_tsvector
----------------------------
 'list':3 'stop':5 'word':6
```

缺少的位置1,2,4是由于停顿词。对有停顿词和没有停顿词的文件所计算的排序是完全不同的：

```sql
SELECT ts_rank_cd (to_tsvector('english', 'in the list of stop words'), to_tsquery('list & stop'));
 ts_rank_cd
------------
       0.05

SELECT ts_rank_cd (to_tsvector('english', 'list stop words'), to_tsquery('list & stop'));
 ts_rank_cd
------------
        0.1
```

至于如何处理停顿词，则取决于具体的词典。例如，ispell词典首先对单词进行规范化处理，然后再查看停顿词列表，而Snowball词干分析器首先检查停顿词列表。不同处理方式的原因是试图减少干扰。

## simple词典（意：简单词典）

`simple`词典模板是将输入的token转换为小写字母，并将其与一个停顿词文件进行核对。如果在该文件中找到它，则返回一个空数组，导致该token被丢弃。如果没有，该词的小写形式将作为规范化的lexeme返回。另外，可以配置词典，使它将非停顿词报告为未识别的，允许它们被传递到列表中的下一个词典。

下面是一个使用`simple`模板来定义词典的例子：

```sql
CREATE TEXT SEARCH DICTIONARY public.simple_dict (
    TEMPLATE = pg_catalog.simple,
    STOPWORDS = english
);
```

在这里，`english`是一个停顿词文件的基本名称。文件的全名是`$SHAREDIR/tsearch_data/english.stop`，其中`$SHAREDIR`指的是PostgreSQL安装的共享数据目录，通常是`/usr/local/share/postgresql`（如果不确定，可以使用`pg_config --sharedir`来查看）。文件格式是一个简单的单词列表，每行一个。空行和尾部的空格被忽略，大写字母被转化成小写字母，但对文件内容不做其他处理。

现在可以测试我们的字典：

```sql
SELECT ts_lexize('public.simple_dict', 'YeS');
 ts_lexize
-----------
 {yes}

SELECT ts_lexize('public.simple_dict', 'The');
 ts_lexize
-----------
 {}
```

如果它在停顿词文件中没有找到的话，我们也可以选择返回NULL而不是小写字母单词。这可以通过设置字典的`Accept`参数为`false`来选择：

```sql
ALTER TEXT SEARCH DICTIONARY public.simple_dict ( Accept = false );

SELECT ts_lexize('public.simple_dict', 'YeS');
 ts_lexize
-----------


SELECT ts_lexize('public.simple_dict', 'The');
 ts_lexize
-----------
 {}
```

采取默认设置`Accept = true`时，只有在把一个`simple`词典放在词典列表的末尾时才有用，因为它不会把任何token传给后面的词典。相反，`Accept = false`只有在至少有一个后续词典时才有用。

> 大多数类型的字典都依赖于配置文件，如停顿词的文件。这些文件必须以UTF-8编码存储。如果不是的话，当它们被读入服务器时将被转译成实际的数据库编码。

> 通常情况下，一个数据库会话只读一次字典配置文件，当它在会话中第一次使用时。如果你修改了一个配置文件，并想迫使现有的会话接受新的内容，可以在字典上发出`ALTER TEXT SEARCH DICTIONARY`命令。这可以是一个"哑"更新，实际上并不改变任何参数值。

## Synonym词典（意：同义词词典）

这个词典模板创建的词典可以将一个词替换成同义词。不支持短语（`Thesaurus`词典支持）。同义词典可以用来解决语言学问题，例如，防止英语词干分析词典将"Paris"简化为"pari"，只需在同义词词典中设置`Paris paris`行，并将其放在`english_stem`词典之前。例如：

```sql
SELECT * FROM ts_debug('english', 'Paris');
   alias   |   description   | token |  dictionaries  |  dictionary  | lexemes 
-----------+-----------------+-------+----------------+--------------+---------
 asciiword | Word, all ASCII | Paris | {english_stem} | english_stem | {pari}

CREATE TEXT SEARCH DICTIONARY my_synonym (
    TEMPLATE = synonym,
    SYNONYMS = my_synonyms
);

ALTER TEXT SEARCH CONFIGURATION english
    ALTER MAPPING FOR asciiword
    WITH my_synonym, english_stem;

SELECT * FROM ts_debug('english', 'Paris');
   alias   |   description   | token |       dictionaries        | dictionary | lexemes 
-----------+-----------------+-------+---------------------------+------------+---------
 asciiword | Word, all ASCII | Paris | {my_synonym,english_stem} | my_synonym | {paris}
```

`synonym`模板唯一需要的参数是`SYNONYMS`，这是其配置文件的名称，在上面的例子中是`my_synonyms`。文件的全称是`$SHAREDIR/tsearch_data/my_synonyms.syn`（其中`$SHAREDIR`指PostgreSQL安装的共享数据目录）。文件的格式是每个要替换的字占一行，后面是它的同义词，用空格隔开。空行和尾部的空格会被忽略。

同义词模板也有一个可选的参数`CaseSensitive`，默认为`false`。当`CaseSensitive`为`false`时，同义词文件中的词会被转化成小写，输入的token是如此。当它为`true`时，单词和token不会被转化成小写，而是按原始值比较。

一个星号（*）可以放在配置文件中同义词的末尾，表示该同义词是一个前缀。当规则被用于`to_tsvector()`时，星号被忽略，但是当它被用于`to_tsquery()`时，返回值将是一个带有前缀匹配token的查询项。例如，假设我们在`$SHAREDIR/tsearch_data/synonym_sample.syn`中有这些规则：

```sql
postgres        pgsql
postgresql      pgsql
postgre pgsql
gogle   googl
indices index*
```

然后我们会得到这些返回：

```sql
CREATE TEXT SEARCH DICTIONARY syn (template=synonym, synonyms='synonym_sample');
SELECT ts_lexize('syn', 'indices');
 ts_lexize
-----------
 {index}

CREATE TEXT SEARCH CONFIGURATION tst (copy=simple);
ALTER TEXT SEARCH CONFIGURATION tst ALTER MAPPING FOR asciiword WITH syn;
SELECT to_tsvector('tst', 'indices');
 to_tsvector
-------------
 'index':1

SELECT to_tsquery('tst', 'indices');
 to_tsquery
------------
 'index':*

SELECT 'indexes are very useful'::tsvector;
            tsvector             
---------------------------------
 'are' 'indexes' 'useful' 'very'

SELECT 'indexes are very useful'::tsvector @@ to_tsquery('tst', 'indices');
 ?column?
----------
 t
```

## Thesaurus词典（意：同义短语词典）

同义短语词典（有时缩写为TZ）是一个词的集合，其中包括关于词和短语关系的信息，例如广义词（BT），狭义词（NT），优先词，非优先词，关联词等等。

thesaurus词典用一个优先词替换所有非优先词，并且可以选择保留原始词来进行索引。PostgreSQL目前对thesaurus字典的实现是synonym字典的扩展，增加了短语支持。一个thesaurus字典需要一个如下格式的配置文件：

```sql
# this is a comment
sample word(s) : indexed word(s)
more sample word(s) : more indexed word(s)
...
```

其中冒号（:）作为短语和其替代物之间的分隔符。

thesaurus词典使用一个子词典（在词典的配置中指定），在检查短语匹配之前对输入文本进行规范化处理，只能选择一个子词典。如果子词典不能识别一个词，就会报告一个错误。在这种情况下，应该删除该词的使用或将它加入到子词典。可以在一个索引词的开头放置一个星号（*）来跳过对它使用子字典，但所有的样本词都必须对子字典所可见。

如果有多个短语与输入相匹配，thesaurus词典会选择最长的匹配，相等则选择最后所定义的。

被字词典所识别的特定的停顿词不能被指定，而是使用`?`来标记停顿词可能出现的位置。例如，假设在子词典中，`a`和`the`是停止词：

```sql
? one ? two : swsw
```

匹配 `a one the two`和`the one a two`，两者都将被`swsw`所取代。

由于thesaurus词典具有识别短语的能力，它必须记住自己的状态并与分析器交互。thesaurus词典使用这些信息来检查它是否应该处理下一个词或停止。thesaurus词典必须被仔细配置。例如，如果词库词典被分配为只处理`asciiword`标记，那么定义`one 7`将无法工作，因为token类型`uint`没有被分配给thesaurus词典。

> thesaurus词典是在索引过程中使用的，所以thesaurus词典的参数有任何变化都需要重新索引。对于大多数其他的词典类型，小的变化，如添加或删除停顿词，并不强制重新索引。

### Thesaurus配置（意：同义短语配置）

要定义一个新的thesaurus词典，请使用`thesaurus`模板。比如：

```sql
CREATE TEXT SEARCH DICTIONARY thesaurus_simple (
    TEMPLATE = thesaurus,
    DictFile = mythesaurus,
    Dictionary = pg_catalog.english_stem
);
```

这里：

- `thesaurus_simple`是新的词典名
- `mythesaurus`是thesaurus词典配置文件的名称。(它的全名是`$SHAREDIR/tsearch_data/mythesaurus.ths`，其中`$SHAREDIR`指的是安装共享数据目录)。
- `pg_catalog.english_stem`是用于词库规范化的子词典（这里是一个Snowball英语词干分析器）。请注意，子词典将有自己的配置（例如停顿词），这里没有显示。

现在可以在配置中把词典`thesaurus_simple`与所需的token类型绑定，例如:

```sql
ALTER TEXT SEARCH CONFIGURATION russian
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart
    WITH thesaurus_simple;
```

### Thesaurus例子

考虑一个简单的天文词库`thesaurus_astro`，其中包含一些天文词的组合:

```sql
supernovae stars : sn
crab nebulae : crab
```

下面我们创建一个字典，并将一些token类型与天文词库和英语词干分析器绑定:

```sql
CREATE TEXT SEARCH DICTIONARY thesaurus_astro (
    TEMPLATE = thesaurus,
    DictFile = thesaurus_astro,
    Dictionary = english_stem
);

ALTER TEXT SEARCH CONFIGURATION russian
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart
    WITH thesaurus_astro, english_stem;
```

现在我们可以看到它是如何工作的。`ts_lexize`对于测试词库不是很有用，因为它把它的输入作为一个单一的token。相反，我们可以使用`plainto_tsquery`和`to_tsvector`，它们会把输入的字符串分成多个token:

```sql
SELECT plainto_tsquery('supernova star');
 plainto_tsquery
-----------------
 'sn'

SELECT to_tsvector('supernova star');
 to_tsvector
-------------
 'sn':1
```

原则上，`to_tsquery`可以将参数用引号引用：

```sql
SELECT to_tsquery('''supernova star''');
 to_tsquery
------------
 'sn'
```

请注意，`supernova star`与`thesaurus_astro`中的`supernovae stars`相匹配，因为我们在词库定义中指定了`english_stem`词干分析器，所以该词干分析器会去除e和s。

要为原词组以及替代词建立索引，只需将其纳入定义的右侧部分即可：

```sql
supernovae stars : sn supernovae stars

SELECT plainto_tsquery('supernova star');
       plainto_tsquery
-----------------------------
 'sn' & 'supernova' & 'star'
```

## Ispell词典

不常用，未翻译，使用请见官网文档。

## Snowball词典

不常用，未翻译，使用请见官网文档。
