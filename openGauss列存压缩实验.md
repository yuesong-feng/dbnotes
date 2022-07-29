对于时序场景，float和timestamp类型占比较大，需要重点关注，较高的压缩率可以降低磁盘空间的使用。

openGauss中，对于float使用Delta2算法，对于float使用XOR算法，推测参考了facebook关于时序数据库的论文，算法选择几乎相同：

Gorilla: A Fast, Scalable, In-Memory Time Series Database

本次实验针对float和timestamp类型，测试了openGauss列存储引擎的数据压缩率。

openGauss版本为3.0.0，表大小为50万条数据。

float值为0到100的随机浮点数，timestamp值为日期相同、一日中时间不同的随机事件戳（日期相同是为了更符合时序场景）。

鉴于以上提到的论文中的算法，对于顺序的数据有理论上更高的压缩率，对于顺序的数据进行了测试，发现顺序数据压缩率显著高于全随机的乱序数据。

测试结果如下：

|压缩级别| float(乱序)     |  float(顺序)    |  timestamp(乱序) | timestamp(时间顺序) |
| --    | --             |  --             | --              | --                |
| NO    | 3968 kB        | 3968 kB         | 3968 kB         | 3968 kB           | 
| LOW   | 3504 kB (88.3%)| 3480 kB (87.7%) | 2968 kB (74.8%) | 2504 KB (63.1%)   |
| MIDDLE| 3240 kB (81.7%)| 2032 KB (51.2%) | 2968 kB (74.8%) | 2504 KB (63.1%)   |
| HIGH  | 2968 kB (74.8%)| 1416 kB (35.7%) | 2768 kB (69.8%) | 1704 KB (42.9%)   |
