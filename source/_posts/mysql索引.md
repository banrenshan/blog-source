---
title: MySQL索引
tags: 
 - mysql
categories:
 - 数据库
 - 索引
---

# 概述

MySQL官方对索引的定义为高效获取数据的有序数据结构。在数据之外,数据库系统还维着满足特定查找算法的数据结构,这些数据结构以某种方式引用(指向)数据，这样就可以在这些数据结构上实现高级查找算法，这种数据结构就是索引。如下面的示意图所示:

![img](./mysql%E7%B4%A2%E5%BC%95/406ba3a9-c7e5-4b06-bed1-c58401976d67.jpg)

![img](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)

右图并不是mysql B+树索引的实现方式，只是举例说明索引的工作机制。

左边是数据表, 一共有两列七条记录,最左边的是数据记录的物理地址(注意逻辑上相邻的记录不一定物理上相邻)。为了加快Col2的查找,可以维护一个右边所示的二叉查找树,每个节点分别包含索引键值和一个指向对应数据记录物理地址的指针,这样就可以运用二叉查找快速获取到相应数据。

一般来说索引本身也很大,不可能全部存储在内存中,因此索引往往以文件的形式存储在磁盘上。

## 索引的优缺点

优点：

- 提高查询效率，降低数据库的IO成本
- 因为索引本身的排序特性，降低数据的排序成本，降低CPU消耗

缺点：

- 索引也是要消耗空间的
- 虽然索引大大提高了查询效率,同时却也降低了更新表的速度。在对表进行INSERT. UPDATE、 DELETE时，MySQL不仅要保存数据，还要更新索引结构

## 索引分类

- 单值索引:即索引只包含单个列,一个表可以有多个单列索引
- 唯一索引:索引列的值必须唯一 ,但允许有空值
- 复合索引: 即一个索引包含多个列

- 聚合索引: 索引中键值的逻辑顺序决定了表中相应行的物理顺序（索引中的数据物理存放地址和索引的顺序是一致的）。InnoDB引擎会为每张表都加一个聚集索引，而聚集索引指向的的数据又是以物理磁盘顺序来存储的，自增的主键会把数据自动向后插入，避免了插入过程中的聚集索引排序问题。如果对聚集索引进行排序，这会带来磁盘IO性能损耗是非常大的。

- - 如果一个主键被定义了，那么这个主键就是作为聚集索引
  - 如果没有主键被定义，那么该表的第一个唯一非空索引被作为聚集索引
  - 如果没有主键也没有合适的唯一索引，那么innodb内部会生成一个隐藏的主键作为聚集索引，这个隐藏的主键是一个6个字节的列，改列的值会随着数据的插入自增。

- 覆盖索引: 指一个查询语句的执行只用从索引中就能够取得，不必从数据表中读取。
- 前缀索引: 就是基于原始索引字段，截取前面指定的字符个数或者字节数来做的索引。

- 稠密索引：每个索引键值都对应有一个索引项，稀疏索引只为某些搜索码值建立索引记录；在搜索时，找到其最大的搜索码值小于或等于所查找记录的搜索码值的索引项，然后从该记录开始向后顺序查询直到找到为止。 

![img](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)PRIMARY 其实就是一个名称为PRIMARY的B树索引。如果表没有PRIMARY，InnoDB会为每一行生成一个6字节的ROWID，并以此作为主键。

# 索引实现

索引是在MySQL的存储引擎层中实现的,而不是在服务器层实现的。所以每种存储引擎的索引都不完全相同。MySQL目前提供了以下4种索引:

- BTREE索引:最常见的索引类型。大部分索引都支持B+树索引。
- HASH索引:只有Memory引擎支持, 使用场景简单。
- R-tree索引(空间索引) :空间索引是MyISAM引擎的一个特殊索引类型,主要用于地理空间数据类型,通常使用较少,不做特别介绍。
- Full-text (全文索引) :文索引也是MyISAM的一个特殊索引类型,主要用于全文索引, InnoDB从Mysql5.6版本开始支持全文索引。

| 索引      | InnoDB      | MyISAM | Memory |
| --------- | ----------- | ------ | ------ |
| Btree     | 支持        | 支持   | 支持   |
| Hash      | 不支持      | 不支持 | 支持   |
| R-tree    | 不支持      | 支持   | 不支持 |
| full-text | 5.6以后支持 | 支持   | 不支持 |

我们平常所说的索引,如果没有特别指明,都是指B+树(多路搜索树)结构组织的索引。其中聚集索引、复合索引、前缀索引、唯一索引默认都是使用B+tree索引,统称为索引。

