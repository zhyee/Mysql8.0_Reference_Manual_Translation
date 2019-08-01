## 15.6.1.2 移动和复制InnoDB表

> 原文地址： [https://dev.mysql.com/doc/refman/8.0/en/innodb-migration.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-migration.html)

这一小节讲解移动或拷贝部分或全部InnoDB表到另一个服务器或实例的技术。例如，你可能需要把一个MySQL实例迁移到一个更大更快的服务器上；你也可能会克隆一个完整的数据库实例到一个MySQL从库上；你可能会拷贝单独的几张表到另一个实例来开发或测试一个应用，或者导入到数仓中来生成报告。

在Winwods上，InnoDB总是以小写字母来存储数据库和表名。为了从Unix迁移一个二进制格式的数据库到Windows或者反过来，你应该总是使用小写字母来命名数据库和表。达到此目的一种简便方法是在创建表之前添加如下配置项到my.cnf或者my.ini配置文件中的[mysqld]配置段。

```
[mysqld]
lower_case_table_names=1
```
> **Note**
> 禁止使用和初始化服务器时不同的`lower_case_table_names`值来启动服务器。

移动和拷贝InnoDB表的技术包括以下：

- 可传输的表空间

- MySQL商业版备份

- 复制数据文件（冷备份）

- 导出和导入（mysqldump）

##### 可传输的表空间

可传输的表空间特性主要使用`FLUSH TABLES ... FOR EXPORT`来让一张InnoDB表准备从一个数据库实例拷贝到另一个。为了使用这个特性，InnoDB表创建时必须打开`innodb_file_per_table`选项以便每张InnoDB表都有它自己的表空间。为了解更多使用的相关信息，请查看`15.6.3.7节 拷贝表空间到另一个实例上`。

##### 商业版MySQL备份

MySQL企业版备份产品能够让你在生成一个一致性读快照时对数据库操作造成最少干扰的情况下来备份一个正在运行的数据库。当MySQL企业版备份正在拷贝表时，能够继续响应读写操作。而且，MySQL企业版备份能够压缩备份文件和备份表的部分数据。和MySQL的bin log结合，你能够执行某一时间点的备份。MySQL企业版备份作为MySQL企业版订阅的一部分。
更多有关企业版的信息，请查看`第30.2节 MySQL企业版版备份工具总览`。

##### 拷贝数据文件（冷备份）

你可以通过拷贝`第15.17.1节 InnoDB 备份`中`Cold Backups`中所列的所有相关文件来迁移一个数据库。

InnoDB数据和日志文件在所有相同浮点数格式的平台上都是二进制兼容和。如果平台间浮点数格式不同但你没有使用FLOAT和DOUBLE数据类型的话，处理方式是相同的：简单拷贝相关的文件。

当你移动或拷贝单文件表的.ibd 文件时，目标数据库文件夹名必须和源数据库相同。存储在InnoDB共享空间的表定义包含数据库名。存储在表空间的事务号和日志序列号在数据库间同样也是不一样的。

为了移动一个 .ibd 文件和关联的表到另一个数据库，使用 `RENAME TABLE`:
```
RENAME TABLE db1.tbl_name TO db2.tbl_name;
```
