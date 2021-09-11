---
title: MySQL优化
tags: 
 - mysql
categories:
 - 数据库
 - 调优
---

本文不涉及索引优化，索引优化参考[mysql索引为知笔记](wiz://open_document?guid=d6bb51d6-57c2-41f0-9475-84e20021d56d&kbguid=&private_kbguid=63531d60-bd31-11ea-aa67-1778e2b2bbbd). 



# sql语句上的优化

## 大批量插入数据

因为InnoDB类型的表是按照主键的顺序保存的，所以将导入的数据按照主键的顺序排列，可以有效的提高导入数据的效率。如果InnoDB表没有主键，那么系统会自动默认创建一个内部列作为主键，所以如果可以给表创建一个主键，将可以利用这点，来提高导入数据的效率。

在导入数据前执行 SET UNIQUE_CHECKS=0，关闭唯一性校验，在导入结束后执行SET UNIQUE_CHECKS=1，恢复唯一性校验，可以提高导入的效率。

如果应用使用自动提交的方式，建议在导入前执行 SET AUTOCOMMIT=0，关闭自动提交，导入结束后再执行 SET AUTOCOMMIT=1，打开自动提交，也可以提高导入的效率

## 优化insert语句

如果需要同时对一张表插入很多行数据时，应该尽量使用多个值表的insert语句，这种方式将大大的缩减客户端与数据库之间的连接、关闭等消耗。使得效率比分开执行的单个insert语句快。

示例， 原始方式为：

 

```sql
insert into tb_test values(1,'Tom'); 
insert into tb_test values(2,'Cat');
insert into tb_test values(3,'Jerry');
```

优化后的方案为 ：

```sql
insert into tb_test values(1,'Tom'),(2,'Cat')，(3,'Jerry'); 1
```

在事务中进行数据插入：

```sql
start transaction; 
insert into tb_test values(1,'Tom'); 
insert into tb_test values(2,'Cat');
insert into tb_test values(3,'Jerry'); 
commit;
```

数据有序插入：

```sql
insert into tb_test values(4,'Tim'); 
insert into tb_test values(1,'Tom'); 
insert into tb_test values(3,'Jerry'); 
insert into tb_test values(5,'Rose'); 
insert into tb_test values(2,'Cat');
-- 优化后
insert into tb_test values(1,'Tom'); 
insert into tb_test values(2,'Cat'); 
insert into tb_test values(3,'Jerry'); 
insert into tb_test values(4,'Tim');
insert into tb_test values(5,'Rose');
```

## 优化order by语句

环境准备：

```sql
CREATE TABLE `emp` (
    `id` INT (11) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR (100) NOT NULL,
    `age` INT (3) NOT NULL,
    `salary` INT (11) DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE = INNODB DEFAULT CHARSET = utf8mb4;
CREATE INDEX idx_emp_age_salary ON emp (age, salary);
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('1', 'Tom', '25', '2300');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('2', 'Jerry', '30', '3500');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('3', 'Luci', '25', '2800');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('4', 'Jay', '36', '3500');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('5', 'Tom2', '21', '2200');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('6', 'Jerry2', '31', '3300');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('7', 'Luci2', '26', '2700');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('8', 'Jay2', '33', '3500');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('9', 'Tom3', '23', '2400');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('10', 'Jerry3', '32', '3100');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('11', 'Luci3', '26', '2900');
INSERT INTO `emp` (`id`, `name`, `age`, `salary`)
VALUES
    ('12', 'Jay3', '37', '4500');
```

### 两种排序方式

 第一种是通过对返回数据进行排序，也就是通常说的 filesort 排序，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序：

 

```sql
mysql> explain select * from emp order by age;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```

 第二种通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要额外排序，操作效率高。

```sql
mysql> explain select id,age from emp order by age;
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key                | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | emp   | NULL       | index | NULL          | idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

多字段排序：

```sql
mysql> explain select id,age from emp order by age,salary;
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key                | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | emp   | NULL       | index | NULL          | idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select id,age from emp order by salary,age;
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type  | possible_keys | key                | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | emp   | NULL       | index | NULL          | idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index; Using filesort |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select id,age from emp order by age desc, salary;
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type  | possible_keys | key                | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | emp   | NULL       | index | NULL          | idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index; Using filesort |
+----+-------------+-------+------------+-------+---------------+--------------------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```

可以看出，最好按照覆盖索引的顺序排序，同时排序升降序保持一致。

### Filesort 的优化

通过创建合适的索引，能够减少 Filesort 的出现，但是在某些情况下，条件限制不能让Filesort消失，那就需要加快 Filesort的排序操作。对于Filesort ， MySQL 有两种排序算法：

- 两次扫描算法 ：MySQL4.1 之前，使用该方式排序。首先根据条件取出排序字段和行指针信息，然后在排序区sort buffer 中排序，如果sort buffer不够，则在临时表 temporary table 中存储排序结果。完成排序之后，再根据行指针回表读取记录，该操作可能会导致大量随机I/O操作
- 一次扫描算法：一次性取出满足条件的所有字段，然后在排序区 sort buffer 中排序后直接输出结果集。排序时内存开销较大，但是排序效率比两次扫描算法要高。

MySQL 通过比较系统变量 max_length_for_sort_data 的大小和Query语句取出的字段总大小， 来判定是否那种排序算法，如果max_length_for_sort_data 更大，那么使用第二种优化之后的算法；否则使用第一种。

可以适当提高 sort_buffer_size 和 max_length_for_sort_data 系统变量，来增大排序区的大小，提高排序的效率。

```sql
mysql> show variables like 'sort_buffer_size';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| sort_buffer_size | 262144 |
+------------------+--------+
1 row in set (0.01 sec)
mysql> show variables like 'max_length_for_sort_data';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| max_length_for_sort_data | 1024  |
+--------------------------+-------+
1 row in set (0.01 sec)
```

## 优化group by 语句

由于GROUP BY 实际上也同样会进行排序操作，而且与ORDER BY 相比，GROUP BY 主要只是多了排序之后的分组操作。当然，如果在分组的时候还使用了其他的一些聚合函数，那么还需要一些聚合函数的计算。所以，在GROUP BY 的实现过程中，与 ORDER BY 一样也可以利用到索引。

如果查询包含 group by 但是用户想要避免排序结果的消耗， 则可以执行order by null 禁止排序。如下 ：

 

```sql
mysql> drop index idx_emp_age_salary on  emp;
mysql> explain select age,count(*) from emp group by age;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select age,count(*) from emp group by age order by null;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | SIMPLE      | emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | Using temporary |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
1 row in set, 1 warning (0.00 sec)
mysql> create index idx_emp_age_salary on emp(age,salary);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> explain select age,count(*) from emp group by age order by null;
+----+-------------+-------+------------+-------+--------------------+--------------------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys      | key                | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+--------------------+--------------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | emp   | NULL       | index | idx_emp_age_salary | idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index |
+----+-------------+-------+------------+-------+--------------------+--------------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

##  优化嵌套查询

Mysql4.1版本之后，开始支持SQL的子查询。这个技术可以使用SELECT语句来创建一个单列的查询结果，然后把这个结果作为过滤条件用在另一个查询中。使用子查询可以一次性的完成很多逻辑上需要多个步骤才能完成的SQL操作，同时也可以避免事务或者表锁死，并且写起来也很容易。但是，有些情况下，子查询是可以被更高效的连接（JOIN）替代。

```sql
mysql> explain select * from t_user where id in (select user_id from user_role );
+----+--------------+-------------+------------+--------+---------------+---------------+---------+----------------+------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys | key           | key_len | ref            | rows | filtered | Extra       |
+----+--------------+-------------+------------+--------+---------------+---------------+---------+----------------+------+----------+-------------+
|  1 | SIMPLE       | t_user      | NULL       | ALL    | PRIMARY       | NULL          | NULL    | NULL           |    6 |   100.00 | Using where |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_key>    | <auto_key>    | 99      | test.t_user.id |    1 |   100.00 | NULL        |
|  2 | MATERIALIZED | user_role   | NULL       | index  | fk_ur_user_id | fk_ur_user_id | 99      | NULL           |    6 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+---------------+---------------+---------+----------------+------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)
mysql> explain select * from t_user u , user_role ur where u.id = ur.user_id;
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref             | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------------+
|  1 | SIMPLE      | ur    | NULL       | ALL    | fk_ur_user_id | NULL    | NULL    | NULL            |    6 |   100.00 | Using where |
|  1 | SIMPLE      | u     | NULL       | eq_ref | PRIMARY       | PRIMARY | 98      | test.ur.user_id |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```

连接(Join)查询之所以更有效率一些 ，是因为MySQL不需要在内存中创建临时表来完成这个逻辑上需要两个步骤的查询工作。

## 优化OR条件

对于包含OR的查询子句，如果要利用索引，则OR之间的每个条件列都必须用到索引 ， 而且不能使用到复合索引； 如果没有索引，则应该考虑增加索引。

```sql
mysql> explain select * from emp where id = 1 or age = 30 ;
+----+-------------+-------+------------+-------------+----------------------------+----------------------------+---------+------+------+----------+-----------------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys              | key                        | key_len | ref  | rows | filtered | Extra                                                     |
+----+-------------+-------+------------+-------------+----------------------------+----------------------------+---------+------+------+----------+-----------------------------------------------------------+
|  1 | SIMPLE      | emp   | NULL       | index_merge | PRIMARY,idx_emp_age_salary | idx_emp_age_salary,PRIMARY | 4,4     | NULL |    2 |   100.00 | Using sort_union(idx_emp_age_salary,PRIMARY); Using where |
+----+-------------+-------+------------+-------------+----------------------------+----------------------------+---------+------+------+----------+-----------------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from emp where id = 1 or id = 3 ;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | emp   | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    2 |   100.00 | Using where |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from emp where id = 1 union select * from emp where id=3;
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
| id | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra           |
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY      | emp        | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL            |
|  2 | UNION        | emp        | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
3 rows in set, 1 warning (0.00 sec)
```

or可以使用union来替代。

## 优化分页查询

一般分页查询时，通过创建覆盖索引能够比较好地提高性能。一个常见又非常头疼的问题就是 limit 2000000,10 ，此时需要MySQL排序前2000010 记录，仅仅返回2000000 - 2000010 的记录，其他记录丢弃，查询排序的代价非常大 。

方案1：在索引上完成排序分页操作，最后根据主键关联回原表查询所需要的其他列内容。

```sql
se1ect * from tB t,(select id from tB order by id limit 2000000, 10 )a where t.id=a.id
```

方案2：该方案适用于主键自增的表，可以把Limit 查询转换成某个位置的查询 。

```sql
select * from tb where tb.id>2000000 limit 10
```

## 查询时指定索引

在查询语句中表名的后面，添加 use index 来提供希望MySQL去参考的索引列表，就可以让MySQL不再考虑其他可用的索引

```sql
mysql> create index idx_seller_name on tb_seller(name);
Query OK, 0 rows affected (0.06 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> explain select * from tb_seller  where name = '小米科 技';
+----+-------------+-----------+------------+------+------------------------------------------+--------------------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys                            | key                      | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+------------------------------------------+--------------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr,idx_seller_name | idx_seller_name_sta_addr | 403     | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+------------------------------------------+--------------------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller  use index(idx_seller_name ) where name = '小米科 技';
+----+-------------+-----------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys   | key             | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name | idx_seller_name | 403     | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller  ignore  index(idx_seller_name ) where name = '小米科 技';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
mysql> create index idx_seller_address on tb_seller(address);
Query OK, 0 rows affected, 1 warning (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 1
mysql> explain select * from tb_seller  where  address='北京市';
+----+-------------+-----------+------------+------+------------------------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys                | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+------------------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | idex_addr,idx_seller_address | NULL | NULL    | NULL |    4 |    75.00 | Using where |
+----+-------------+-----------+------------+------+------------------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller use index(idx_seller_address)  where  address='北京市';
+----+-------------+-----------+------------+------+--------------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys      | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+--------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | idx_seller_address | NULL | NULL    | NULL |    4 |    75.00 | Using where |
+----+-------------+-----------+------------+------+--------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller force  index(idx_seller_address)  where  address='北京市';
+----+-------------+-----------+------------+------+--------------------+--------------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys      | key                | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+--------------------+--------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_address | idx_seller_address | 403     | const |    3 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+--------------------+--------------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```



# 其他知识

## count(\*) 和 count(1)和count(列名)

```sql
INSERT INTO `test`.`count_t` (`a`, `b`) VALUES ('1', '1');
INSERT INTO `test`.`count_t` (`a`, `b`) VALUES ('2', NULL);
INSERT INTO `test`.`count_t` (`a`, `b`) VALUES (NULL, NULL);

select count(1) from count_t; -- 3
select count(*) from count_t; -- 3 
select count(a) from count_t; -- 2 
```

* count(1) 统计表中的所有的记录数，包含字段为null 的记录。
* count(字段)会统计该字段在表中出现的次数，忽略字段为null 的情况

* 列名为主键，count(列名)会比count(1)快  
* 如果表多个列并且没有主键，则 count（1） 的执行效率优于 count（*） 
