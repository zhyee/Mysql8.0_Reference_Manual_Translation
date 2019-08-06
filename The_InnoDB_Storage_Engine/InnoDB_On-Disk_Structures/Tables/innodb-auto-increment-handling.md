## 15.6.1.4 InnoDB中的自增处理

> 原文地址： [https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)

InnoDB提供了一个可配置的锁机制，该机制能够有效提升向拥有自增列的表中新增数据时的SQL语句的可靠性和性能。为了在InnoDB表中使用自增机制，一个自增列必须被定义为索引的一部分以便可以在表中使用类似`SELECT MAX(ai_col)`的查询来获取列的最大值。一般来说，通过定义自增列为索引的首列来实现。

这一节介绍`AUTO_INCREMENT`锁定模式的行为，不同的`AUTO_INCREMENT`锁定模式设置，以及InnoDB如何初始化 `AUTO_INCREMENT`的计数器。

- InnoDB AUTO_INCREMENT 锁定模式

- InnoDB AUTO_INCREMENT 锁定模式的使用影响

- InnoDB AUTO_INCREMENT 计数器初始化

介绍 `innodb_autoinc_lock_mode`的设置会涉及下面的这些名词：

- "INSERT-like"语句

在一个表中会生成新记录的所有语句，包括`INSERT`,`INSERT ... SELECT`,`REPLACE`,`REPLACE ... SELECT` 和 `LOAD DATA`。包括“simple-inserts”，“bulk-inserts”和“mixed-mode inserts”。

- "Simple inserts"

指的是插入的数据行数是能够提前确定的语句（当对语句进行最初处理的时候）。这包括单行和多行没有嵌套子查询的`INSERT`和`REPLACE`语句，但`INSERT ... ON DUPLICATE KEY UPDATE` 除外。

- "Bulk inserts"

插入数据的行数（以及需要自增长的数值）是无法提前预知的语句。这包括`INSERT ... SELECT`,`REPLACE ... SELECT`和 非简单INSERT的`LOAD DATA`语句。InnoDB每次只分配给AUTO_INCREMENT列一个值，就像是一行一行的处理一样。

- "Mixed-mode inserts"

对新插入数据中部分自增列指定了明确的值的“simple inserts”语句，例如下面这个例子，其中c1是表t1的自增列：

> INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');

另一种"mixed-mode inserts"是`INSERT ... ON DUPLICATE KEY UPDATE`，最坏的情况下实际上是`INSERT`之后立即`UPDATE`,因此给自增列分配的值在更新阶段也许用得上也许用不上。

可以为`innodb_autoinc_lock_mode`配置参数设置三个值：0，1和2，分别代表“传统”、“连续”、“穿插”锁定模式。对于MySQL8.0来说，穿插锁定模式（innodb_autoinc_lock_mode=2）是默认设置。8.0以前的版本,连续锁定模式是默认值（innodb_autoinc_lock_mode=1）。

MySQL8.0默认的穿插锁定模式也反映在8.0把默认基于语句的复制改为基于行的复制。基于语句的复制需要连续自增锁定模式来确保自增值的分配符合给定SQL语句的可预见或可重复的执行顺序，而基于行的复制对SQL语句的执行顺序不敏感。

- innodb_autoinc_lock_mode = 0 ("traditional"锁定模式)

“traditional”锁定模式是和之前MySQL5.1中的`innodb_autoinc_lock_mode`配置参数的行为保持一致。提供传统的锁定模式主要是为了向后兼容，性能测试和绕过因可能在含义上存在差异而由“mixed-mode inserts”造成的问题。

在这种锁定模式下，所有的“INSERT-like”语句都会为插入数据到带有自增列的表时获得一个表级 AUTO-INC 锁。这种锁一般会持有到语句执行结束（不是事务的结束）来确保自增值的分配用一种符合给定INSERT语句预计的和可重复的顺序。