## 全文索引

**创建索引**：

```sql
CREATE TABLE articles (
    id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
    title VARCHAR (200),
    body TEXT,
    FULLTEXT (title, body) WITH PARSER ngram
) ENGINE = INNODB DEFAULT CHARSET=utf8mb4 COMMENT='文章表';
ALTER TABLE articles ADD FULLTEXT INDEX title_body_index (title,body) WITH PARSER ngram;
```

**使用索引查找数据**：

```sql
mysql> SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('精神');
+----+-----------------+-------------------------+
| id | title           | body                    |
+----+-----------------+-------------------------+
|  1 | 弘扬正能量      | 贯彻党的18大精神        |
+----+-----------------+-------------------------+
```

**高级查询**：

```sql
mysql> SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+精神 -贯彻' IN BOOLEAN MODE);
```

\+ 表示必须包含 , -表示必须不包含,默认代表可以出现可以不出现,但是出现时在查询结果集中的排名较高一些.也就是该结果和搜索词的相关性高一些. BOOLEAN mode表示且的关系。 去除IN后面的语句，表示或的关系。

## hash索引

假如有一个非常非常大的表，比如用户登录时需要通过email检索出用户，如果直接在email列建索引，除了索引区间匹配，还要进行字符串匹配比对，email短还好，如果长的话这个查询代价就比较大。若此时，在email建立哈希索引，性能就比字符串比对查询快多了。

## B树索引

我们来讲解mysql的底层实现B+树之前，先来看看B树，B+树是对B树的改造，两者各有应用场景，不区分优劣。

### B树

BTree又叫多路平衡搜索树，一棵m叉的BTree特性如下:

- 树中每个节点最多包含m个孩子。
- 除根节点与叶子节点外.每个节点至少有[ceil(m/2]个孩子（向上取整）。
- 若根节点不是叶子节点。则至少有两个孩子。
- 所有的叶子节点都在同一层。
- 每个非叶子节点由n个key与n+1个指针组成,其中[cil(m/2)-1]<=n <= m-1 .

以5叉BTree为例, key的数量由公式[cell(m/2)-1]<=n<= m-1推导。所以2 <=n<=4。当n>4时,中间节点分裂到父节点，两边节点分裂。以插入CNGAHEKQMFWLTZDPRXYS数据为例。演变过程如下:

![img](./mysql%E7%B4%A2%E5%BC%95/77ed9535-11f4-4777-81a0-bc26e6607eee.jpg)

![img](./mysql%E7%B4%A2%E5%BC%95/b65bb9ac-a1c6-452f-8cff-6e1506c43387.jpg)

![img](./mysql%E7%B4%A2%E5%BC%95/2bfdb14b-8c26-4789-87d3-781d525a9ace.jpg)

![img](./mysql%E7%B4%A2%E5%BC%95/05cf2ba1-7968-45d6-8bde-9063595e8418.jpg)

### B+树

- n叉B+Tree最多含有n个key ,而BTree最多含有n-1个key。
- B+Tree的叶子节点保存数据信息,按照key大小顺序排列。
- 所有的非叶子节点都可以看作是key的索引部分。

如下图所示：

![img](./mysql%E7%B4%A2%E5%BC%95/fc582bce-6a87-49e4-82a1-2045d04c9cc9.jpg)

我们来看B+树如何插入数据：

1. 若为空树，那么创建一个节点并将记录插入其中，此时这个叶子结点也是根结点，插入操作结束。

![img](./mysql%E7%B4%A2%E5%BC%95/0.26820642709678083.png)

2. 针对叶子类型结点：根据key值找到叶子结点，向这个叶子结点插入记录。插入后，若当前结点key的个数小于等于m-1（5-1 = 4），则插入结束。否则将这个叶子结点分裂成左右两个叶子结点，左叶子结点包含前m/2个（2个）记录，右结点包含剩下的记录，将第m/2+1个（3个）记录的key进位到父结点中（父结点一定是索引类型结点），进位到父结点的key左孩子指针向左结点，右孩子指针向右结点。将当前结点的指针指向父结点，然后执行第3步。

1. 1. 依次插入8，10，15。

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.8833728135591741.png)

   2. 插入16 。插入16后超过了关键字的个数限制，所以要进行分裂。在叶子结点分裂时，分裂出来的左结点2个记录，右边3个记录，中间第三个数成为索引结点中的key（10），分裂后当前结点指向了父结点（根结点）。结果如下图所示。

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.06741745962404759.png)

