## 15.1 InnoDB简介

> 原文地址：https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html

- [15.1.1 使用InnoDB存储引擎的优势](Introduction_to_InnoDB/innodb-benefits.md)
- [15.1.2 InnoDB存储引擎最佳实践](Introduction_to_InnoDB/innodb-best-practices.md)
- [15.1.3 验证你的默认存储引擎是否是InnoDB](Introduction_to_InnoDB/innodb-check-availability.md)
- [15.1.4 对InnoDB进行测试和基准测试](Introduction_to_InnoDB/innodb-benchmarking.md)

InnoDB是一个兼顾了高可用和高性能的存储引擎，在MySQL8.0里，它是默认的存储引擎。如果你没有配置一个其他的存储引擎作为默认的存储引擎，那么省略了`ENGINE=`的`CREATE TABLE`建表语句默认创建的就是InnoDB表。

### InnoDB的主要优势

- 它的DML操作遵循ACID模型，利用可提交和可回滚的事务以及故障恢复能力来保护用户的数据，具体请查阅 “第15.2节 InnoDB和ACID模型”。
- 行级锁和ORACLE风格的一致性读增强了多用户并发能力和性能，具体请查阅 “第15.7节 InnoDB的锁和事务模型”。
- InnoDB表在磁盘上的数据组织方式优化了基于主键的查询，每张InnoDB表都拥有一个称之为“聚簇索引”的主键索引，
这种组织数据的方式让主键查找消耗最少的IO性能。详情请查阅 “第15.6.2.1节 聚簇索引和非聚簇索引”
- 为了保证数据的完整性，InnoDB引擎支持外键约束。在有外键的情况下，增删改操作都会被检查以确保数据在多个表之间保持一致，详情请查阅 “第15.6.1.5节 InnoDB的外键约束”。

### 表15.1 InnoDB存储引擎特性

|  **特性** |  **是否支持** |
| ------------ | ------------ |
| B-tree 索引  |  是 |
|  备份/按时间点恢复（server层实现，非存储引擎层） | 是  |
|  分布式数据库支持 |  否 |
|  聚簇索引 | 是  |
| 数据压缩  | 是  |
|  数据缓存 | 是  |
| 数据加密  | 是（在server层通过加密函数来实现，5.7及以上版本，支持data-at-rest tablespace加密）  |
|  外键支持 | 是  |
| 全文索引  | 是（InnoDB在MySQL5.6及以上版本支持）  |
|  地理空间数据类型支持 | 是  |
|  地理空间索引支持 | 是（InnoDB在MySQL5.7及以上版本支持）  |
| hash索引  | 是  |
| 索引缓存  | 是  |
|  锁粒度 | 行级  |
|  MVCC | 是  |
| 主从复制  | 是（在server层实现，而不是存储引擎层）  |
|  存储容量限制 | 64TB  |
| T-tree索引  | 否  |
|  事务 | 是  |
|  数据字典统计更新 | 是  |


如果想对比InnoDB和MySQL其他的存储引擎，可以查看 “第16章 其他存储引擎” 中的存储引擎特性表。

### InnoDB的功能增强和新特性

如果想详细了解InnoDB的功能增强以及新特性，可以参阅：
- “第1.4节 MySQL8.0新特性”中的InnoDB 功能增强清单
- 版本说明

### 其他InnoDB相关的信息和资源

- InnoDB相关的术语和定义，请参考附录。
- InnoDB存储引擎专门的论坛，请访问 [MySQL Forums::InnoDB](http://forums.mysql.com/list.php?22)。
- InnoDB使用和MySQL相同的GNU GPL2（1991.06）开源协议发布，有关协议的信息，请访问 [http://www.mysql.com/company/legal/licensing/](http://www.mysql.com/company/legal/licensing/)。
