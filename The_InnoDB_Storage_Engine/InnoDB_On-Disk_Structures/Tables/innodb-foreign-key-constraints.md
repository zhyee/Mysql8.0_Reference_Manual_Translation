## InnoDB的外键约束

> 原文地址： [https://dev.mysql.com/doc/refman/8.0/en/innodb-foreign-key-constraints.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-foreign-key-constraints.html)

这一章节会介绍InnoDB存储引擎如何处理外键约束，主要包括以下几个主题：

- 外键的定义

- 关联操作

- 外键在`Generated Column`和虚拟索引上的使用限制

对于外键如何使用的信息和例子，请查看`第13.1.20.6节 使用外键约束`。

#### 外键定义

InnoDB表的外键定义符合以下条件：

- InnoDB允许外键关联任何有索引的一列或多列。但是，被关联的表中必须保证被关联的列是索引的第一列。InnoDB添加到索引里的隐藏列也算作是第一列（请查看`第15.6.2.1节 聚簇索引和二级索引`）。

- InnoDB不支持为用户自定义分区的表定义外键。这意味这只有非用户自定义分区的InnoDB表存在外键或者被外键索引的列。

- InnoDB允许外键关联一个非唯一键。这是InnoDB对标准SQL的扩展。

#### 关联操作

InnoDB表外键的关联行为受下列行为影响：

- 如果MySQL服务器允许SET DEFAULT，会被InnoDB拒绝生效。InnoDB不允许`CREATE TABLE`和`ALTER TABLE`语句使用此语法。

- 如果父表中有相同的被关联的索引的数据行，InnoDB执行外键检测就好像父表中拥有相同索引值的其他行不存在。例如，你已经定义了一个RESTRICT类型约束，并且一个子数据行有多个父数据行，InnoDB不允许删除这些父数据行中的任意一行。

- InnoDB通过深度优先算法执行级联操作，基于外键相关的索引中的记录。

- 如果ON UPDATE CASCADE或者ON UPDATE SET NULL在级联操作时递归的更新它之前已经更新的表，它表现的如同RESTRICT。这意味着你不能使用关联本身的ON UPDATE CASCADE或ON UPDATE SET NULL操作。这样处理是为了避免级联更新时无限循环。另一方面，关联自身的ON DELETE SET NULL有可能和ON DELETE CASCADE。级联操作不能嵌套操作15层。

- 和MySQL的惯例一致，一个SQL语句增删改很多行数据，InnoDB会逐行检查唯一键和外键约束。当执行外键约束时，InnoDB对它已经扫描到的子数据行或父数据行施加行级的共享锁。InnoDB会即时检查外键约束，而不是推迟到事务提交。根据SQL标准，默认的行为应该要延迟检查。即当整个SQL语句已经执行完之后再做约束见检查。在InnoDB实现延迟的约束检查之前，一些事还无法实现，例如使用外键关联自己的删除记录操作。

#### 外键在`Generated Column`和虚拟索引上的使用限制

- 在一个`stored generated column`上外键不能使用CASCADE，SET NULL，或SET DEFAULT as ON UPDATE进行关联操作，也不能使用SET NULL或者SET DEFAULT as ON DELETE关联操作。

-  外键在`stored generated column`的源字段上限制使用CASCADE，SET NULL，或者SET DEFAULT as ON UPDATE 或 ON DELETE 的关联操作。

- 外键无法关联`virtual generated column`。

- MySQL8.0之前，外键无法关联建立在`virtual generated column`列上的二级索引。