3. 针对索引类型结点：若当前结点key的个数小于等于m-1（4），则插入结束。否则，将这个索引类型结点分裂成两个索引结点，左索引结点包含前(m-1)/2个key（2个），右结点包含m-(m-1)/2个key（3个），将第m/2个key进位到父结点中，进位到父结点的key左孩子指向左结点,，进位到父结点的key右孩子指向右结点。将当前结点的指针指向父结点，然后重复第3步。

1. 1. 插入17 

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.06569865214238471.png)

   2. 插入18，当前结点的关键字个数大于5，进行分裂。分裂成两个结点，左结点2个记录，右结点3个记录，关键字16进位到父结点（索引类型）中，将当前结点的指针指向父结点。

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.06219906456896608.png)

   3. 插入若干数据后

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.5311225259410879.png)

   4. 在上图中插入7，当前结点的关键字个数超过4，需要分裂。左结点2个记录，右结点3个记录。分裂后关键字7进入到父结点中，将当前结点的指针指向父结点，结果如下图所示

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.8105587102045797.png)

      当前结点的关键字个数超过4，需要继续分裂。左结点2个关键字，右结点2个关键字，关键字16进入到父结点中，将当前结点指向父结点，结果如下图所示。

      ![img](./mysql%E7%B4%A2%E5%BC%95/0.9891318328299135.png)



# 索引语法

## 创建索引

```sql
CREATE [UNIQUE | FULLTEXT | SPATIAL] INDEX index_name
    [index_type]
    ON table_name (key_part,...)
    [index_option]
    [algorithm_option | lock_option] ...
key_part:
    col_name [(length)] [ASC | DESC]
index_option: {
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'
}
index_type:
    USING {BTREE | HASH}
algorithm_option:
    ALGORITHM [=] {DEFAULT | INPLACE | COPY}
lock_option:
    LOCK [=] {DEFAULT | NONE | SHARED | EXCLUSIVE}
```

## 查看索引：

```sql
SHOW INDEX FROM  tbl_name
```

## 删除索引：

```sql
DROP INDEX index_name ON tbl_name
```

## 修改表上索引：

```sql
ALTER TABLE tbl_name
    -- 添加索引
    ADD INDEX  index_name [index_type] (key_part,...) [index_option] 
    -- 删除索引
    DROP INDEX index_name
```

# 索引设计原则

- 对查询频次较高,且数据量比较大的表建立索引。
- 索引字段的选择,最佳候选列应当从where子句的条件中提取,如果where子句中的组合比较多,那么应当挑选最常用、过滤效果最好的列或其组合
- 使用唯一索引，区分度越高,使用索引的效率越高。
- 索引可以有效的提升查询数据的效率,但索引数量不是多多益善,索引越多,维护索引的代价越大。对于插入、更新、删除等DML操作比较频繁的表来说,索引过多,会引入相当高的维护代价。另外索引过多的话, MySQL也会犯选择困难病,虽然最终仍然会找到一个可用的索引,但无疑提高了选择的代价。
- 使用短索引,索引创建之后也是使用硬盘来存储的,因此提升索引访问的I/O效率,也可以提升总体的访问效率。假如构成索引的字段总长度比较短,那么在给定大小的存储块内可以存储更多的索引值,相应的可以有效的提升MySQL访问索引|的I/O效率。
- 利用最左前缀, N个列组合而成的组合索引,那么相当于是创建了N个索引,如果查询时where子句中使用了组成该索引的前几个字段,那么这条查询SQL可以利用组合索引来提升查询效率。

下面的章节，我们将讲述索引调优的内容

# explain

参考：[mysql explain详解 - 云+社区 - 腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1093229)

```sql
mysql> explain select * from z_class ;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | z_class | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
```

我们来学习一下各个字段的含义，先创建下表：

