## 15.1.2 InnoDB最佳实践

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-best-practices.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-best-practices.html)

这一章节介绍使用InnoDB的最佳实践。

- 为每一张表最经常访问的一个或多个列设置主键索引，如果没有很明显的主键字段，可以使用一个自增字段。

- 根据多张表中相同的ID值，使用JOIN从这些表中提取数据时。如果想获得最高的JOIN性能，在JOIN的列上加上外键，并把它们定义为相同的数据类型。增加外键能够确保关联列能被索引，从而提升性能。外键同时能够把删除和更新操作同步到所有相关联的表中，并且阻止向子表中插入父表没有关联ID的数据。

- 关闭自动提交。一秒钟提交数百次会限制性能（受限于你的存储设备的写入速度）。

- 如果你不想太频繁的提交，或者你也不想运行大量的增删改语句几个小时也没有一次提交，你可以将相关联的DML操作分组放到不同的事务中，通过 `START TRANSACTION` 和 `COMMIT` 语句来包裹它们。

- 不要使用`LOCK TABLES`语句。在不牺牲可靠性和高性能的情况下，InnoDB能够处理多个会话同时读写同一张表。为了获取若干行的独占写权限，你可以使用`SELECT ... FOR UPDATE`语句来锁住你想更新的行数据。

- 打开 `innodb_file_per_table` 配置选项或者使用通用表空间，可以把数据和索引放到各个单独的文件中，而不是全部使用系统表空间。
`innodb_file_per_table` 选项是默认打开的。

- 评估你的数据库数据或者访问者是否可以利用InnoDB的表或页压缩功。你可以在不牺牲读写性能的情况下压缩InnoDB表。

- 你可以使用 `--sql_mode=NO_ENGINE_SUBSTITUTION` 来限制别人在`CREATE TABLE`语句中用`ENGINE=`语法来创建其他的存储引擎表。
