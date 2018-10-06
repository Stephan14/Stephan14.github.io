---
layout:     post
title:      "mysql sql语句分析"
subtitle:   "explain 详解"
date:       2018-10-06 17:44:00
author:     "阿里技术学习博客"
header-img: "img/eiffel-tower-1975412_1920.jpg"
tags:
    - mysql
---

## 语法
```
explain < table_name >
```

## explain输出解释
```
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
```

### id
我的理解是SQL执行的顺利的标识，SQL从大到小的执行。

例如:
```
mysql> explain select * from (select * from ( select * from t3 where id=3952602) a) b;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | <derived3> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  3 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

很显然这条SQL是从里向外的执行，就是从id=3向上执行。

### select_type
就是select类型，可以有以下几种

#### SIMPLE

简单SELECT(不使用UNION或子查询等) 例如:
```
mysql> explain select * from t3 where id=3952602;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
```

#### PRIMARY
我的理解是最外层的select，例如:
```
mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

#### UNION

UNION中的第二个或后面的SELECT语句。例如
```
mysql> explain select * from t3 where id=3952602 union all select * from t3 ;
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type  | table      | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | PRIMARY      | t3         | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
|  2 | UNION        | t3         | ALL   | NULL              | NULL    | NULL    | NULL  | 1000 |       |
|NULL | UNION RESULT | <union1,2> | ALL   | NULL              | NULL    | NULL    | NULL  | NULL |       |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
```

#### DEPENDENT UNION
UNION中的第二个或后面的SELECT语句，取决于外面的查询
```
mysql> explain select * from t3 where id in (select id from t3 where id=3952602 union all select id from t3)  ;
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
| id | select_type        | table      | type   | possible_keys     | key     | key_len | ref   | rows | Extra                    |
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
|  1 | PRIMARY            | t3         | ALL    | NULL              | NULL    | NULL    | NULL  | 1000 | Using where              |
|  2 | DEPENDENT SUBQUERY | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 | Using index              |
|  3 | DEPENDENT UNION    | t3         | eq_ref | PRIMARY,idx_t3_id | PRIMARY | 4       | func  |    1 | Using where; Using index |
|NULL | UNION RESULT       | <union2,3> | ALL    | NULL              | NULL    | NULL    | NULL  | NULL |                          |
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
```

#### UNION RESULT
UNION的结果。

```
mysql> explain select * from t3 where id=3952602 union all select * from t3 ;
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type  | table      | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | PRIMARY      | t3         | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
|  2 | UNION        | t3         | ALL   | NULL              | NULL    | NULL    | NULL  | 1000 |       |
|NULL | UNION RESULT | <union1,2> | ALL   | NULL              | NULL    | NULL    | NULL  | NULL |       |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
```

#### SUBQUERY

子查询中的第一个SELECT。
```
mysql> explain select * from t3 where id = (select id from t3 where id=3952602 )  ;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra       |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
|  1 | PRIMARY     | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |             |
|  2 | SUBQUERY    | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       |       |    1 | Using index |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
```

#### DEPENDENT SUBQUERY
子查询中的第一个SELECT，取决于外面的查询

```
mysql> explain select id from t3 where id in (select id from t3 where id=3952602 )  ;
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
| id | select_type        | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra                    |
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
|  1 | PRIMARY            | t3    | index | NULL              | PRIMARY | 4       | NULL  | 1000 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 | Using index              |
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
```

#### DERIVED
派生表的SELECT(FROM子句的子查询)

```
mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

### table

显示这一行的数据是关于哪张表的。有时不是真实的表名字，看到的是derivedx(x是个数字，我的理解是第几步执行的结果)

```
mysql> explain select * from (select * from ( select * from t3 where id=3952602) a) b;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | <derived3> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  3 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

### type

这列很重要，显示了连接使用了哪种类别,有无使用索引，从最好到最差的连接类型为const、eq_reg、ref、range、indexhe和ALL。

#### system
这是const联接类型的一个特例。表仅有一行满足条件.如下(t3表上的id是 primary key)

```
mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

#### const   

表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const表很快，因为它们只读取一次！

const用于用常数值比较PRIMARY KEY或UNIQUE索引的所有部分时。在下面的查询中，tbl_name可以用于const表：
```
SELECT * from tbl_name WHERE primary_key=1；
SELECT * from tbl_name WHERE primary_key_part1=1和 primary_key_part2=2；
```
例如：
```
mysql> explain select * from t3 where id=3952602;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
```

#### eq_ref

对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型，**它用在一个索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY**，eq_ref可以用于使用= 操作符比较的带索引的列。比较值可以为常量或一个使用在该表前面所读取的表的列的表达式。
```
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
    AND ref_table.key_column_part2=1;
```
例如：
```
mysql> create unique index  idx_t3_id on t3(id) ;
Query OK, 1000 rows affected (0.03 sec)
Records: 1000  Duplicates: 0  Warnings: 0

mysql> explain select * from t3,t4 where t3.id=t4.accountid;
+----+-------------+-------+--------+-------------------+-----------+---------+----------------------+------+-------+
| id | select_type | table | type   | possible_keys     | key       | key_len | ref                  | rows | Extra |
+----+-------------+-------+--------+-------------------+-----------+---------+----------------------+------+-------+
|  1 | SIMPLE      | t4    | ALL    | NULL              | NULL      | NULL    | NULL                 | 1000 |       |
|  1 | SIMPLE      | t3    | eq_ref | PRIMARY,idx_t3_id | idx_t3_id | 4       | dbatest.t4.accountid |    1 |       |
+----+-------------+-------+--------+-------------------+-----------+---------+----------------------+------+-------+
```

#### ref
对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取。如果联接只使用键的最左边的前缀，或如果键不是UNIQUE或PRIMARY KEY（换句话说，如果联接不能基于关键字选择单个行的话），则使用ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。ref可以用于使用=或<=>操作符的带索引的列。

在下面的例子中，MySQL可以使用ref联接来处理ref_tables：
```
SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
    AND ref_table.key_column_part2=1;
