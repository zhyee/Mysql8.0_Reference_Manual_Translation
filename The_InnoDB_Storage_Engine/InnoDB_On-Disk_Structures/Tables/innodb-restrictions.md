## 15.6.1.6 InnoDB表的限制

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-restrictions.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-restrictions.html)

这一小节将从以下三个主题来讨论InnoDB表的制约：

- 最大最小值

- InnoDB表的限制

- 锁和事务

> **Warning**
> 在NFS（网络文件系统）上使用InnoDB，回顾一下`Using NFS with MySQL`一节中概述的潜在可能出现的问题


#### 最大最小限制

- 一张表最多能容纳1017个列，虚拟列也算在内。

- 一张表最多可以设置64个二级索引。

- 使用`DYNAMIC`或者`COMPRESSED`行格式的InnoDB表中的索引前缀长度限制在3072个字节以内。

使用`REDUNDANT`或`COMPACT`行格式的InnoDB表的索引前缀长度限制在767字节以内。例如，假设有一个utf8mb4字符集的TEXT或VARCHAR列，如果该列中存储了超过191个字符且每个字符都是4字节，这时在该列上创建索引可能就会达到这个上限。

试图创建索引时超过了这个限制会返回错误。

索引使用的字段前缀长度限制同样适用于全字段的索引。

- 如果在创建MySQL实例时你通过指定`innodb_page_size`配置选项来减少InnoDB page size 到8KB或者4KB，索引的最大长度也会按比例的减少，3072字节的上限是基于16KB的innodb_page_size。也就是说，当page的大小是8KB时，索引长度上限是1536个字节，当page大小是4KB时，索引的长度上限相应的变为768个字节。

- 最多允许建立16个列的多列索引，超过限制会返回错误。

```
ERROR 1070 (42000): Too many key parts specified; max 16 parts allowed
```

- 行的最大长度，不包括可变长度的列（VARBINARY,VARCHAR和TEXT），对于4KB，8KB，16KB，32KB的page来说稍小于page大小的一半。例如，对于默认的16KB的innodb_page_size最大的行长度差不多是8000个字节。但是，对于64KB的InnoDB page大小来说，最大的行长度大约是16000字节。LONGBLOB和LONGTEXT列必须小于4GB，且整个行的长度，包括BLOB和TEXT列，必须小于4GB。

如果一行数据小于半个page的长度，则整行数据都存储在这个page里，如果超过了半个page，则可变长度列选择存放在外部的非page的存储上直到行的长度缩小到半个page以内，正如`第15.11.2节 文件空间管理`所介绍的那样。

- 尽管InnoDB内部支持行的长度大于65535字节，MySQL本身强制一个行所有列的长度之和为65535：

```sql
mysql> CREATE TABLE t (a VARCHAR(8000), b VARCHAR(10000),
    -> c VARCHAR(10000), d VARCHAR(10000), e VARCHAR(10000),
    -> f VARCHAR(10000), g VARCHAR(10000)) ENGINE=InnoDB;
ERROR 1118 (42000): Row size too large. The maximum row size for the
used table type, not counting BLOBs, is 65535. You have to change some
columns to TEXT or BLOBs
```

查看`第C.10.4节 表列数和行大小限制`。

- 在一些较老的操作系统上，文件大小被限制在2GB以内。这不是InnoDB本身的限制，但是如果你需要一个更大的表空间，配置使用几个较小的数据文件来取代一个大文件。

- InnoDB log 文件的大小最大可以支持512GB。

- 最小表空间大小稍微大于10MB，最大表空间大小依据InnoDB page size。

##### 表15.3 InnoDB表空间大小上限

| InnoDB Page 大小 | 表空间大小上限 |
|:-------- |:-------- |
| 4KB | 16TB |
| 8KB | 32TB |
| 16KB | 64TB |
| 32KB | 128TB |
| 64KB | 256TB |

最大表空间大小也是一张表的最大上限。

- 表空间文件路径，包括文件名称，不能超过Windows的最长路径限制。Windows10之前，最长路径限制是260个字节。对于Windows10版本1607而言，移除了Win32文件和路径的最长路径限制，但是你必须启用这个特性。

- InnoDB默认的页大小是16KB。创建MySQL实例时你可以通过配置innodb_page_size选项来增加或减少page大小。

InnoDB支持32KB和64KB的page大小，但是ROW_FORMAT是COMPRESSED的情况下是不支持页的大小超过16KB的。当innodb_page_size=32KB，扩展的大小是2MB，当innodb_page_size=64KB，扩展大小是4MB。

使用特殊的InnoDB page页大小的MySQL实例无法使用不同的page大小的MySQL实例的数据文件和日志文件。

#### InnoDB表的限制

- `ANALYZE TABLE`通过在每个索引树上执行`random dives`并且相应的更新索引基数评估值来显示索引的基数（和`SHOW INDEX`显示输出的Cardinality列一样）。由于仅仅是评估值，重复执行`ANALYZE TABLE`可能会产生不同的值。这使得`ANALYZE TABLE`在InnoDB表上运行速度很快，但却不是100%精确的，因为它没把所有的行纳入统计。