```sql
CREATE TABLE `t_role` (
    `id` VARCHAR (32) NOT NULL,
    `role_name` VARCHAR (255) DEFAULT NULL,
    `role_code` VARCHAR (255) DEFAULT NULL,
    `description` VARCHAR (255) DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_role_name` (`role_name`)
) ENGINE = INNODB DEFAULT CHARSET = utf8;
CREATE TABLE `t_user` (
    `id` VARCHAR (32) NOT NULL,
    `username` VARCHAR (45) NOT NULL,
    `password` VARCHAR (96) NOT NULL,
    `name` VARCHAR (45) NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_user_username` (`username`)
) ENGINE = INNODB DEFAULT CHARSET = utf8;
CREATE TABLE `user_role` (
    `id` INT (11) NOT NULL auto_increment,
    `user_id` VARCHAR (32) DEFAULT NULL,
    `role_id` VARCHAR (32) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `fk_ur_user_id` (`user_id`),
    KEY `fk_ur_role_id` (`role_id`),
    CONSTRAINT `fk_ur_role_id` FOREIGN KEY (`role_id`) REFERENCES `t_role` (`id`) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT `fk_ur_user_id` FOREIGN KEY (`user_id`) REFERENCES `t_user` (`id`) ON DELETE NO ACTION ON UPDATE NO ACTION
) ENGINE = INNODB DEFAULT CHARSET = utf8;
CREATE TABLE `z_class` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `NAME` varchar(20) NOT NULL COMMENT '名称',
  PRIMARY KEY (`id`),
  KEY `NAME` (`NAME`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;
create table addr(
    id int primary key auto_increment,
  province varchar(20) ,
  city VARCHAR(20),
  area varchar(20),
  UNIQUE KEY (province,city,area)
)ENGINE = INNODB DEFAULT CHARSET = utf8;
insert into `t_user` (`id`, `username`, `password`, `name`) values('1','super','$2a$10$TJ4TmCdK.X4wv/tCqHW14.w70U3CC33CeVncD3SLmyMXMknstqKRe',' 超级管理员');
insert into `t_user` (`id`, `username`, `password`, `name`) values('2','admin','$2a$10$TJ4TmCdK.X4wv/tCqHW14.w70U3CC33CeVncD3SLmyMXMknstqKRe',' 系统管理员'); 
insert into `t_user` (`id`, `username`, `password`, `name`) values('3','itcast','$2a$10$8qmaHgUFUAmPR5pOuWhYWOr291WJYjHelUlYn07k5ELF8ZCrW0Cui', 'test02'); 
insert into `t_user` (`id`, `username`, `password`, `name`) values('4','stu1','$2a$10$pLtt2KDAFpwTWLjNsmTEi.oU1yOZyIn9XkziK/y/spH5rftCpUMZa','学 生1'); 
insert into `t_user` (`id`, `username`, `password`, `name`) values('5','stu2','$2a$10$nxPKkYSez7uz2YQYUnwhR.z57km3yqKn3Hr/p1FR6ZKgc18u.Tvqm','学 生2'); 
insert into `t_user` (`id`, `username`, `password`, `name`) values('6','t1','$2a$10$TJ4TmCdK.X4wv/tCqHW14.w70U3CC33CeVncD3SLmyMXMknstqKRe','老师 1'); 
INSERT INTO `t_role` (`id`, `role_name`, `role_code`, `description`) VALUES('5','学 生','student','学生'); 
INSERT INTO `t_role` (`id`, `role_name`, `role_code`, `description`) VALUES('7','老 师','teacher','老师'); 
INSERT INTO `t_role` (`id`, `role_name`, `role_code`, `description`) VALUES('8','教 学管理员','teachmanager','教学管理员'); 
INSERT INTO `t_role` (`id`, `role_name`, `role_code`, `description`) VALUES('9','管 理员','admin','管理员'); 
INSERT INTO `t_role` (`id`, `role_name`, `role_code`, `description`) VALUES('10','超 级管理员','super','超级管理员');
INSERT INTO user_role(id,user_id,role_id) VALUES(NULL, '1', '5'),(NULL, '1', '7'), (NULL, '2', '8'),(NULL, '3', '9'),(NULL, '4', '8'),(NULL, '5', '10') ;
```

数据如下：

t_user表：

| 1    | super  | $2a$10$TJ4TmCdK.X4wv/tCqHW14.w70U3CC33CeVncD3SLmyMXMknstqKRe | 超级管理员 |
| ---- | ------ | ------------------------------------------------------------ | ---------- |
| 2    | admin  | $2a$10$TJ4TmCdK.X4wv/tCqHW14.w70U3CC33CeVncD3SLmyMXMknstqKRe | 系统管理员 |
| 3    | itcast | $2a$10$8qmaHgUFUAmPR5pOuWhYWOr291WJYjHelUlYn07k5ELF8ZCrW0Cui | test02     |
| 4    | stu1   | $2a$10$pLtt2KDAFpwTWLjNsmTEi.oU1yOZyIn9XkziK/y/spH5rftCpUMZa | 学 生1     |
| 5    | stu2   | $2a$10$nxPKkYSez7uz2YQYUnwhR.z57km3yqKn3Hr/p1FR6ZKgc18u.Tvqm | 学 生2     |
| 6    | t1     | $2a$10$TJ4TmCdK.X4wv/tCqHW14.w70U3CC33CeVncD3SLmyMXMknstqKRe | 老师 1     |

t_role表：

| 1    | 1    | 5    |
| ---- | ---- | ---- |
| 2    | 1    | 7    |
| 3    | 2    | 8    |
| 4    | 3    | 9    |
| 5    | 4    | 8    |
| 6    | 5    | 10   |

## id

id数字在多表查询时用来标识查询的顺序，例如t1的id是1 ，t2的id是2，则先查询t2,再查询t1。我们来看具体的例子：

```sql
mysql> explain select * from t_user where id = (select user_id from user_role where id=1);
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | PRIMARY     | t_user    | NULL       | const | PRIMARY       | PRIMARY | 98      | const |    1 |   100.00 | NULL  |
|  2 | SUBQUERY    | user_role | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
```

id相同时，根据表中的顺序确定优先级：

```sql
mysql> explain select * from t_user u join user_role t on u.id=t.user_id where u.id=1;
+----+-------------+-------+------------+------+---------------+---------------+---------+-----------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key           | key_len | ref       | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+---------------+---------+-----------+------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | ALL  | PRIMARY       | NULL          | NULL    | NULL      |    6 |    16.67 | Using where |
|  1 | SIMPLE      | t     | NULL       | ref  | fk_ur_user_id | fk_ur_user_id | 99      | test.u.id |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+---------------+---------+-----------+------+----------+-------------+
```

## select_type

用来表示查询的类型，可能的值如下表

| 类型         | 说明                                                         | 备注 |
| ------------ | ------------------------------------------------------------ | ---- |
| simple       | 简单的select查询，查询中不包含子查询或者UNION                |      |
| primay       | 查询中若包含任何复杂的子查询，最外层查询标记为该标识         |      |
| subquery     | 在SELECT 或 WHERE 列表中包含了子查询                         |      |
| derived      | 在FROM 列表中包含的子查询，被标记为 DERIVED（衍生） MYSQL会递归执行这些子查询，把结果放在临时表中 |      |
| union        | 若第二个SELECT出现在UNION之后，则标记为UNION ； 若UNION包含在FROM子句的子查询中，外层SELECT将被标记为 ： DERIVED |      |
| union result | 从UNION表获取结果的SELECT                                    |      |

```sql
mysql> explain select * from t_user;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | t_user | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
mysql> explain select * from t_user where id = (select user_id from user_role where id=1);
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | PRIMARY     | t_user    | NULL       | const | PRIMARY       | PRIMARY | 98      | const |    1 |   100.00 | NULL  |
|  2 | SUBQUERY    | user_role | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
mysql> explain select * from ( select * from t_user where id= 1 union select * from t_user where id= 2) t;
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | PRIMARY      | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | NULL            |
|  2 | DERIVED      | t_user     | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |    6 |    16.67 | Using where     |
|  3 | UNION        | t_user     | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |    6 |    16.67 | Using where     |
| NULL | UNION RESULT | <union2,3> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
```

## table

这一列表示 explain 的一行正在访问哪个表。

当 from 子句中有子查询时，table列是 <derivenN> 格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。当有 union 时，UNION RESULT 的 table 列的值为 <union1,2>，1和2表示参与 union 的 select 行id。

## type

type 显示的是访问类型，是较为重要的一个指标，可取值为：

| type   | 含义                                                         |      |
| ------ | ------------------------------------------------------------ | ---- |
| null   | MySQL不访问任何表，索引，直接返回结果                        |      |
| system | 表只有一行记录(等于系统表)，这是const类型的特例，一般不会出现 |      |
| const  | 表示通过索引一次就找到了，const 用于比较primary key 或者 unique 索引。因为只匹配一行数 据，所以很快。如将主键置于where列表中，MySQL 就能将该查询转换为一个const。 |      |
| eq_ref | 类似ref，区别在于使用的是唯一索引，使用主键的关联查询，关联查询出的记录只有一条。常见于主键或唯一索引扫描 |      |
| ref    | 非唯一性索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，返回所有匹配某个单独值的所有行（多个） |      |
| range  | 只检索给定返回的行，使用一个索引来选择行。 where 之后出现 between ， < , > , in 等操作。 |      |
| index  | index 与 ALL的区别为 index 类型只是遍历了索引树， 通常比ALL 快， ALL 是遍历数据文件。 |      |
| all    | 将遍历全表以找到匹配的行                                     |      |

一般来说，我们需要达到range级别，最好达到ref.



```sql
mysql> explain select 1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
mysql> explain select * from t_user where username='admin';
+----+-------------+--------+------------+-------+----------------------+----------------------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type  | possible_keys        | key                  | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+-------+----------------------+----------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t_user | NULL       | const | unique_user_username | unique_user_username | 137     | const |    1 |   100.00 | NULL  |
+----+-------------+--------+------------+-------+----------------------+----------------------+---------+-------+------+----------+-------+
mysql> explain select * from t_user m join user_role n where m.id=n.user_id;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------------+
|  1 | SIMPLE      | n     | NULL       | ALL    | fk_ur_user_id | NULL    | NULL    | NULL           |    6 |   100.00 | Using where |
|  1 | SIMPLE      | m     | NULL       | eq_ref | PRIMARY       | PRIMARY | 98      | test.n.user_id |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------------+
mysql> explain select * from z_class where name='二班';
+----+-------------+---------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | z_class | NULL       | ref  | NAME          | NAME | 62      | const |    2 |   100.00 | Using index |
+----+-------------+---------+------------+------+---------------+------+---------+-------+------+----------+-------------+
mysql> explain select * from user_role where id>2;
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user_role | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    4 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
mysql> explain select username from t_user where username!='aa';
+----+-------------+--------+------------+-------+----------------------+----------------------+---------+------+------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys        | key                  | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------+----------------------+----------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | t_user | NULL       | index | unique_user_username | unique_user_username | 137     | NULL |    6 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+----------------------+----------------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

## key

- possible_keys : 显示可能应用在这张表的索引， 一个或多个。
- key ： 实际使用的索引， 如果为NULL， 则没有使用索引。
- key_len : 表示索引中使用的字节数， 该值为索引字段最大可能长度，并非实际使用长度，在不损失精确性的前提下， 长度越短越好 。

key_len计算规则如下：

- 字符串 

- - char(n)：n字节长度
  - varchar(n)：2字节存储字符串长度，如果是utf-8，则长度 3n + 2

- 数值类型 

- - tinyint：1字节
  - smallint：2字节
  - int：4字节
  - bigint：8字节　　

- 时间类型　 

- - date：3字节
  - timestamp：4字节
  - datetime：8字节

![img](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)如果字段允许为 NULL，需要1字节记录是否为 NULL

![img](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。

20*3+2

 

```sql
mysql> explain select * from addr where province='上海' ;
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | addr  | NULL       | ref  | province      | province | 63      | const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from addr where province='上海' and city='上海';
+----+-------------+-------+------------+------+---------------+----------+---------+-------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref         | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | addr  | NULL       | ref  | province      | province | 126     | const,const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from addr where province='上海' and city='上海' and area='普通';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | no matching row in const table |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from addr where province='上海' and city='上海' and area='浦东';
+----+-------------+-------+------------+-------+---------------+----------+---------+-------------------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref               | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+----------+---------+-------------------+------+----------+-------------+
|  1 | SIMPLE      | addr  | NULL       | const | province      | province | 189     | const,const,const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+----------+---------+-------------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from addr where province='上海'  and area='浦东';
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | addr  | NULL       | ref  | province      | province | 63      | const |    1 |    50.00 | Using where; Using index |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.01 sec)
mysql> explain select * from addr where  city='上海' and area='浦东';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | addr  | NULL       | index | NULL          | province | 189     | NULL |    2 |    50.00 | Using where; Using index |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from addr where   area='浦东';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | addr  | NULL       | index | NULL          | province | 189     | NULL |    2 |    50.00 | Using where; Using index |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

