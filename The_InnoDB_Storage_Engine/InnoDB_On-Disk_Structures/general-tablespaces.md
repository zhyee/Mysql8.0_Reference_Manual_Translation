## 15.6.3.3 通用表空间
> 原文地址 [https://dev.mysql.com/doc/refman/8.0/en/general-tablespaces.html](https://dev.mysql.com/doc/refman/8.0/en/general-tablespaces.html)

通用表空间是使用`CREATE TABLESPACE`语句创建的一个共享的表空间。通用表空间的用法和特性将会分下面的几个主题进行介绍：

- 通用表空间的用法

- 创建一个通用表空间

- 在通用表空间中创建表

- 通用表空间支持的行格式

- 使用`ALTER TABLE`语句在表空间之间移动表

- 重命名一个通用表空间

- 移除一个通用表空间

- 通用表空间的局限性

### 通用表空间的用处

通用表空间提供了如下的特性和能力：

- 和系统表空间类似，通用表空间是可以存储多张表数据的共享空间。

- 通用表空间比单文件表空间有一个内存上的优势。服务器会在表空间的整个生命周期内把它的元数据保存在内存中。在较少的通用表空间里存储的多张表，相比用单文件表空间存储同样多的表，会花费更少的内存来存储表空间元数据。

- 通用表空间数据文件可以保存在MySQL data目录的相对目录或独立的其他目录下，从而拥有和单文件表空间类似的多数据文件和存储设备的管理能力。如何单文件表空间一样，能够把数据文件放到MySQL data目录之外地方的能力允许你分别管理关键表的性能，比如为指定的表设置RAID或DRBD，或者把表绑定到特定的磁盘等等。

- 通用表空间支持所有的行格式及其特性。

- `CREATE TABLE`语句可以使用`TABLESPACE`选项来在通用表空间、单文件表空间或者系统表空间中创建表。

- `ALTER TABLE`语句可以使用`TABLESPACE`选项来在通用表空间，单文件表空间和系统表空间之间移动表数据。在之前的版本中，是无法把一张表从单文件表空间移动到系统表空间的，利用通用表空间的特性你现在可以这样做了。

### 创建通用表空间

可以使用`CREATE TABLESPACE`语句来创建通用表空间。

```
CREATE TABLESPACE tablespace_name
    [ADD DATAFILE 'file_name']
    [FILE_BLOCK_SIZE = value]
        [ENGINE [=] engine_name]
```

通用表空间可以位于MySQL的data目录中，也可以位于data目录之外。为了避免和单文件表空间冲突，在data目录的子目录中创建通用表空间是不允许的。当在data目录之外创建通用表空间时，该目录必须存在并且提前被innodb识别到。为了能让InnoDB引擎识别一个目录，可以把该目录加到`innodb_directories`参数里。`innodb_directories`是一个只读的启动项，修改它的值需要重启服务器。

例如：
在MySQL的data目录下创建通用表空间：

> mysql> CREATE TABLESPACE `ts1` ADD DATAFILE 'ts1.ibd' Engine=InnoDB;

或者

> CREATE TABLESPACE `ts1` Engine=InnoDB;

`ADD DATAFILE`语法是MySQL8.0.14版本的可选项，而在之前的版本中是必须项。如果在创建表空间时没有指定`ADD DATAFILE`语法，会隐式的创建一个唯一文件名的数据文件。此唯一文件名是一个128位UUID，由横线分割的5组16进制数（aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee）。通用表空间数据文件的后缀是.ibd。在主从环境中，主库的文件名和从库的文件名是不同的。


在data目录之外创建通用表空间语法如下：

> mysql> CREATE TABLESPACE `ts1` ADD DATAFILE '/my/tablespace/directory/ts1.ibd' Engine=InnoDB;

你可以用data目录的相对路径来指定所在的目录，只要这个目录不是在data目录之下即可。在下例中，`my_tablespace`目录是和data目录同一级别：

> mysql> CREATE TABLESPACE `ts1` ADD DATAFILE '../my_tablespace/ts1.ibd' Engine=InnoDB;

```
**Note**
CREATE TABLESPACE语句中必须指定ENGINE = InnoDB，或者InnoDB设置为MySQL服务器的默认的存储引擎（default_storage_engine=InnoDB）。

```

### 在通用表空间中创建表

创建完InnoDB的通用表空间后，你可以使用`CREATE TABLE tbl_name ... TABLESPACE [=] tablespace_name` 或 `ALTER TABLE tbl_name TABLESPACE [=] tablespace_name`来向该表空间中增加表，正如下面示例中所示：

CREATE TABLE:

> mysql> CREATE TABLE t1(c1 INT PRIMARY KEY) TABLESPACE ts1;

ALTER TABLE:

> mysql> ALTER TABLE t2 TABLESPACE ts1;

```
**NOTE**
向共享表空间中添加表分区在MySQL5.7.24中被标记为过时的功能，并且在MySQL8.0.13中正式被移除了。这里说的共享表空间包括InnoDB系统表空间和通用表空间。
```

具体的语法信息，请查阅`CREATE TABLE`和`ALTER TABLE`语句。

### 通用表空间支持的行格式

通用表空间支持所有的数据表行格式（REDUNDANT,COMPACT,DYNAMIC,COMPRESSED），但由于不同的物理页大小，压缩的和非压缩的表无法共存于同一块通用表空间里。

对于一个包含压缩表（ROW_FORMAT=COMPRESSED）的通用表空间来说，必须指定`FILE_BLOCK_SIZE`配置值，并且`FILE_BLOCK_SIZE`必须是一个和`innodb_page_size`值有关的有效的页大小，并且，压缩表的物理页（`KEY_BLOCK_SIZE`）必须等于 `FILE_BLOCK_SIZE`/1024，例如，如果`innodb_page_size`=16KB且`FILE_BLOCK_SIZE`=8K，则`KEY_BLOCK_SIZE`的值必须为8。

下表展示了`innodb_page_size`，`FILE_BLOCK_SIZE`和`KEY_BLOCK_SIZE`三者之间允许的值组合。`FILE_BLOCK_SIZE`的值有可能是以字节为单位。对于给定的`FILE_BLOCK_SIZE`值，只需要将`FILE_BLOCK_SIZE`的值除以1024便可以确定`KEY_BLOCK_SIZE`的大小。InnoDB 32K和64K的页大小是不支持表压缩的。更多关于`KEY_BLOCK_SIZE`的信息，请查看`CREATE TABLE`语法和`第15.9.1.2节 创建压缩表`。

**表15.3 支持压缩表的页大小、FILE_BLOCK_SIZE和KEY_BLOCK_SIZE三者之间的组合**

| InnoDB页大小（innodb_page_size） | 允许的FILE_BLOCK_SIZE值 | 允许的KEY_BLOCK_SIZE值 |
| ------------------------------  | ---------------------- | ---------------------- |
|64KB                             |64K(65536)              |不支持压缩表          |
|32KB                             |32K(32768)              |不支持压缩表         |
|16KB |16K(16384) |N/A：如果`innodb_page_size`值等于`FILE_BLOCK_SIZE`，则该表空间不支持压缩表  |
|16KB                             |8K(8192)                |8                       |
|16KB                             |4K(4096)                |4                       |
|16KB                             |2K(2048)                |2                        |
|16KB                             |1K(1024)                |1                        |
|8KB   |8K(8192)   |N/A：如果`innodb_page_size`值等于`FILE_BLOCK_SIZE`，则该表空间不支持压缩表 |
|8KB                             |4K(4096)                 |4                        |
|8KB                             |2K(2048)                 |2                        |
|8KB                             |1K(1024)                 |1                        |
|4KB  | 4K(4096) |N/A：如果`innodb_page_size`值等于`FILE_BLOCK_SIZE`，则该表空间不支持压缩表 |
|4KB                             |2K(2048)                 |2                        |
|4KB                             |1K(2048)                 |1                       |

下面这个例子演示创建一个通用表空间，并向其添加压缩表。这个例子假设`innodb_page_size`的大小是16KB，`FILE_BLOCK_SIZE`是8192，则压缩表的`KEY_BLOCK_SIZE`的值需要为8。

```
mysql> CREATE TABLESPACE `ts2` ADD DATAFILE 'ts2.ibd' FILE_BLOCK_SIZE = 8192 Engine=InnoDB;
mysql> CREATE TABLE t4(c1 INT PRIMARY KEY) TABLESPACE ts2 ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

如果你在创建通用表空间时不指定`FILE_BLOCK_SIZE`的大小，`FILE_BLOCK_SIZE`默认等于`innodb_page_size`的大小，而当`FILE_BLOCK_SIZE`的值等于`innodb_page_size`时，则该表空间只能存储没有压缩行格式的表（COMPACT,REDUNDANT和DYNAMIC行格式）。