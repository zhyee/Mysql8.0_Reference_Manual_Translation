## 15.5.2 更新缓冲

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html)

更新缓冲是一种用于缓存对不在buffer pool里的二级索引页的修改。这些更新可能来自`INSERT`,`UPDATE`或者`DELETE`等操作(DML)，稍后这些更新会被其他的读操作合并到buffer pool中。

##### 图15.3 更新缓存

![](../../resources/innodb-change-buffer.png)

与聚簇索引不同，二级索引通常是非唯一的，并且插入二级索引的数据通常相对是无序的。相同的，删除和更新可能会影响索引树中不相邻的索引页。在稍后当有别的操作把影响的页读到缓存池时会合并更新到缓存池中，避免从磁盘读二级索引页到缓存池中造成的大量的随机IO。

在系统负载较低的时候或者在安全关闭服务器期间，清理操作会周期性的运行并把更改的页写到磁盘中。清理操作能够高效的向磁盘块写大量的索引值，相比每个值更改之后立即写到磁盘上。

当有大量的行和二级索引更新时，合并更新缓冲可能需要花费好几个小时。在这期间，磁盘I/O会上升，这会导致依赖磁盘的查询会显著变慢。合并更改缓冲可能在事务提交甚至服务器关闭并重启之后继续运行。（相关更多信息，请参见 `第15.20.2节 强制InnoDB恢复`）。

在内存中，更新缓冲是buffer pool的一部分。在磁盘上，更新缓冲是系统表空间的一部分，这块空间用于服务器关闭时缓存更改的索引。

更新缓冲中缓存的数据类型由配置选项`innodb_change_buffering`决定。更多信息请查看`配置更新缓冲`。同时你也可以配置缓冲池的最大值。更多信息查看`配置更新缓冲最大值`。

更新缓冲不支持含有降序索引列的辅助索引和降序索引列的主键索引。

关于更新缓冲的FAQ，请查看`第A.15节 MySQL8.0 FAQ：InnoDB缓冲`。

#### 配置更新缓冲
当对一个表执行INSERT、UPDATE或DELETE操作时，存在索引的列值（尤其是辅助索引）通常是无序的，而这需要大量的I/O来让辅助索引重新保持有序。更新缓冲把不在buffer pool中的辅助索引修改缓存起来，这样可以避免立即从磁盘读取数据所造成的高代价的I/O。当页被加载进buffer pool时更新缓存中的修改会被合并到buffer pool中来，并在稍后刷新到磁盘上。InnoDB的主线程在服务器相对空闲或者在慢关闭期间合并这个缓存起来的变更。

因为更新缓冲能够减少磁盘读写，所以这个特性特别适合处理I/O密集型的工作，例如有大量类似批量插入的DML操作的应用。

然而，更新缓存是buffer pool的一部分，这会降低buffer pool缓存数据页的能力。如果你的数据几乎都存在buffer pool里，或者你的表拥有较少的辅助索引，关闭更新索引可能是非常有用的。如果你的数据完全存在buffer pool里，更新缓存也不会增加额外的负担，因为它仅仅用于不在buffer pool里的数据页。

你可以使用`Innodb_chnage_buffering`配置参数来控制InnoDB更新缓存对哪些操作进行缓存。你可以对插入、删除（索引记录起初被标记为删除的时候）和清理操作（索引记录被物理删除的时候）。更新操作是插入和删除的结合。默认的`innodb_change_buffering`值是all。

允许的`innodb_change_buffering`值如下：

- *all*

all是默认值：插入、标记删除、和清理操作都会被缓存。

- *none*

不会缓存任何操作。

- *inserts*

缓存插入操作。

- *deletes*

缓存标记删除的操作。

- *changes*

缓存插入和标记删除的操作。

- *purges*

缓存后台的物理删除操作。

你可以在MySQL配置文件（my.cnf或my.ini）中设置或者使用`SET GLOBAL`语句来动态修改`innodb_change_buffering`值，这需要足够的权限来全局修改系统变量。具体请查阅`第5.1.9.1节 系统变量权限`。修改配置影响新操作的缓存行为，已缓存的实体合并行为不会受影响。

#### 配置更新缓存的最大容量

`innodb_change_buffer_max_size`参数允许配置更新缓存占buffer pool空间的最大占比。`innodb_change_buffer_max_size`默认被设置为25，允许的最大值是50。