```
例如：

```
mysql> drop index idx_t3_id on t3;
Query OK, 1000 rows affected (0.03 sec)
Records: 1000  Duplicates: 0  Warnings: 0

mysql> create index idx_t3_id on t3(id) ;
Query OK, 1000 rows affected (0.04 sec)
Records: 1000  Duplicates: 0  Warnings: 0

mysql> explain select * from t3,t4 where t3.id=t4.accountid;
+----+-------------+-------+------+-------------------+-----------+---------+----------------------+------+-------+
| id | select_type | table | type | possible_keys     | key       | key_len | ref                  | rows | Extra |
+----+-------------+-------+------+-------------------+-----------+---------+----------------------+------+-------+
|  1 | SIMPLE      | t4    | ALL  | NULL              | NULL      | NULL    | NULL                 | 1000 |       |
|  1 | SIMPLE      | t3    | ref  | PRIMARY,idx_t3_id | idx_t3_id | 4       | dbatest.t4.accountid |    1 |       |
+----+-------------+-------+------+-------------------+-----------+---------+----------------------+------+-------+
2 rows in set (0.00 sec)
```

#### ref_or_null
该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行。在解决子查询中经常使用该联接类型的优化。

在下面的例子中，MySQL可以使用ref_or_null联接来处理ref_tables：
```
SELECT * FROM ref_table
WHERE key_column=expr OR key_column IS NULL;
```

#### index_merge
该联接类型表示使用了索引合并优化方法。在这种情况下，key列包含了使用的索引的清单，key_len包含了使用的索引的最长的关键元素。

例如：
```
mysql> explain select * from t4 where id=3952602 or accountid=31754306 ;
+----+-------------+-------+-------------+----------------------------+----------------------------+---------+------+------+------------------------------------------------------+
| id | select_type | table | type        | possible_keys              | key                        | key_len | ref  | rows | Extra                                                |
+----+-------------+-------+-------------+----------------------------+----------------------------+---------+------+------+------------------------------------------------------+
|  1 | SIMPLE      | t4    | index_merge | idx_t4_id,idx_t4_accountid | idx_t4_id,idx_t4_accountid | 4,4     | NULL |    2 | Using union(idx_t4_id,idx_t4_accountid); Using where |
+----+-------------+-------+-------------+----------------------------+----------------------------+---------+------+------+------------------------------------------------------+
1 row in set (0.00 sec)
```
#### unique_subquery
该类型替换了下面形式的IN子查询的ref：
```
value IN (SELECT primary_key FROM single_table WHERE some_expr)
```
unique_subquery是一个索引查找函数，可以完全替换子查询，效率更高。

#### index_subquery

该联接类型类似于unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：

```
value IN (SELECT key_column FROM single_table WHERE some_expr)
```

#### range

只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range。

```
mysql> explain select * from t3 where id=3952602 or id=3952603 ;
+----+-------------+-------+-------+-------------------+-----------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys     | key       | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+-------------------+-----------+---------+------+------+-------------+
|  1 | SIMPLE      | t3    | range | PRIMARY,idx_t3_id | idx_t3_id | 4       | NULL |    2 | Using where |
+----+-------------+-------+-------+-------------------+-----------+---------+------+------+-------------+
1 row in set (0.02 sec)
```

#### index
该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小，当查询只使用作为单索引一部分的列时，MySQL可以使用该联接类型。

#### ALL
对于每个来自于先前的表的行组合，进行完整的表扫描。如果表是第一个没标记const的表，这通常不好，并且通常在它情况下很差。通常可以增加更多的索引而不要使用ALL，使得行能基于前面的表中的常数值或列值被检索出。

### possible_keys
possible_keys列指出MySQL能使用哪个索引在该表中找到行。注意，该列完全独立于EXPLAIN输出所示的表的次序。这意味着在possible_keys中的某些键实际上不能按生成的表次序使用。

如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查WHERE子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用EXPLAIN检查查询

### key
key列显示MySQL实际决定使用的键（索引）。如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。

### key_len
key_len列显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。
使用的索引的长度。在不损失精确性的情况下，长度越短越好。

### ref
ref列显示使用哪个列或常数与key一起从表中选择行。

### rows
rows列显示MySQL认为它执行查询时必须检查的行数。

### Extra

该列包含MySQL解决查询的详细信息,下面详细.

#### Distinct
一旦MYSQL找到了与行相联合匹配的行，就不再搜索了

#### Not exists
MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行，就不再搜索了

#### Range checked for each

Record（index map:#）
没有找到理想的索引，因此对于从前面表中来的每一个行组合，MYSQL检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一

#### Using filesort
看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行

#### Using index
列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候

#### Using temporary
看到这个的时候，查询需要优化了。这里，MYSQL需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上

#### Using where
使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index，这就会发生，或者是查询有问题

## 参考资料
[MySQL EXPLAIN详解](https://www.jianshu.com/p/ea3fc71fdc45)
