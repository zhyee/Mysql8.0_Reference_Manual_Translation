## 11.1 数字类型

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/numeric-types.html](https://dev.mysql.com/doc/refman/8.0/en/numeric-types.html)

- [11.1.1 数字类型语法](data-types/numeric-types.md)
- [11.1.2 整形（精确值类型）- INTEGER,INT,SMALLINT,TINYINT,MEDIUMINT,BIGINT](data-types/)
- [11.1.3 定点数（精确值类型）- DECIMAL,NUMERIC](data-types/)
- [11.1.4 浮点数（非精确值类型）- FLOAT,DOUBLE](data-types/)
- [11.1.5 比特值类型 - BIT](data-types/)
- [11.1.6 数字类型的属性](data-types/)
- [11.1.7 超出范围和溢出处理](data-types/)

MySQL支持所有的标准SQL数字类型，这些类型包括精确的数字类型（INTEGER、SMALLINT、DECIMAL和NUMERIC），也支持非精确数字类型（FLOAT、REAL和DOUBLE PRECISION）。INT是INTEGER的同义词，DEC和FIXED是DECIMAL的同义词，MySQL把DOUBLE作为DOUBLE PRECISION的同义词（非标准扩展），同时在没有开启REAL_AS_FLOAT模式时，MySQL也把REAL当做DOUBLE PRECISION的同义词（非标准扩展）。

BIT 数据类型存储比特值，MyISAM、MEMORY、INNODB和NDB表支持BIT 类型。

关于MySQL如何处理给字段赋值时超出范围以及表达式计算结果溢出的情况，请看 `第11.1.7节 超出范围和溢出处理`。

关于数据类型对存储的要求，请查看 `第11.7节 数据类型存储要求`。

关于数值操作函数的相关介绍，请查看 `第12.6节 数值函数和操作符`。保存数值型运算结果的数据类型与操作数的类型以及运算类型有关。更多信息请查阅 `第12.6.1节 算数运算符`。