在一个频繁执行插入、更新和删除操作的服务器上考虑增大`innodb_change_buffer_max_size`的值，由于合并更新缓存的节奏跟不上新的更新缓存产生的速度，导致更新缓存达到最大容量上限。

如果服务器有很多只供查询的不变数据，或者更新缓存消耗了太多buffer pool的空间导致buffer pool里的页面过快失效，可以考虑降低`innodb_change_buffer_max_size`的值。

在典型的负载情况下尝试不同的设置值以达到一个最优配置。`innodb_change_buffer_max_size`设置是动态生效的，修改它的值不需要重启服务器。

#### 监控更新缓存

下面的选项可以用来监控更新缓存的状态：

- InnoDB标准监控输出包括更新缓存的状态信息。可以发起 `SHOW ENGINE INNODB STATUS`查询语句来查看监控信息。

> mysql> SHOW ENGINE INNODB STATUS\G

更新缓存状态信息处于`INSERT BUFFER AND ADAPTIVE HASH INDEX`段，展示的内容类似如下：

```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 4425293, used cells 32, node heap has 1 buffer(s)
13577.57 hash searches/s, 202.47 non-hash searches/s
```

更多详细信息请查看 `第15.16.3节 InnoDB标准监控和锁监控输出`。

- INFORMATION_SCHEMA.INNODB_METRICS 表提供了InnoDB标准监控输出中的大部分数据指标以及额外的一些信息。可以通过下面的查询语句来查看更新缓存的指标以及指标说明：

> mysql> SELECT NAME, COMMENT FROM INFORMATION_SCHEMA.INNODB_METRICS WHERE NAME LIKE '%ibuf%'\G

对于 INNODB_METRICS 表的使用，请查看`第15.14.6节 InnoDB INFORMATION_SCHEMA Metrics Table`。

- `INFORMATION_SCHEMA.INNODB_BUFFER_PAGE` 表提供了buffer pool中每个内存页的元数据信息，包括更新缓存索引和更新缓存位图页。更新缓存用`PAGE_TYPE.IBUF_IDNEX`来标识更新缓存索引页类型，用`IBUF_BITMAP`来标识更新缓存位图页类型。

> #####Warning
> 查询`INNODB_BUFFER_PAGE`表可能引起严重的性能降低。为了避免影响性能，在测试实例上复现你想排查的问题。

例如，你可以查询`INNODB_BUFFER_PAGE`表来确定`IBUF_INDEX`和`IBUF_BITMAP`类型的内存页数量以及占缓存池总页数的百分比。

```
mysql> SELECT (SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE PAGE_TYPE LIKE 'IBUF%') AS change_buffer_pages,
       (SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE) AS total_pages,
       (SELECT ((change_buffer_pages/total_pages)*100))
       AS change_buffer_page_percentage;
+---------------------+-------------+-------------------------------+
| change_buffer_pages | total_pages | change_buffer_page_percentage |
+---------------------+-------------+-------------------------------+
|                  25 |        8192 |                        0.3052 |
+---------------------+-------------+-------------------------------+
```

如果想了解`INNODB_BUFFER_PAGE`表提供的其他数据，请查看`第25.39.1节 INFORMATION_SCHEMA INNODB_BUFFER_PAGE 表`。了解相关的使用信息，请查看`第15.14.5节 INNODB INFORMATION_SCHEMA Buffer Pool表`。

- Performance Schema 提供了为高级的性能监控所需的更新缓存排他锁等待表盘视图信息。可以使用如下的查询或查看更新缓存表盘视图信息。

```
mysql> SELECT * FROM performance_schema.setup_instruments
       WHERE NAME LIKE '%wait/synch/mutex/innodb/ibuf%';
+-------------------------------------------------------+---------+-------+
| NAME                                                  | ENABLED | TIMED |
+-------------------------------------------------------+---------+-------+
| wait/synch/mutex/innodb/ibuf_bitmap_mutex             | YES     | YES   |
| wait/synch/mutex/innodb/ibuf_mutex                    | YES     | YES   |
| wait/synch/mutex/innodb/ibuf_pessimistic_insert_mutex | YES     | YES   |
+-------------------------------------------------------+---------+-------+
```

为了解更多InnoDB锁等待的信息，请查看`第15.15.2节 用Performance Schema表监控InnoDB排他锁等待`。