在基于语句的复制的情况下，这意味着当一个SQL语句同步到一个从数据库时，会使用和主库相同的自增值。执行多个`INSERT`语句的结果是不可控的，并且从库会再次生成和主库相同的数据。如果由多个`INSERT`语句所产生的自增值是交错的，两个并发的`INSERT`语句产生的结果是存在不确定性的，且不能可靠的使用基于语句的复制来从主库同步数据到从库。

为了清楚的解释这点，试想下面这样一个例子：

```sql
CREATE TABLE t1 (
  c1 INT(11) NOT NULL AUTO_INCREMENT,
  c2 VARCHAR(10) DEFAULT NULL,
  PRIMARY KEY (c1)
) ENGINE=InnoDB;
```

假设有两个事务正在运行，每一个都正在向带有自增列的表插入数据。其中一个事务使用`INSERT ... SELECT`语句来插入1000行数据，另一个事务使用简单的`INSERT`语句插入一行数据：

```sql
Tx1: INSERT INTO t1 (c2) SELECT 1000 rows from another table ...
Tx2: INSERT INTO t1 (c2) VALUES ('xxx');
```

InnoDB无法预知从Tx1事务中的`SELECT`能够查询到多少条数据，它会和语句的执行一样一次分配一个自增值。由于会持有表级锁到一条语句执行结束，所以同一时间t1仅能有一个`INSERT`语句在执行，且由不同语句产生的自增值是不会交错生成的。由事务Tx1的`INSERT ... SELECT`语句产生的自增值是连续的，而由事务Tx2的`INSERT`语句引起的单个自增值有可能大于或者小于这些由事务Tx1生成的自增值，这依赖于语句的执行先后。

只要SQL语句以bin log相同的顺序执行（在基于语句的复制或者数据恢复的场景下），从库上产生的结果就会和事务Tx1和Tx2在主库上首次运行的结果一致。因此，持有表级锁到语句执行结束使得基于语句的复制场景中的`INSERT`语句自增是安全的。但是，这种表级锁限制了多个事务同事执行插入语句是的并发能力和扩展性。

在上面的例子中，如果不使用表级锁，事务Tx2中插入数据的自增值严格根据它执行的时间。如果事务Tx2中的`INSERT`语句正好在事务Tx1中的`INSERT`语句执行时执行（既不是之前也不是之后），两个`INSERT`语句执行所产生的自增值是不确定的，而且每次运行可能结果都会不一样。

在连续锁定模式下，InnoDB能够避免为能预知数据行数的“simple inserts”语句使用表级的AUTO-INC锁，并且能够为基于语句的复制保持确定性和安全性。

如果你不是使用重复执行bin log的SQL语句来恢复或者复制，穿插锁定模式能够用来消除表级的AUTO-INC锁的弊端从而实现更高的并发性能，缺点是允许分配的自增值之间存在间隙和并发执行的语句产生的自增值之间有互相穿插的可能性。


- innodb_autoinc_lock_mode = 1 (“连续”锁定模式)

在这种模式下，“Bulk inserts”使用特殊的AUTO-INC表级锁直到语句执行结束。这被应用到所有`INSERT ... SELECT`，`REPLACE ... SELECT`和`LOAD DATA`语句。同一时间仅仅只有一个语句能够持有AUTO-INC锁。如果批量插入操作的源表和目标表之间是不同的，对源表中查出的第一行数据施加一个共享锁之后再会对目标表施加AUTO-INC锁。如果批量插入的源表和目标表是同一张表，在对所有查出的数据施加了共享锁之后再施加AUTO-INC锁。

“Simple inserts”（插入的数据行数是能够提前预知的）在一个排它锁（一个轻量级的锁）的控制下获取所需要的自增值，从而避免了表级的AUTO-INC锁，该锁仅仅在分配自增值的处理时施加，而不是一直到语句执行结束。除非是另一个事务施加了一个AUTO-INC锁，否则不会使用AUTO-INC锁。如果另一个事务持有一个AUTO-INC锁，一个“simple inserts”会等待这个AUTO-INC锁，好像它是一个“bulk insert”一样。