## rows

表示扫描的行数

## extra

| extra           | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| using filesort  | 说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取， 称为“文件排序”, 效率低。 |
| using temporary | 使用了临时表保存中间结果，MySQL在对查询结果排序时使用临时表。常见于 order by 和 group by； 效率低 |
| using index     | 返回的列数据只使用了索引中的信息，而没有再去访问表中的行记录，是性能高的表现。 |
| Using where     | mysql服务器将在存储引擎检索行后再进行过滤。就是先读取整行数据，再按 where 条件进行检查，符合就留下，不符合就丢弃。 |
| distinct        | 一旦mysql找到了与行相联合匹配的行，就不再搜索了              |

# 索引失效的情况

准备环境：

```sql
CREATE TABLE `tb_seller` (
    `sellerid` VARCHAR (100),
    `name` VARCHAR (100),
    `nickname` VARCHAR (50),
    `password` VARCHAR (60),
    `status` VARCHAR (1),
    `address` VARCHAR (100),
    `createtime` datetime,
    PRIMARY KEY (`sellerid`)
) ENGINE = INNODB DEFAULT charset = utf8mb4;
create index idx_seller_name_sta_addr on tb_seller(name,status,address);
INSERT INTO `tb_seller` (
    `sellerid`,
    `name`,
    `nickname`,
    `password`,
    `status`,
    `address`,
    `createtime`
)
VALUES
    (
        'alibaba',
        '阿里巴巴',
        '阿里小 店',
        'e10adc3949ba59abbe56e057f20f883e',
        '1',
        '杭州市',
        '2088-01-01 12:00:00'
    );
INSERT INTO `tb_seller` (
    `sellerid`,
    `name`,
    `nickname`,
    `password`,
    `status`,
    `address`,
    `createtime`
)
VALUES
    (
        'baidu',
        '百度科技有限公司',
        '百度小 店',
        'e10adc3949ba59abbe56e057f20f883e',
        '1',
        '北京市',
        '2088-01-01 12:00:00'
    );
INSERT INTO `tb_seller` (
    `sellerid`,
    `name`,
    `nickname`,
    `password`,
    `status`,
    `address`,
    `createtime`
)
VALUES
    (
        'huawei',
        '华为科技有限公司',
        '华为小 店',
        'e10adc3949ba59abbe56e057f20f883e',
        '0',
        '北京市',
        '2088-01-01 12:00:00'
    );
INSERT INTO `tb_seller` (
    `sellerid`,
    `name`,
    `nickname`,
    `password`,
    `status`,
    `address`,
    `createtime`
)
VALUES
    (
        'itcast',
        '传智播客教育科技有限公司',
        '传智播 客',
        'e10adc3949ba59abbe56e057f20f883e',
        '1',
        '北京市',
        '2088-01-01 12:00:00'
    );
```

