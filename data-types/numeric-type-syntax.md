## 11.1.1 数值类型语法

> 原文地址：[https://dev.mysql.com/doc/refman/8.0/en/numeric-type-syntax.html](https://dev.mysql.com/doc/refman/8.0/en/numeric-type-syntax.html)

对于整型数据类型来说，*M* 表示最大显示宽度，支持的最大显示宽度是255，正如 `第11.1.6节 数值型的属性` 所介绍的，显示宽度和一个类型所能存储的值的范围没有关联。

对于浮点数和定点数来说，*M* 是一个数字的总位数。

在MySQL8.0.17版本中，整型类型的显示宽度属性被标记为过时的，并会在将来的版本中移除。

如果你为数值型字段指定了ZEROFILL属性，MySQL会自动给这个字段添加UNSIGNED属性。

在MySQL8.0.17版本中，数值型字段的ZEROFILL属性被标记为过时的，并会在将来的版本中移除。考虑使用一个替代方案来实现相同的效果，例如，应用中可以使用LPAD()函数来添加前置0来达到想要的宽度，或者用CHAR类型来存储格式化后的数值。

数值类型允许设置UNSIGNED或SIGNED属性，默认是SIGNED，因此设置SIGNED属性和没设置的效果是一样的。

在MySQL8.0.17中，FLOAT、DOUBLE和DECIMAL（以及它们的同义词）类型的UNSIGNED的属性被标记为过时属性，对于它们的支持在将来的版本中将会被移除，考虑使用一个简单的CHECK约束来代替这钟用法。

SERIAL 是 BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE 的别名

定义在一个整形字段上的SERIAL DEFAULT VALUE是 NOT NULL AUTO_INCREMENT UNIQUE 的别名。

> **WARNING** 
>
> 当你进行整型之间的减法运算时，如果其中有一个数是UNSIGNED，除非开启了NO_UNSIGNED_SUBTRACTION模式，否则结果也是一个UNSIGNED整数，请查看 `第12.11节 类型转换函数和运算`。

- BIT[( *M* )]

  比特类型，M 表示每个值得比特位数，从1到64，如果省略M 则默认值为1。

- TINYINT[( *M* )]  [UNSIGNED]  [ZEROFILL]

  较小的整数，有符号数的范围是 -128 ~ 127，无符号数的范围是 0 ~ 255 。

- BOOL, BOOLEAN

  这两种类型是TINYINT(1) 的别名，0认为是false，非零值认为是true：

```

mysql> SELECT IF(0, 'true', 'false');
+------------------------+
| IF(0, 'true', 'false') |
+------------------------+
| false                  |
+------------------------+

mysql> SELECT IF(1, 'true', 'false');
+------------------------+
| IF(1, 'true', 'false') |
+------------------------+
| true                   |
+------------------------+

mysql> SELECT IF(2, 'true', 'false');
+------------------------+
| IF(2, 'true', 'false') |
+------------------------+
| true                   |
+------------------------+

```

然而，TRUE和FALSE仅仅分别是1和0的别名，如下所示：

```

mysql> SELECT IF(0 = FALSE, 'true', 'false');
+--------------------------------+
| IF(0 = FALSE, 'true', 'false') |
+--------------------------------+
| true                           |
+--------------------------------+

mysql> SELECT IF(1 = TRUE, 'true', 'false');
+-------------------------------+
| IF(1 = TRUE, 'true', 'false') |
+-------------------------------+
| true                          |
+-------------------------------+

mysql> SELECT IF(2 = TRUE, 'true', 'false');
+-------------------------------+
| IF(2 = TRUE, 'true', 'false') |
+-------------------------------+
| false                         |
+-------------------------------+

mysql> SELECT IF(2 = FALSE, 'true', 'false');
+--------------------------------+
| IF(2 = FALSE, 'true', 'false') |
+--------------------------------+
| false                          |
+--------------------------------+

```

  最后两个查询的结果之所以都是false是因为2既不等于1也不等于0。

- SMALLINT[( *M* )] [UNSIGNED] [ZEROFILL]

  一个较小的整数，有符号数范围 -32768 ~ 32767, 无符号数的范围是 0 ~ 65535。

- MEDIUMINT[( *M* )] [UNSIGNED] [ZEROFILL]

  中等大小的整数，有符号数的范围是 -8388608 ~ 8388607， 无符号数的范围是 0 ~ 16777215。

- INT[( *M* )] [UNSIGNED] [ZEROFILL]

  普通大小的整数，有符号数的范围数 -2147483648 ~ 2147483647，无符号数的范围是 0 ~ 4294967295。

- INTEGER[( *M* )] [UNSIGNED] [ZEROFILL]

  INT的别名

- BIGINT[( *M* )] [UNSIGNED] [ZEROFILL]

  一个非常大的整数，有符号数的范围是 -9223372036854775808 ~ 9223372036854775807，无符号数的范围是 0 ~ 18446744073709551615。
  
  SERIAL 是 BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE的别名。
  
  关于BIGINT有一些注意点：
  
  - 所有的算术运算使用有符号BIGINT或DOUBLE值进行，因此除非是用于位操作函数，否则不应该使用超过9223372036854775807（63比特位）的无符号整数，如果你使用了超过了9223372036854775807的无符号整数，则运算结果的低位可能由于在把BIGINT转换为DOUBLE时取整而出现误差。
  
    MySQL可以处理以下几种情形的BIGINT：
    
      - 在BIGINT字段上利用整数来存储一个较大的无符号数。
      
      - 在MIN( *col_name* ) 或 MAX( *col_name* ), 当 *col_name* 是一个BIGINT字段时。
      
      - 当操运算数都是整数时使用运算符(+、-、*等)。
      
   - 你总是可以在一个BIGINT字段上使用字符串来存储一个精确的整数，在这种使用情况下MySQL会执行string-to-number的转换从而避免双进度数的中间转换形式。
   
   - 当运算数都是整数时，-、+和*运算符使用BIGINT算术运算。这就意味着两个大数之间的乘积（或返回整数的函数）大于9223372036854775807时，你可能无法得到预期值。
   
- DECIMAL[( *M*[,*D*])] [UNSIGNED] [ZEROFILL]