一般来说这种锁定模式确保了在无法提前预知插入数据的行数（和在语句执行时分配自增值）时，所有由“INSERT-like”语句分配的自增值都是连续的，并且基于语句的复制是安全的。

简单来说，就是这种锁定模式能够在安全的进行语句的主从复制下显著提高扩展性。进一步来说，如同“traditional”锁定模式，任何语句分配的自增值都是连续的。和所有语句在“traditional”锁定模式下相比含义上是没有变化的，除了有一个重要的不同点。

这个重要区别是关于“mixed-mode inserts”的，当用户对一个多行插入的“simple insert”中部分但不是全部自增值显式赋值时。在这样的场景下，InnoDB会分配比实际插入的行数更多的自增值。所有值都是基于最近执行的语句所产生的自增值之后自动连续分配的，“超过”的数字会丢失。

- innodb_autoinc_lock_mode = 2（“穿插”锁定模式）

在这种锁定模式下，任何“INSERT-like”语句都不使用表级AUTO-INC锁，并且多个语句可以同时执行。这是最快和最可扩展的锁定模式，但是对于基于语句的复制和从bin log中重新执行sql来恢复的场景是不 **安全** 的。

在这种锁定模式下，自增值能够在并发执行“INSERT-like”语句时被保证唯一并且是单调递增的。由于多个语句能够在同一时间产生数字（也就意味着，产生的数据在不同的语句之间会互相穿插），任何执行的语句所产生的自增值都可能是不连续的。

如果执行的语句都是提前能够预知插入的行数的“simple inserts”，“mixed-mode inserts”除外，对每一个语句所产生的自增值之间都是没有间隙的。但是，当执行“bulk inserts”时，任何执行的语句产生的自增值都是可能存在间隙的。

#### 使用不同InnoDB自增锁定模式的影响

- 主从复制中使用自增

如果你正在使用基于语句的主从复制，把`innodb_autoinc_lock_mode`设置为0或1并且在主从服务器之间使用相同的配置值。如果你把`innodb_autoinc_lock_mode`的值设为2或者主从服务器之间使用不同的锁定模式，自增值则无法保证在主从之间保持一致。

如果你是使用基于行或者混合格式的主从复制，则所有的自增锁定模式都是安全的，由于基于行的主从复制对SQL语句的执行顺序并不敏感（混合格式对不安全的基于语句的复制会换成使用基于行的复制）。

- 自增值的“缺失”和自增序列的间隙

在所有的自增锁定模式下（0，1和2），如果一个生成了自增值的事务回滚，那么这些自增值就会“丢掉”。一旦为一个自增列分配了一个自增值，它就不能回滚了，不管这个“INSERT-like”是否执行完成，或者语句所在的事务回滚。这种丢掉的值都不会再被重用。因此，一张表的自增列的值之间可能会存在间隙。

- 为自增列指定NULL或0值

在所有的自增锁定模式下（0，1或2），如果用户为自增列指定了NULL或者0，那么InnoDB会把该列没有指定值来对待并为之生成一个新值。

- 为自增列分配一个负数值

在所有的自增锁定模式下（0，1或2），如果你给自增列指定了一个负数值，那么自增机制的处理是不确定的。

- 自增值已经超过了所指定整形类型的最大值上限

在所有的自增锁定模式下（0，1或2），如果自增值已经超过所指定的数据类型所能存储的最大值的时候，此时自增机制的行为是不确定的。

- “bulk inserts”时自增值之间间隙

当 `innodb_autoinc_lock_mode`的值设置为0（“传统锁定模式”）或 1（“连续锁定模式”）,任何语句生成的自增值都是连续的，之间没有空隙，因为在语句执行期间一直会施加表级的AUOTO-INC锁直到语句执行结束，此时只可能有一个语句在执行。

当`innodb_autoinc_lock_mode`的值设置为2（“穿插锁定模式”）,此时执行“bulk inserts”语句自增值之间有可能会有间隙，但只有在同时并发执行“INSERT-like”语句时才有可能发生。