| sellerid | name                     | nickname  | password                         | status | address | createtime     |
| -------- | ------------------------ | --------- | -------------------------------- | ------ | ------- | -------------- |
| alibaba  | 阿里巴巴                 | 阿里小 店 | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市  | 2088/1/1 12:00 |
| baidu    | 百度科技有限公司         | 百度小 店 | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市  | 2088/1/1 12:00 |
| huawei   | 华为科技有限公司         | 华为小 店 | e10adc3949ba59abbe56e057f20f883e | 0      | 北京市  | 2088/1/1 12:00 |
| itcast   | 传智播客教育科技有限公司 | 传智播 客 | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市  | 2088/1/1 12:00 |

## 违背左前缀法则

```sql
mysql> explain select * from tb_seller where status='1';
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |    25.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller where name='阿里巴巴' and status='1';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref         | rows | filtered | Extra |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 410     | const,const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

如果符合最左法则，但是出现跳跃某一列，只有最左列索引生效：

 

```sql
mysql> explain select * from tb_seller where name='阿里巴巴' and address='北京市';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |    25.00 | Using index condition |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller where name='阿里巴巴' and status='1' and address='北京市';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref               | rows | filtered | Extra |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 813     | const,const,const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

## 范围查询要放在where条件的最后

```sql
mysql> explain select * from tb_seller where name='阿里巴巴' and status>'1' and address='北京';
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_seller | NULL       | range | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 410     | NULL |    1 |    25.00 | Using index condition |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

## 不要在索引列上进行运算

```sql
mysql> explain select * from tb_seller where substring(name,0,4)='阿里巴巴' ;
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select substring(name,0,2) from tb_seller where name='阿里巴巴' ;
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |   100.00 | Using index |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## 字符串不加单引号造成索引失效

 

