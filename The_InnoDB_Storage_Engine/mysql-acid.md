## 15.2 InnoDB和ACID模型

> 原文地址：https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html

ACID模型是一套增强数据库可靠性的设计原则，而这对于商业数据和关键业务的应用程序来说是非常重要的。MySQL数据库拥有像InnoDB存储引擎类似的基本遵循ACID模型的组件，因此遇到软硬件故障时不会导致数据错乱或者返回错误的结果。当你使用遵循ACID的特性时，你不需要重复造轮子来进行一致性检验或者自己实现故障恢复机制。如果你有额外的保障机制，或者高可用的硬件，抑或是可以容忍一小部分数据丢失或不一致的情况下，你可以调整MySQL的配置降低部分ACID可靠性来换取更高的性能和吞吐量。

接下来的内容讨论MySQL尤其是InnoDB存储引擎的特性是如何与ACID模型关联起来的：

- *A*：原子性(atomicity)
- *C*：一致性(consistency)
- *I*：隔离性(isolation)
- *D*: 持久性(durability)

### Atomicity

ACID模型中的原子性主要跟InnoDB的事务相关，相关的MySQL特性包括：

- 自动提交的设置。
- `COMMIT`语句。
- `ROLLBACK`语句。
- 从`INFORMATION_SCHEMA`表中可操作的数据。

### Consistency

ACID原则中一致性主要体现在InnoDB内部处理数据在意外宕机时对数据的保护，相关的MySQL特性包括：

- InnoDB双写缓冲。
- InnoDB故障恢复。

### Isolation

ACID原则中的隔离性体现在InnoDB的事务上，尤其是应用到每个事务上的隔离级别。相关的MySQL特性包括：
- 自动提交设置。
- `SET ISOLATION LEVEL`语句
- 行锁的相关的内容。在MySQL性能调优时，你可以从`INFORMATION_SCHEMA`表中看到这些详细信息。

### Durability

ACID原则中的持久性体现在和你硬件配置相关的MySQL特性上。由于持久性和CPU、网路以及存储设备的好坏息息相关，这方面也是最难提供指导方案的（并且这些指导方案可能意味着购买新的硬件）。相关的MySQL特性包括:

- InnoDB双写缓冲(`doublewrite buffer`)，通过`innodb_doublewrite`配置选项来打开或关闭此功能。
- `innodb_flush_log_at_trx_commit`配置项。
- `sync_binlog`配置项。
- `innodb_file_per_table`配置项。
- 存储设备上的写缓冲，例如硬件、SSD、RAID阵列。
- 配有备用电池的缓存设备。
