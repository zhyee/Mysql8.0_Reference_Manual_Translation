## 1.2 印刷及语法约定

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/manual-conventions.html](https://dev.mysql.com/doc/refman/8.0/en/manual-conventions.html)

由于本手册仅供参考，因此不提供有关SQL或关系数据库概念的一般说明。它还不会教您如何使用操作系统或命令行解释器。
  
当需要从特定程序中执行命令的时候，命令提示符会显示应该使用哪个应用。例如，shell>提示符表明从登录的shell中执行命令，root-shell>也是类似但是是以root身份执行，mysql>提示符说明命令是从[mysql](https://dev.mysql.com/doc/refman/8.0/en/mysql.html)客户端程序中执行的.

```sql
shell> 在这里输入一个shell命令
root-shell> 以root身份在这里输入一个shell命令
mysql> 在这里输入一个mysql语句 
```

在某些地方，不同的系统可能会相互区分，以表明命令应在两个不同的环境中执行，比如，当主从模式下命令可能会带有master或者slave的前缀：

```sql
master> 以master身份在此处输入mysql命令	
slave> 以slave身份在此处输入mysql命令
```

"shell"是你的命令解释器。在UNIX系统中，通常是指sh，csh，或者bash这些程序。在Windows中，对应的程序是command.com或者cmd.exe，通常在一个终端窗口中运行。
当你输入一条示例中的命令或语句时，不要带上示例中的提示。
输入语句的时候，需要把数据库，数据表，或者字段的名字替换成对应的。要确定哪个需要被替换，本手册使用db_name,tbl_name,col_name来表示该被替换的项。比如，像这样一条SQL语句：

```sql
mysql> SELECT col_name FROM db_name.tbl_name;
```

当你输入类似的语句时就需要把相应的语句中数据库名，数据表名，以及字段名替换掉，比如这样：

```sql
mysql> SELECT author_name FROM biblio_db.author_list
```

SQL语句的关键词是大小写不敏感的，所以大写小写都行。本手册使用大写。
在描述语法时，方括号（[]）表示这个词或语句是可选的。例如，在下面这个语句中，IF EXISTS就是可选的：

```sql
DROP TABLE [IF EXISTS] tbl_name
```

当一个选项具有多种可选项时，用竖线“|”将选项分开。如果选项是非必选的，可选项用方括号（[]）括起来：

```sql
TRIM([[BOTH | LEADING| TRAILING] [remstr] FROM] str)
```

如果选项是必选的，可选项用花括号（{}）括起来：

```sql
{DESCRIBE | DESC} tbl_name [col_name | wild]
```

省略号(…)表示语句中省略的部分，通常是为了将复杂的语法简化。例如，SELECT ... INTO OUTFILE 是将在其他语句之后的具有INTO OUTFILE子句的SELECT语句简化了。  
省略号也可以表示，在其之前的语句是可以重复的。在下边的例子中，reset_option可以具有多个值，每个值之间用逗号分隔：

```sql
RESET reset_option [,reset_option] ...
```

在命令中设置shell变量采用的是Bourne Shell的语法格式，比如，在遵循Bourne Shell语法的语句中，设置cc环境变量并执行configure命令是这样的：
 
```sql
shell> CC=gcc ./configure
```

如果使用chs或tcsh，就需要以一些不同的方式输入命令：

```sql
shell> setenv CC gcc
shell> ./configure
```
