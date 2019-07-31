## 15.6.1.1 创建InnoDB表

> 原文地址： [https://dev.mysql.com/doc/refman/8.0/en/using-innodb-tables.html](https://dev.mysql.com/doc/refman/8.0/en/using-innodb-tables.html)

你可以使用 `CREATE TABLE`语句来创建一张InnoDB表。

> CREATE TABLE t1 (a INT, b CHAR (20), PRIMARY KEY (a)) ENGINE=InnoDB;

如果InnoDB是默认的存储引擎，建表时则你不需要用`ENGINE=InnoDB`子句指定存储引擎，默认就是InnoDB表。想要查看默认的存储引擎，使用下面的语句：

```
mysql> SELECT @@default_storage_engine;
+--------------------------+
| @@default_storage_engine |
+--------------------------+
| InnoDB                   |
+--------------------------+
```

如果你打算在一个默认存储引擎不是InnoDB的服务器上使用`mysqldump`或者主从同步来又一次执行`CREATE TABLE`语句的话，你可能仍需要使用`ENGINE=InnoDB`指定存储引擎。

一张InnoDB表和它的索引可以创建在系统表空间，单文件表空间或者通用表空间。默认的，当选项`innodb_file_per_table`开启时，InnoDB表是隐式的在一个独立的file-per-table表空间。相对的，当选项`innodb_file_per_table`关闭时，InnoDB表默认是在系统空间表创建的。如果想在通用表空间建表，你需要使用`CREATE TABLE ... TABLESPACE`语法来实现。更多信息请查阅`第15.6.3.3节 通用表空间`。

当你在单文件表空间创建一张表时，MySQL在数据目录中创建一个.ibd表空间文件。一般的，一张在InnoDB系统空间创建的表是存在已有的ibdata file,默认的数据在Mysql的data目录下。一张在通用表空间创建的表位已有的通用表空间文件.ibd文件。通用表空间的文件可以在MySQL数据目录之内或之外创建。更多信息请查看`第15.6.3.3节 通用表空间`。

内部实现来说，InnoDB为每一张表的实体添加进数据字典。此实体包含数据库名。例如，如果t1表是创建在test数据库上，数据库名的数句字典实体名称是'test/t1'。这意味着你可以在不同的数据库中创建相同名字的表，并且表明不会冲突。

##### InnoDB表和行格式

InnoDB表的默认行格式可以由`innodb_default_row_format`配置选项来配置，它的默认值是DYNAMIC。 动态和压缩的行格式允许你充分利用InnoDB表的优势，例如表压缩和高效的跨页超长列值支持。为了能使用这个行格式，配置`innodb_file_per_table`必须是打开的(默认是打开的)。

```
SET GLOBAL innodb_file_per_table=1;
CREATE TABLE t3 (a INT, b CHAR (20), PRIMARY KEY (a)) ROW_FORMAT=DYNAMIC;
CREATE TABLE t4 (a INT, b CHAR (20), PRIMARY KEY (a)) ROW_FORMAT=COMPRESSED;
```
