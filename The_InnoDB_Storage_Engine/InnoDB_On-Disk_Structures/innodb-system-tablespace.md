## 15.6.3.1 系统表空间
> 原文地址 [https://dev.mysql.com/doc/refman/8.0/en/innodb-system-tablespace.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-system-tablespace.html)

系统表空间是用于保存更新缓冲（change buffer）的存储区域。如果一张表是在系统空间创建的，而不是单文件表或在通用表空间上创建，那么系统表空间可能会同时包含数据和索引。在旧的MySQL版本中，系统表空间保存了InnoDB的数据词典。在MySQL8.0中，InnoDB在MySQL的数据词典中保存了元数据。查看 `第14章 MySQL数据词典`来了解更多信息。在之前的MySQL发布版中，系统表空间还包括双写缓冲的存储空间。在MySQL8.0.20中，这部分存储区域被单独放到了双写缓冲的文件中。请查看`第15.6.4节 双写缓冲`获得更多信息。

系统表空间可以有一个或多个数据文件。默认的，一个独立的系统表空间会在data目录中创建一个名为`ibdata1`的数据文件。系统表空间数据文件的大小和数量由`innodb_data_file_path`启动选项来决定。对于配置的相关介绍，请查看`系统表空间数据文件配置`。

关于系统表空间的其他信息分别在本节下面的两个主题中介绍：

- 重新设置系统表空间的大小

- 使用原始的磁盘分区作为系统表空间

### 重新设置系统表空间的大小

这一节介绍如何增加或减少系统表空间的大小。

**增加系统表空间的大小**

增加系统表空间大小最简单的方式是配置它的大小为自增长的。可以通过指定`innodb_data_file_path`设置项最后一个数据文件的`autoextend`属性并重启MySQL服务器来实现。例如：

```
innodb_data_file_path=ibdata1:10M:autoextend
```

当指定了`autoextend`属性后，数据文件在需要的时候会自动按每次8M的大小增长。`innodb_autoextend_increment`属性控制每次增长的大小。

你也可以通过新增另一个数据文件来增加系统表空间的大小。具体操作步骤如下：

1. 停止MySQL服务器。

2. 如果`innodb_data_file_path`配置项的最后一个数据文件定义了`autoextend`属性，先把它移除，并且修改size属性为当前数据文件的大小。为了确定合适的文件大小，查看你的文件系统来获得当前的文件大小，在这个当前文件大小值附近向下取整换算成最小的MB数，1MB=1024*1024。

3. 添加一个新的数据文件到`innodb_data_file_path`配置项，可以选择添加或不添加`autoextend`属性。`autoextend`属性只能在`innodb_data_file_path`值上的最后一个数据文件上设置。

4. 启动MySQL服务器。

举例来说，现有一个表空间并设置了一个自增的数据文件：

```
innodb_data_home_dir = 
innodb_data_file_path = /ibdata/ibdata1:10M:autoextend
```

假如这个数据文件现在已经增长到988M的大小，则新增一个数据文件并给它设置自增长属性，可以按如下方式设置：

```
innodb_data_home_dir = 
innodb_data_file_path = /ibdata/ibdata1:988M;/disk2/ibdata2:50M:autoextend
```

新增一个数据文件时，如果这个文件不存在，InnoDB会在启动服务器的时候创建和初始化这个新的数据文件。

```
**Note**
你无法通过修改一个已有的系统表空间数据文件的size属性来增加它的大小。例如，把`innodb_data_file_path`设置从`ibdata1:10M:autoextend`修改为 `ibdata1:12M:autoextend`，则启动服务器的时候会产生如下的错误：
    [ERROR] [MY-012263] [InnoDB] The Auto-extending innodb_system 
    data file './ibdata1' is of a different size 640 pages (rounded down to MB) than 
    specified in the .cnf file: initial 768 pages, max 0 (relevant if non-zero) pages!

这个错误表明已存在的数据文件的大小（InnoDB数据页表示）与配置文件中指定的大小是不同的。如果你遇到了这个错误，你需要还原`innodb_data_file_path`的设置，然后参考上面的修改系统表大小的步骤。

InnoDB 页大小由`innodb_page_size`变量来决定，默认的大小是16384字节。
```

**减少系统表空间的大小**

当前不支持减少一个已存在的系统表空间大小。实现一个较小的系统表空间的唯一方法是创建一个新的MySQL实例，并分配想要的空间大小，然后备份旧的MySQL实例的数据再导入新的MySQl实例中。

关于创建备份的有关知识，请查看`第15.18.1节 InnoDB备份`。

为了避免过大的系统表空间，可以考虑使用单文件表（file-per-table）空间来存储你的数据。当你创建一张InnoDBb表时单文件表空间是默认的类型。和系统表空间不同，清空和删除一个单文件表空间类型得表后，占用的磁盘空间会被归还给操作系统，具体请查看`第15.6.3.2节 单文件表空间`。

### 使用原始磁盘分区作为系统表空间

你可以使用原始的磁盘分区来作为系统表空间的数据文件。这项技术可以在不加重文件系统负载的情况下在Windows上或某些Linux和Unix系统上使用不带缓冲的I/O。比较使用原始分区和不使用的性能差别，来判定这项技术是否的确能够提升你的系统性能。

当你使用一块原始的磁盘分区时，确保你运行MySQL服务器的用户有读写该磁盘分区的权限。例如，如果你运行MySQL服务器的用户是`mysql`,则磁盘分区必须能够被`mysql`读写。如果你使用`--memlock`选项，则MySQL服务器必须以root用户运行，那么该磁盘分区也需要能够被root用户读写。

下面的流程涉及到了选项文件的修改，想获得有关的更多信息，请查看`第4.2.2.2节 使用选项文件`。

**在Linux或Unix系统上分配一块原始磁盘分区**

1. 当你创建一个新的数据文件时，在`innodb_data_file_path`值的数据文件大小后面立马跟上关键词`newraw`。分区的容量必须大于你指定的数据文件大小。注意InnoDB中的1MB是1024*1024字节，而磁盘标注规格上的1MB通常是1,000,000字节。

```
[mysqld]
innodb_data_home_dir=
innodb_data_file_path=/dev/hdd1:3Gnewraw;/dev/hdd2:2Gnewraw
```

2. 重启服务器。InnoDB注意到`newraw`关键词时会初始化这个新分区。然而，此时还不能创建和修改任何的InnoDB表，否则你下次重启服务器时，InnoDB会重新初始化分区，你的所有修改也将会丢失。（作为一个安全手段，当一个分区指定了`newraw`关键词，InnoDB引擎会阻止用户对数据的修改。）

3. 当InnoDB初始化完新分区，停止MySQL服务器，把配置文件中的`newraw`改为`raw`:

```
[mysqld]
innodb_data_home_dir=
innodb_data_file_path=/dev/hdd1:3Graw;/dev/hdd2:2Graw
```

4. 重启MySQL服务器，InnoDB现在允许对数据进行修改了。

**在Windows系统上分配一块原始磁盘分区**

在Windows系统上，除了`innodb_data_file_path`的配置有略微的不同之外，其他的步骤都与Linux和Unix系统的类似。

1. 当你创建一个新的数据文件时，在`innodb_data_file_path`值的数据文件大小后面立马跟上关键词`newraw`：

```
[mysqld]
innodb_data_home_dir=
innodb_data_file_path=//./D::10Gnewraw
```
//./ 指代Windows中的\\.\，表示要访问物理驱动器。在上面的例子里，D:是分区的盘符。

2. 重启服务器。InnoDB观察到newraw关键词后会去初始化新的分区。

3. 初始化完毕之后，停止服务器，修改配置文件把`newraw`修改为`raw`:

```
[mysqld]
innodb_data_home_dir=
innodb_data_file_path=//./D::10Graw
```

4. 重启服务器，现在InnoDB允许你对数据进行修改了。