对于锁定模式是1或2的情况下，在连续执行的语句之间自增值是可能存在间隙的，因为对于每个“bulk inserts”语句所需要的实际自增值数量是无法提前预知的，所以有可能会比所需的自增值数量多分配了一些。

- “mixed-mode inserts”分配的自增值

考虑“mixed-mode insert”,一种只给插入的部分数据行（不是全部）指定了自增值的“simple inserts”。这样的语句在锁定模式0，1和2 之间的行为都是不一样的。例如，假设c1是表t1的自增列，并且最近生成的自增值是100。

```sql
mysql> CREATE TABLE t1 (
    -> c1 INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    -> c2 CHAR(1)
    -> ) ENGINE = INNODB;
```

现在，考虑下面的“mixed-mode insert”语句：

```sql
mysql> INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');
```

当`innodb_autoinc_lock_mode`设置为0（“traditional”），插入的4行数据是：

```sql
mysql> SELECT c1, c2 FROM t1 ORDER BY c2;
+-----+------+
| c1  | c2   |
+-----+------+
|   1 | a    |
| 101 | b    |
|   5 | c    |
| 102 | d    |
+-----+------+
```

因为自增值的分配一次分配一个值而不是在语句开始执行时立即分配所有值，所以下一个可用的自增值是103，不管当前是否在并发执行“INSERT-like”语句（任意类型的）这个结论都成立。

当`innodb_autoinc_lock_mode`的值设为1时（“consecutive”），插入的四个新行数据仍是：

```sql
mysql> SELECT c1, c2 FROM t1 ORDER BY c2;
+-----+------+
| c1  | c2   |
+-----+------+
|   1 | a    |
| 101 | b    |
|   5 | c    |
| 102 | d    |
+-----+------+
```

但是，在这种情况下，下一个自增值是105而不是103，因为在语句执行的时候分配了4个自增值但只使用了其中的两个。不管当前是否在并发执行任意的“INSERT-like”语句该结论都成立。

当 `innodb_autoinc_lock_mode`的值设置为2时（“interleaved”），插入的4行新数据是：

```sql
mysql> SELECT c1, c2 FROM t1 ORDER BY c2;
+-----+------+
| c1  | c2   |
+-----+------+
|   1 | a    |
|   x | b    |
|   5 | c    |
|   y | d    |
+-----+------+
```

x和y的值是唯一的并且大于任何之前创建的值。然而，x和y的具体值依赖于同一时间并发执行的语句所创建的值。

最后，考虑下面的语句，当最近执行语句产生的自增值是100：

```sql
mysql> INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (101,'c'), (NULL,'d');
```

对于任意的`innodb_autoinc_lock_mode`设置值，这个语句都会引发一个duplicate-key 23000的错误码（Can't write;duplicate key in table），由于101已经被分配给数据行（null, 'b'）然后插入数据行（101， 'c'）就失败了。

- 修改一组`INSERT`语句中间的自增值

在 MySQL5.7以及之前的版本中，修改一组`INSERT`语句中间的自增值会导致`Duplicate entry`错误。例如，你执行一个`UPDATE`操作把一个自增列的值更新为比当前所有自增值还要大的值，随后没有指定一个未使用的自增值的`INSERT`操作会引发`Duplicate entry`错误。在MySQL8.0及以后版本，如果你更新一个自增列的值到一个比当前所有值还要大的值，新值将会被保持下来，随后的`INSERT`操作分配的自增值将从这个更大的新值开始。下面用一个例子来验证这种行为。

```sql
mysql> CREATE TABLE t1 (
    -> c1 INT NOT NULL AUTO_INCREMENT,
    -> PRIMARY KEY (c1)
    ->  ) ENGINE = InnoDB;

mysql> INSERT INTO t1 VALUES(0), (0), (3);

mysql> SELECT c1 FROM t1;
+----+
| c1 |
+----+
|  1 |
|  2 |
|  3 |
+----+

mysql> UPDATE t1 SET c1 = 4 WHERE c1 = 1;

mysql> SELECT c1 FROM t1;
+----+
| c1 |
+----+
|  2 |
|  3 |
|  4 |
+----+

mysql> INSERT INTO t1 VALUES(0);

mysql> SELECT c1 FROM t1;
+----+
| c1 |
+----+
|  2 |
|  3 |
|  4 |
|  5 |
+----+
```

