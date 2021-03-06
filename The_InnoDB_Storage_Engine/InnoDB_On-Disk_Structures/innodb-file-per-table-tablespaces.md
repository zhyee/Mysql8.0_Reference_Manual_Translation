## 15.6.3.2 单文件表的表空间
> 原文地址 [https://dev.mysql.com/doc/refman/8.0/en/innodb-file-per-table-tablespaces.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-file-per-table-tablespaces.html)

一个单文件表的表空间保存了单独一张InnoDB表的数据以及索引，这些数据都保存在文件系统上一个单独的数据文件中。

本节中的单文件表的表空间将按照如下几个主题来介绍它相关的特征：

- [单文件表的表空间配置](#单文件表的表空间配置)
- [单文件表的表空间数据文件](#单文件表表空间数据文件)
- [单文件表的表空间所具有的优势](#单文件表表空间具备的优势)
- [单文件表的表空间的缺点](#单文件表空间的缺点)

### 单文件表的表空间配置

InnoDB默认在单文件类型得表空间创建表，这个行为受`innodb_file_per_table`配置变量控制。禁用`innodb_file_per_table`会导致InnoDB在系统空间创建表。

`innodb_file_per_table`配置项可以在配置文件中设定，也可以在服务器运行的时候使用`SET GLOBAL`语句来进行设定。运行时修改配置需要足够的权限来设置全局系统变量，具体请查看`5.1.9.1 系统变量的权限`。

使用配置文件：

```
[mysqld]
innodb_file_per_table=ON
```

服务器运行时使用`SET GLOBAL`:

> mysql> SET GLOBAL innodb_file_per_table=ON;

### 单文件表表空间数据文件

单文件表的表空间构建在一个.ibd数据文件中，这个文件存储在MySQL数据文件夹下某个元数据目录里。.ibd文件用表名来命名（table_name.ibd）。例如表test.t1的数据文件位于MySQL数据目录下的test目录中：

```
mysql> USE test;  
  
mysql> CREATE TABLE t1 (
   id INT PRIMARY KEY AUTO_INCREMENT,
   name VARCHAR(100) 
 ) ENGINE = InnoDB; 
 
shell> cd /path/to/mysql/data/test
shell> ls 
t1.ibd
```

你可以使用 `CREATE TABLE`的`DATA DIRECTORY`子句来显示得在MySQL数据目录之外创建单文件表空间的数据文件。更多相关信息，请查阅 `第15.6.1.2节 创建外部的表`。


### 单文件表表空间具备的优势

单文件表表空间相比较于共享的表空间（例如系统表空间和通用表空间）有如下一些优势：

- 单文件表表空间中创建的表再被清空或删除后，磁盘空间可以归还给操作系统。在共享表空间表被清空或删除后释放的空间只能被`InnoDB`数据使用，换句话说，共享表空间的数据文件在表被清空或删除时，它的大小并不会减小。

- 共享空间中的表在进行表复制`ALTER TABLE`操作时会增加磁盘占用。这种操作可能会需要和表中数据加索引相等的空间。这块空间不会返还给操作系统。

- 在单文件表空间执行表的`TRUNCATE TABLE`操作性能会更好。

- 单文件表的数据文件可以保存在不同的存储设备上，以此可以达到I/O优化，空间管理或者备份的目的。请查看`第15.6.1.2节 在MySQL目录之外创建表`

- 你可以从另一个MySQL实例来导入一个单文件表。请查看 `第15.6.1.3节 导入InnoDB表`。

- 当文件表空间类型中创建的表同时支持`DYNAMIC`和`COMPRESSED`行格式，而系统表空间是不支持的。请查看`第15。10节 InnoDB行格式`。

- 在发生错误时，或备份和bin log不可用时，或MySQL实例无法重启时，存储在不同表空间数据文件中的表能够节省恢复的时间或提高恢复的成功率。

- 在不影响其他InnoDB表的使用时，你可以使用MySQL的企业版备份来备份或恢复单文件表空间中创建的表。这有助于在多个备份清单中的表或需要较少备份频率的表。请查阅`执行部分备份`一节来获得更多详细内容。


- 单文件表空间可以利用监控文件系统中数据文件的大小来监控表数据的大小。

- 当`innodb_flush_method`设置为`O_DIRECT`时，标准的Linux文件系统不允许对类似共享表空间这样的单个数据文件进行并发写入。结果就是，使用单文件表在同样的设置下可能会有性能上的提升。

- 共享表空间的大小受到表空间64TB上限的限制。作为对比，每一个单文件表空间都可以达到64TB的上限，从而为每一个表数据提供了足够的增长空间。


### 单文件表空间的缺点

作为和共享表空间（例如系统表空间和通用表空间）对比，单文件表空间有如下的不足：

- 单文件表每一张表可能都有存在只有表自己能够使用的空间，如果管理不当这可能会导致空间的浪费。

- `fsync`操作需要在很多的单文件表数据文件上执行，而相对一个共享的表空间数据文件来说只有一个数据文件。由于`fsync`操作是针对每个文件的，多张表的写操作无法合并，最终导致系统上有非常多的`fsync`操作。

- **mysqld** 必须为每一个单文件表空间维持一个打开的文件句柄，如果你有非常多的单文件表空间中表的话可能会影响性能。

- 每张表都有它自己的数据文件因此需要更多的文件描述符。

- 可能会造成更多的碎片，从而影响`DROP TABLE`和表扫描的性能。然而，如果管理好碎片的话，单文件表空间能够提升这些操作的性能。

- 单文件表空间中的表被删除是需要扫描buffer pool，对于一个庞大的buffer pool这可能需要花费好几秒钟。这种扫描需施加一个大范围的内部锁之下，有可能会阻塞其他的操作。

- `innodb_autoextend_increment`变量决定自增的共享表每次自增的大小，但这个配置不会应用到单文件表的数据文件，已初始化的单文件表会以一个小的尺寸来增长，以4MB的增量增长。