```sql
mysql> explain select substring(name,0,2) from tb_seller where name='阿里巴巴' and status=1 and address='北京';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+--------------------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |    25.00 | Using where; Using index |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+--------------------------+
1 row in set, 2 warnings (0.00 sec)
```

没有对字符串加单引号，MySQL的查询优化器，会自动的进行类型转换，造成索引失效。

## 尽量使用覆盖索引，避免select *

 

```sql
mysql> explain select *  from tb_seller where name='阿里巴巴' and status=1 and address='北京';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |    25.00 | Using index condition |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
1 row in set, 2 warnings (0.00 sec)
mysql> explain select name,status,address  from tb_seller where name='阿里巴巴' and status=1 and address='北京';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+--------------------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |    25.00 | Using where; Using index |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+--------------------------+
1 row in set, 2 warnings (0.00 sec)
```

- using index ：使用覆盖索引的时候就会出现
- using where：在查找使用索引的情况下，需要回表去查询所需的数据 
- using index condition：查找使用了索引，但是需要回表查询数据 
- using index ; using where：查找使用了索引，但是需要的数据都在索引列中能找到，所以不需要回表 查询数据

-  以%开头的Like模糊查询，索引失效

如果仅仅是尾部模糊匹配，索引不会失效。如果是头部模糊匹配，索引失效。

 