#### InnoDB自增计数器初始化

这一小节介绍InnoDB如何初始化AUTO_INCREMENT计数器

如果你为一张InnoDB表指定了自增列，在内存中的表对象包含一个自增计数器，用来给自增列分配自增值。

在MySQL5.7以及之前的版本，自增计数器只存储在内存中而不在磁盘上。重启服务器之后为了初始化一个自增计数器，在首次向带有自增列的表中插入数据时，InnoDB会执行一个等价于以下语句的语句。

```sql
SELECT MAX(ai_col) FROM table_name FOR UPDATE;
```

在 MySQL8.0中，改变了这种行为，当前最大的自增计数器值每次改变时会被写到redo log中且在每一个检查点会被保存到引擎独有的系统表中。这种改变使得当前最大的自增计数器值能够在服务器重启之后仍旧有效。

当一个服务器正常关闭再重启，InnoDB使用存储在数据词典系统表中的最大自增值来初始化自增计数器。


当一个服务器由于宕机后重启，InnoDB使用存储在数据词典系统表中的最大自增值来初始化自增计数器，并且扫描redo log中上次检查点写入的自增值。如果redo log里记录的值大于内存中的自增计数值，将会使用redo log中的值。然而，服务器宕机的情况下，无法保证之前已分配的自增值不被重用。每次由`INSERT`或`UPDATE`操作导致的自增值改变时，新的自增值被写入到redo log，但如果在redo log中的内容被刷到磁盘之前就宕机了，当服务器重启且自增计数器初始化后，之前分配的值可能是会被重用的。

在MySQL8.0及之后版本中InnoDB唯一使用等同于SELECT MAX(ai_col) FROM table_name FOR UPDATE语句来初始化自增计数器的场景是导入表空间时不带.cfg元数据文件。否则，当前的最大自增计数器值来源于.cfg元数据文件。

在MySQL5.7以及之前版本，服务器重启会忽略AUTO_INCREMENT = N 表选项，这个表选项分别用于在创建表时初始化计数器值和更新表时修改当前的计数器值。在MySQL8.0中，服务器重启不会忽略AUTO_INCREMENT = N表选项。如果你初始化自增计数器为一个指定值，或者你更改自增计数器值到一个更大的值，重启服务器后新的值会持续生效。

> **NOTE**
> ALTER TABLE ... AUTO_INCREMENT = N can only change the auto-increment counter value to a value larger than the current maximum.

在MySQL5.7及之前版本，服务器重启之后立即执行ROLLBACK操作可能会导致这个回滚的事务分配的自增值被重用，实际上回滚的是当前最大的自增值。在MySQL8.0中，当前最大的自增值是持续的，不会重用之前分配的值。

如果在自增计数器初始化之前使用`SHOW TABLE STATUS`语句来检测一张表，InnoDB会打开这张表和使用存储在数据词典系统表中的自增值来初始化计数器。这个值会被存储在内存中以便之后的inserts和updates使用。计数器的初始化使用一个普通的排他锁去读取表数据直到事务结束。新建表时用户指定大于0的自增值InnoDB也会遵循同样的操作来初始化自增计数器。

自增计数器初始化之后，插入一行数据时如果你没有显式指定一个自增值，InnoDB会隐式增长计数器并分配一个新值给列。如果你插入一行数据并且显式指定了一个自增值，且这个值大于当前最大的计数器值，则计数器会设置为这个指定值。

只要MySQL服务器在运行，InnoDB都会使用内存里的计数器。当服务器停止并重启后，InnoDB会重新初始化自增计数器，正如之前所说的。

`auto_increment_offset`配置选项决定了自增列值的起点，默认设置时1。

`auto_increment_increment`配置选项控制连续列值之间的步长，默认设置是1。
