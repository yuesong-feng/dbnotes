# PostgreSQL常用命令

`generate_series(a, b)` [a, b]间所有整数的序列

`random()` [0, 1)间的随机数

`a + random() * (b - a)` [a, b)间的随机数

`floor(a + random() * (b - a))` [a, b)间的随机整数

`chr(int4(random() * 26) + 65)` 随机大写字母

`chr(int4(random() * 26) + 97)` 随机小写字母

`repeat(exp, n)` 重复n次exp

`now()` 当前时间戳