```sql
mysql>  explain select *  from tb_seller where name like '阿里%';
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_seller | NULL       | range | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
mysql>  explain select *  from tb_seller where name like '%巴巴';
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |    25.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

如何解决这个问题呢，使用覆盖索引

 

```sql
mysql>  explain select name,status,address  from tb_seller where name like '阿里%';
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | tb_seller | NULL       | range | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | NULL |    1 |   100.00 | Using where; Using index |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
mysql>  explain select name,status,address  from tb_seller where name like '%巴巴';
+----+-------------+-----------+------------+-------+---------------+--------------------------+---------+------+------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key                      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+--------------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | tb_seller | NULL       | index | NULL          | idx_seller_name_sta_addr | 813     | NULL |    4 |    25.00 | Using where; Using index |
+----+-------------+-----------+------------+-------+---------------+--------------------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

## 用or分割开的条件

如果or前的条件中的列有索引，而后面的列中没有索引，那么涉及的索引都不会被用到。

 

```sql
mysql>  explain select *  from tb_seller where name ='阿里巴巴' or status='1' and createTime='2088-01-01 12:00:00';
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys            | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | idx_seller_name_sta_addr | NULL | NULL    | NULL |    4 |    29.69 | Using where |
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql>  explain select *  from tb_seller where name ='阿里巴巴' and  status='1' and createTime='2088-01-01 12:00:00';
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref         | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 410     | const,const |    1 |    25.00 | Using where |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## 如果MySQL评估使用索引比全表更慢，则不使用索引

 

```sql
mysql> create index idex_addr on tb_seller(address);
Query OK, 0 rows affected (0.47 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> explain select * from tb_seller where address='北京市';
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | idex_addr     | NULL | NULL    | NULL |    4 |   100.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller where address='杭州市';
+----+-------------+-----------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idex_addr     | idex_addr | 403     | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
```

为什么会出现这种情况呢？因为表中的数据大部分是北京市，mysql认为全表扫描更快。

## is NULL ， is NOT NULL 有时索引失效。

 

```sql
mysql> explain select * from tb_seller where  name is null;
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table     | partitions | type | possible_keys            | key                      | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_seller | NULL       | ref  | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | const |    1 |   100.00 | Using index condition |
+----+-------------+-----------+------------+------+--------------------------+--------------------------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller where  name is not null;
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys            | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | idx_seller_name_sta_addr | NULL | NULL    | NULL |    4 |   100.00 | Using where |
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

我们看到此时is null是走索引的，is not null是全表扫描。这是因为name字段绝大部分是有值的，现在我们修改该列，使其大部分为空，再来看看结果：

 

```sql
mysql> update tb_seller set name=null ;
Query OK, 4 rows affected (0.01 sec)
Rows matched: 4  Changed: 4  Warnings: 0
mysql> explain select * from tb_seller where  name is not null;
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_seller | NULL       | range | idx_seller_name_sta_addr | idx_seller_name_sta_addr | 403     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+--------------------------+--------------------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.01 sec)
mysql> explain select * from tb_seller where  name is  null;
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys            | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | idx_seller_name_sta_addr | NULL | NULL    | NULL |    4 |   100.00 | Using where |
+----+-------------+-----------+------------+------+--------------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

##  in 走索引， not in 索引失效

 

```sql
mysql> explain select * from tb_seller where  sellerid in ('baidu');
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_seller | NULL       | const | PRIMARY       | PRIMARY | 402     | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller where  sellerid not in ('baidu');
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | range | PRIMARY       | PRIMARY | 402     | NULL |    3 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

此时可以看到 in 和not in 都使用了索引，但是如果我们增加一个表中并不存在的值，就会变得不同，如下：

 

```sql
mysql> explain select * from tb_seller where  sellerid not in ('baidu','huawei','hh');
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |    4 |   100.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from tb_seller where  sellerid  in ('baidu','huawei','hh');
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_seller | NULL       | range | PRIMARY       | PRIMARY | 402     | NULL |    3 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

