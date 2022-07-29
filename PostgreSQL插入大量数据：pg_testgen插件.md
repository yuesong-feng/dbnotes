# PostgreSQL test generator
在进行数据库开发、测试时，新建表之后，时常想自己插入数据，但十分麻烦。

pg_testgen插件可以产生大量随机数据，方便进行数据库开发测试。

插件地址：[pg_testgen](https://github.com/yuesong-feng/pg_testgen)

### 安装方法：
```bash
cd contrib/pg_testgen //进入插件目录
make
make install
```
然后进入数据库、启用插件即可：
```bash
CREATE EXTENSION pg_testgen;
```

### API:

| func | descryption |
| -- | -- |
| rand_int() | random 32 bits integer |
| rand_int(a, b) | random 32 bits integer between [a, b] |
| rand_text() | random text (size in [1, 32]) |
| rand_text(a) | random text (size is a) |
| rand_text(a, b) | random text (size between [a, b]) |
| rows_int(r) | r rows of rand_int() |
| rows_int(r, a, b) | r rows of rand_int(a, b) |
| rows_text(r) | r rows of rand_text()|
| rows_text(r, a) | r rows of rand_text(a) |
| rows_text(r, a, b) | r rows of rand_text(a, b) |

### 如何使用：

表结构为`t(id int, txt text)`，使用`insert into t select rows_int(5000, 1, 100), rows_text(5000, 20, 30);`可以批量插入5000条随机数据，id字段为值在[1, 100]之间的随机整数，txt字段为长度在[20, 30]的随机字符串。
