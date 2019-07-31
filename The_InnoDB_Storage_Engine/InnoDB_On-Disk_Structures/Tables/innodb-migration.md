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
