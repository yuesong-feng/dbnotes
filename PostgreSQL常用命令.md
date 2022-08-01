`generate_series(a, b)` [a, b]间所有整数的序列

`random()` [0, 1)间的随机数

`a + random() * (b - a)` [a, b)间的随机数

`floor(a + random() * (b - a))` [a, b)间的随机整数

`chr(int4(random() * 26) + 65)` 随机大写字母

`chr(int4(random() * 26) + 97)` 随机小写字母

`repeat(exp, n)` 重复n次exp

先创建`random_string()`函数：

```
CREATE OR REPLACE FUNCTION random_string(
  num INTEGER,
  chars TEXT default '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
) RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
  res_str TEXT := '';
BEGIN
  IF num < 1 THEN
      RAISE EXCEPTION 'Invalid length';
  END IF;
  FOR __ IN 1..num LOOP
    res_str := res_str || substr(chars, floor(random() * length(chars))::int + 1, 1);
  END LOOP;
  RETURN res_str;
END $$;
```

`random_string(n)` 生长度为n的随机字符串，字符为大小写字母和数字

`random_string(n, 'str')` 生长度为n的随机字符串，字符为大小写字母和数字

`now()` 当前时间戳
