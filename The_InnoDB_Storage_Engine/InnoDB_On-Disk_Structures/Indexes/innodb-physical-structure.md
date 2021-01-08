## 15.6.2.2 InnoDB索引的物理结构

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-physical-structure.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-physical-structure.html)

除了特殊的地理空间索引，InnoDB索引使用B-tree数据结构。地理空间索引使用R-trees，一种用于索引多维数据的特殊数据结构。索引记录存储在B-tree或R-tree数据结构的叶子页面上。默认的索引页大小是16KB。

当InnoDB聚集索引插入新记录时，InnoDB尝试保留1/16的磁盘页空间以备将来插入和更新索引记录。如果索引记录是按某种固定顺序（升序或降序）插入的，形成的索引页大约是15/16。如果索引记录是以随机顺序插入，磁盘页是1/2到15/16。

当创建或重建B-tree索引时InnoDB会执行批量加载。这种创建索引的方式叫做有序索引构建。`innodb_fill_factor`配置选项能够设定有序索引构建时每个B-tree页使用空间的百分比，以便留下空间满足将来索引增长的需要。有序索引构建是不支持空间索引的。了解更多详情，请查看`第15.6.2.3节 有序索引构建`。如果把`innodb_fill_factor`的值设为100则会每个聚集索引页会保留1/16空间以满足将来索引增长的需要。

如果InnoDB索引页的使用比率降到了MERGE_THRESHOLD（没有指定值的话默认是50%）以下，InnoDB会尝试收缩索引树以释放页空间。MERGE_THRESHOLD设置会对B-tree和R-tree同时生效。更多信息请查看`第15.8.11节 为索引页配置合并临界值`。

你可以在初始化MySQL实例之前通过设置`innodb_page_size`选项来为MySQL实例中的所有InnoDB表空间设定页的大小。一旦一个实例的页大小设定好了之后，除非你重新初始化实例，否则不能再更改页的大小。支持的值有64KB,32KB,16KB(默认值)，8KB和4KB。

使用了特定页大小的MySQL实例无法使用不同页大小的数据文件或者日志文件。
