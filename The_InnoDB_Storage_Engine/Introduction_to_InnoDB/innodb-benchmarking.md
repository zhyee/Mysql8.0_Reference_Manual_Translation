## 15.1.4 对InnoDB进行功能测试和基准测试

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-benchmarking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-benchmarking.html)

如果InnoDB不是你的默认存储引擎，你可以在命令行中用参数 `--default-storage-engine=InnoDB`，或者在MySQL服务器的配置文件中的 [mysqld]段配置值`default-storage-engine=innodb`来重启你的服务器，然后验证你的服务器或应用程序是否运行正常。

由于改变默认的存储引擎只会影响之后新建的表，运行你的应用程序的所有安装和设置步骤来确认安装的正确性。然后测试所有的程序功能以确保所有的数据加载、编辑和查询功能正常。如果有表依赖于其他存储引擎所独有的特性，你可能会收到一个报错；这时你可以在`CREATE TABLE`建表语句加上`ENGINE=other_engine_name`选项来避免这个问题。

如果你不能毫无后顾之忧的做出切换存储引擎的决定，并且你想预览一下使用`ALTER TABLE table_name ENGINE=InnoDB`创建的每一张InnoDB表是否能正常工作，或者你想在不影响原来的表的情况下测试一下查询或其他操作，使用下面的命令来复制原来的表：

> CREATE TABLE InnoDB_Table (...) ENGINE=InnoDB AS SELECT * FROM other_engine_table;

安装最新版的MySQL服务器并运行基准测试来评估在正式环境中一个完整项目的性能。

测试应用程序需要覆盖从安装到高负载整个的产品功能。当数据库在高负载的情况下杀掉数据库进程来模拟断电的情况，然后验证重启数据库后数据是否成功的恢复。

测试所有的主从复制配置项，尤其是在主从数据库之间使用了不同的版本和配置的情况下。
