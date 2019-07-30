## 15.1.3 确定InnoDB是否是数据库的默认存储引擎

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-check-availability.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-check-availability.html)

使用 `SHOW ENGINES` 语句来查看所有可用的MySQL存储引擎，看InnoDB引擎是否有DEFAULT字样。

> mysql> SHOW ENGINES;

或者，查询 `INFORMATION_SCHEMA.ENGINES` 表。

> mysql> SELECT * FROM INFORMATION_SCHEMA.ENGINES;
