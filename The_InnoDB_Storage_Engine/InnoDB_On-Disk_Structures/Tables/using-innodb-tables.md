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

```sql
SET GLOBAL innodb_file_per_table=1;
CREATE TABLE t3 (a INT, b CHAR (20), PRIMARY KEY (a)) ROW_FORMAT=DYNAMIC;
CREATE TABLE t4 (a INT, b CHAR (20), PRIMARY KEY (a)) ROW_FORMAT=COMPRESSED;
```

或者，你可以使用`CREATE TABLE ... TABLESPACE` 语法在通用表空间来创建一张InnoDB表。通用表空间支持所有的行格式。更多信息请查看`第15.6.3.3节 通用表空间`。

```sql
CREATE TABLE t1 (c1 INT PRIMARY KEY) TABLESPACE ts1 ROW_FORMAT=DYNAMIC;
```

`CREATE TABLE ... TABLESPACE` 语法也能用于在系统表空间创建动态行格式的InnoDB表，以及拥有紧凑或者冗余行格式的表。

```sql
CREATE TABLE t1 (c1 INT PRIMARY KEY) TABLESPACE = innodb_system ROW_FORMAT=DYNAMIC;
```

更多关于InnoDB行格式的信息，请查阅`第15.10章 InnoDB行格式`。对于如何决定一张InnoDB表采用何种行格式以及InnoDB表行格式的物理特征，请查阅 `第15.10章 InnoDB行格式`。

##### InnoDB表和主键

始终为InnoDB表创建主键索引，可以定义单列或者多列类似如下方式：

- 被最重要的查询使用。
- 从不会是空值。
- 从不会有重复值。
- 插入之后就很少改变。

例如，在一张存储人信息的表中，你不该以(firstname,lastname)建立主键，因为多人会有相同的名字，有的人没有lastname,并且有的人可能会改变他的名字。因为有这么多的约束，通常可能找不到一些明显的列来作为主键，因此你可以创建一个数字列来作为主键的全部或部分列。你可以定义一个自增的列以便当一条记录插入时自增值自动插入到列里。

虽然一张表不定义主键也能正常工作，但是主键和很多性能因素有关并且对于任何庞大或者频繁使用的表来说都是一个需要考虑的至关重要的设计因素。建议你在使用`CREATE TABLE`建表语句时总是指定一个主键。如果你创建了一张表并插入了很多的数据，然后再用`ALTER TABLE`来添加主键的话，会比在建表时就定义主键慢的多。

##### 查看InnoDB表的属性

想要查看InnoDB表的属性，可以使用`SHOW TABLE STATUS`语句：

```
mysql> SHOW TABLE STATUS FROM test LIKE 't%' \G;
*************************** 1. row ***************************
           Name: t1
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 0
 Avg_row_length: 0
    Data_length: 16384
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2015-03-16 15:13:31
    Update_time: NULL
     Check_time: NULL
      Collation: utf8mb4_0900_ai_ci
       Checksum: NULL
 Create_options:
        Comment:
```

更多关于`SHOW TABLE STATUS`语句的输出，请查看`第13.7.6.36节 SHOW TABLE STATUS语法`。

InnoDB表属性也可以从 InnoDB Information Schema系统表查询得到：

```
mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLES WHERE NAME='test/t1' \G
*************************** 1. row ***************************
     TABLE_ID: 45
         NAME: test/t1
         FLAG: 1
       N_COLS: 5
        SPACE: 35
   ROW_FORMAT: Compact
ZIP_PAGE_SIZE: 0
   SPACE_TYPE: Single
```

更多信息请查阅`15.14.3节 InnoDB INFORMATION_SCHEMA 元信息表`。
