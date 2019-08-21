## 15.6.2.2 InnoDB索引的物理结构

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-physical-structure.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-physical-structure.html)

除了特殊的地理空间索引，InnoDB索引使用B-tree数据结构。地理空间索引使用R-trees，一种用于索引多维数据的特殊数据结构。索引记录存储在B-tree或R-tree数据结构的叶子页面上。默认的索引页大小是16KB。