你可以打开`innodb_stats_persistent`配置选项让`ANALYZE TABLE`收集的统计值更加的精确和稳定，如`第15.8.10.1节，配置持久性的统计参数`。当打开设置时，当索引列的数据有较大改变时重新运行`ANALYZE TABLE`是非常重要的，因为统计不会周期性的更新（例如服务器重启之后）。

如果打开了持续统计设置，你可以通过修改`innodb_stats_persistent_sample_pages`系统变量来更改random dives数量。如果持续统计设置是关闭的，则修改`innodb_stats_transient_sample_pages`系统变量来代替。

MySQL使用索引基数评估值来优化join查询。如果join查询没有正确的被优化，可以尝试使用`ANALYZE TABLE`。少数情况下`ANALYZE TABLE`无法为你特定的表产生足够的值，你可以使用FORCE INDEX让你的查询强制使用一个特定的索引，或者设置`max_seeks_for_key`系统变量来确保MySQL更倾向于使用索引查找而非表扫描。查看`第B.4.5节 关于优化的问题`

- 如果一个查询语句或者事务正在执行，并且`ANALYZE TABLE`在同一张表执行,紧跟着再执行一个`ANALYZE TABLE`操作，第二个`ANALYZE TABLE`操作将会一直被阻塞直到查询或事务结束。出现这种现象是由于当`ANALYZE TABLE`执行结束后会标记当前已加载的表定义是过时的。新的语句和事务（包括第二个`ANALYE_Tables语句`）必须再重新加载表定义到缓存中，并且在当前执行的语句或事务完成之前和老的表定义被清除之后才可以执行。并发加载表定义是不被支持的。

- `SHOW TABLE STATUS` 除掉表保留的物理大小外无法给出InnoDB表的精确的统计，行数总计仅仅是一个用于SQL优化的粗略的预估值。

- InnoDB内部不会存储一张表的行数，因为并发的事务同一时间可能会“看”到不同的行数。因此，SELECT COUNT(\*) 语句仅仅会统计对当前事务可见的行数。

更多关于InnoDB如何执行`SELECT COUNT(*)`语句，请参考`第12.20.1节， 聚合（GROUP BY）函数介绍`中关于`COUNT()`的介绍。

- 在Windows上，InnoDB内部总是存储数据库名和表名的小写形式。为了把二进制格式的数据库从Unix迁移到Windows或者从Windows迁移到Unix，全部使用小写来命名数据库和表名。

- 一个自增列ai_col必须定义在一个索引当中以便InnoDB能够执行等价于“SELECT MAX(ai_col)”语句的查询来获得自增列当前的最大值。这通常是把自增列定义为某些索引的首列来实现。

- InnoDB在初始化一张表中事先指定的自增列时，会对自增列相关联的索引的末端施加一个排它锁。

当`innodb_autoinc_lock_mode=0`时，InnoDB使用一个特殊的AUTO-INC表锁模式，当访问自增计数器时，一旦获得这个AUTO-INC锁会一直持续到语句执行结束。当有客户端持有这个AUTO-INC锁时，其他的客户端无法向表中插入数据。当`innodb_autoinc_lock_mode=1`时处理“bulk inserts”也会表现出相同的行为。当`innodb_auto_lock_mode=2`时不会使用表级的AUTO-INC锁。更多信息请查看`第15.6.1.4节 InnoDB自增的处理`。

- 当一个自增整型字段超出了值的范围，随后的插入操作会返回一个重复key的错误，这是MySQL的一般处理方式。

- DELETE FROM tb1_name 不会重新生层一张表，而是一行一行的来删除所有行。

- Cascaded外键操作不会激活触发器。

- 在创建表的时候不能使用InnoDB内部列的名称（包括 DB_ROW_ID，DB_TRX_ID，DB_ROLL_PTR和DB_MIX_ID）。此限制不管大小写都会生效。

```sql
mysql> CREATE TABLE t1 (c1 INT, db_row_id INT) ENGINE=INNODB;
ERROR 1166 (42000): Incorrect column name 'db_row_id'
```

#### 锁定和事务

当`innodb_table_locks=1`（默认值）时，`LOCK TABLES`会对每张表施加两个锁。除了在MySQL层施加一个表锁外，还会施加一个InnoDB的表锁。MySQL4.1.2之前的版本不会施加InnoDB表锁；老的处理方式可以通过设置`innodb_table_locks=0`来达到。如果不施加InnoDB表锁，即使一些记录被其他的事务锁定了，`LOCK TABLES`操作也会执行成功。

在MySQL 8.0版本中，`innodb_table_locks=0`对于使用`LOCK TABLES ... WRITE`显式施加表锁的语句是不起作用的。对于隐式加锁（例如通过触发器）， `innodb_table_locks=0`不会对`LOCK TABLES ... WRITE`或`LOCK TABLES ... READ`语句起作用。

- 一个事务持有的所有锁在事务提交或者退出时释放。因此，没必要过多关注当`autocommit=1`模式时在InnoDB表上使用`LOCK TABLES`，因为施加的InnoDB表锁会立即释放。

- 你无法在事务运行到一半的时候来增加锁定一些表因为`LOCK TABLES`是隐式执行的。
