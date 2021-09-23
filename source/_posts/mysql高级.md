---
title: MySQL高级
tags: 
 - mysql
categories:
 - 数据库
---

#  安装

- 安装过程中，必须初始化data目录以及系统数据库。rpm是自动完成的，win和linux的压缩包则需要开发者自己操作。
- 安装过程中会生成临时密码，密码存储在 ```/var/log/mysqld.log``。`

## rpm安装

 

```shell
# 1.下载yum仓库，并安装仓库
shell> wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
shell> yum -y install mysql57-community-release-el7-10.noarch.rpm
# 2.安装mysql
shell> yum -y install mysql-community-server
# 3.启动
shell> systemctl start mysqld.service
```

## linux压缩包安装

 

```shell
# 创建用户
shell> groupadd mysql
shell> useradd -r -g mysql -s /bin/false mysql
# 解压文件
shell> cd /usr/local
shell> tar zxvf /path/to/mysql-VERSION-OS.tar.gz
shell> ln -s full-path-to-mysql-VERSION-OS mysql
#创建数据目录
shell> cd mysql
shell> mkdir mysql-files
shell> chown mysql:mysql mysql-files
shell> chmod 750 mysql-files
# 初始化数据目录和系统数据库
shell> bin/mysqld --initialize --user=mysql
# 支持安全连接（可选）
shell> bin/mysql_ssl_rsa_setup
# 启动
shell> bin/mysqld_safe --user=mysql &
# shell> bin/mysqld --user=mysql &
# 配置自启动
shell> cp support-files/mysql.server /etc/init.d/mysql.server
```

## window安装

```bash
mysqld.exe --initialize-insecure --console
```

- --initialize-insecure: 不生成初始化密码
- --initialize：生成初始化密码

## 安装后

**修改密码**

```sql
alter user 'root'@'localhost' identified by '1234';
```

# 约束

## 外键约束

1. 创建表的时候添加，格式如下:

```sql
[CONSTRAINT 约束名称] FOREIGN KEY(外键字段) REFERENCES 主表名称(主表字段) 
    [ON DELETE {RESTRICT | CASCADE | SET NULL | NO ACTION}] 
    [ON UPDATE {RESTRICT | CASCADE | SET NULL | NO ACTION}]
```

2. 表创建之后添加：

```sql
ALTER TABLE employee ADD FOREIGN KEY(dept_id) REFERENCES department(id);
```

ON DELETE后面的四个参数：代表的是当删除主表的记录时，所做的约定。

- RESTRICT：主记录存在依赖则不允许删除。  
- CASCADE（级联）：如果主表的记录删掉，则从表中相关联的记录都将被删掉。
- SET NULL：将外键设置为空。
- NO ACTION：等同于RESTRICT

### 示例

```sql
-- 创建班级表
CREATE TABLE z_class (
    id INT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR (20) NOT NULL COMMENT '名称'
) ENGINE = INNODB DEFAULT charset utf8;
-- 创建学生表
CREATE TABLE z_student (
    id INT PRIMARY KEY auto_increment,
    NAME VARCHAR (20) NOT NULL COMMENT '名称',
    c_id INT,
    CONSTRAINT forign_cId_cls_id FOREIGN KEY (c_id) REFERENCES z_class (id) ON UPDATE RESTRICT
) ENGINE = INNODB DEFAULT charset utf8;
-- 测试插入
insert INTO z_class(name) value('一班');
insert INTO z_student(name,c_id) value('张三',1);
update z_student set name='王五' where id=1;
update z_class set name='二班' where id=1;
update z_class set id=2 where id=1;
-- [Err] 1451 - Cannot delete or update a parent row: 
--- a foreign key constraint fails (`test`.`z_student`, CONSTRAINT `forign_cId_cls_id` FOREIGN KEY (`c_id`) REFERENCES `z_class` (`id`))
-- 修改更新约束为CASCADE 之后，此时没有抛出异常，并且从表的外键更改，保持和主表的主键一致。
update z_class set id=2 where id=1;
-- 修改更新约束为 SET NULL ,此时没有抛出异常，主键发生更改，从表的外键置空。
update z_class set id=1 where id=2;
-- 修改更新约束为 NO ACTION,此时抛出异常
update z_class set id=2 where id=1;
```

# 视图

视图(View )是一种虚拟存在的表。视图并不在数据库中实际存在,行和列数据来自定义视图的查询中使用的表,并且是在使用视图时动态生成的。通俗的讲,视图就是一条SELECT语句执行后返回的结果集。 视图相对于普通的表的优势主要包括以下几项。

* 简单:使用视图的用户完全不需要关心后面对应的表的结构、关联条件和筛选条件,对用户来说已经是过滤好的复合条件的结果集。
* 安全:使用视图的用户只能访问他们被允许查询的结果集,对表的权限管理并不能限制到某个行某个列,但是通过视图就可以简单的实现。
* 数据独立:一旦视图的结构确定了,可以屏蔽表结构变化对用户的影响,源表增加列对视图没有影响;源表修改列名,则可以通过修改视图来解决,不会造成对访问者的影响。

>  更新视图中的数据时，同时会更新表中的数据，不建议做更新操作

## 语法

**创建视图**

```sql
CREATE [OR REPLACE] [ALGORITHM = {UNDEFINED| MERGE| TEMPTABLE}]
VIEW view_name [(column_list)]
AS select_statement
[WITH [CASCADED | LOCAL] CHECK OPTION]
```

**修改视图**

```sql
ALTER [ALGORITHM = {UNDEFINED| MERGE |TEMPTABLE]]
VIEW view_name [(column_list)]
AS select_statement
[WITH [CASCADED| LOCAL] CHECK OPTION]
```

> WITH [CASCADED | LOCAL] CHECK OPTION决定了是否允许更新数据使记录不再满足视图的条件。
> LOCAL :只要 满足本规图的条件就可以更新。
> CASCADED :必须满足所有针对该视图的所有视图的条件才可以更新.

**查看视图**

```sql
show create view view_name;
```

**删除视图**

```sql
DROP VIEW [IF EXISTS] view_ name[，view_ name]...[RESTRICT | CASCADE]
```

# 存储过程和函数

存储过程和函数是事先经过编译并存储在数据库中的一段SQL语句的集合,调用存储过程和函数可以简化应用开发人员的很多工作,减少数据在数据库和应用服务器之间的传输,对于提高数据处理的效率是有好处的。
存储过程和函数的区别在于：函数必须有返回值,而存储过程没有。

## 创建存储过程

```sql
CREATE PROCEDURE procedure_name ( [proc_parameter[,...]] )
begin 
-- SQL语句
end ；
```

示例：

```sql
delimiter $

create procedure pro_test()
begin
select 'Hello Mysq1' ；
end$

delimiter ；
```

> delimiter关键字用来声明SQL语句的分隔符,用来告诉MySQL解释器该段命令是否已经结束了. mysq|是否可以执行了。默认情况下, delimiter是分号。在命令行客户端中, 如果有一行命令以分号结束 ,那么回车后, mysq将会执行该命令。

## 调用存储过程

```sql
call procedure_name( [proc_parameter[,...]] )
```

## 查看存储过程

```sql
-- 查询db_ name数据库中的所有的存储过程
select name from mysq1.proc where db='db_ name';
-- 查询存储过程的状态信息
show procedure status ;
-- 查询某个存储过程的定义
show create procedure test.pro_test1  ;
```

## 删除存储过程

```sql
drop procedure procedure_name
```

## 语法

存储过程是可以编程的，这意味着可以使用变量、表达式、控制结构等来完成比较复杂的功能。

### 变量

使用DECLARE声明一个变量，该变量scope只能在BEGIN ... END 块中：

```sql
declare var_name[,...] type [default value]
```

示例：

```sql
CREATE PROCEDURE test1 ()
BEGIN
    DECLARE num INT DEFAULT 10; -- 声明变量
    SELECT num;
END
```

```sql
create  PROCEDURE test2()
BEGIN 
    DECLARE num INT DEFAULT 10; 
    SELECT 11 INTO num ; -- 将查询到的数据填充到变量中
    SELECT num;
END

```

```sql
create  PROCEDURE test2()
BEGIN 
    DECLARE num INT DEFAULT 10; 
    set num=1000 ; -- 设置变量
    SELECT num;
END
```

### if条件判断

```sql
CREATE PROCEDURE test3 ()
BEGIN
    DECLARE num INT DEFAULT 175;
    DECLARE content VARCHAR (20);

    IF num > 180 THEN
        SET content = 'high';
    ELSEIF num > 170 THEN
        SET content = 'middle';
    ELSE
        SET content = 'low';
    END IF;

    SELECT content;
END
```

### 传递参数

* IN: 表示输入参数
* OUT：表示输出参数
* `INOUT`: 即可用作输入参数，又可以用作输出参数

输入参数：

```sql
CREATE PROCEDURE test4 (in num INT)
BEGIN
    DECLARE content VARCHAR (20);

    IF num > 180 THEN
        SET content = 'high';
    ELSEIF num > 170 THEN
        SET content = 'middle';
    ELSE
        SET content = 'low';
    END IF;

    SELECT content;
END
```

输出参数：

```sql
CREATE PROCEDURE test5 (in num INT,out content VARCHAR(20))
BEGIN
    IF num > 180 THEN
        SET content = 'high';
    ELSEIF num > 170 THEN
        SET content = 'middle';
    ELSE
        SET content = 'low';
    END IF;
END

CALL test5(180,@content); -- 调用 @表示会话变量 @@定义系统变量
SELECT @content
```

> 使用了输出参数之后，我们就不需要使用select 语句来输出结果了。

### case结构

```sql
-- 方式一
CASE case_value
WHEN when_value THEN statement
[WHEN when_value THEN statement] ...
[ELSE statement]
END CASE ;
-- 方式二
CASE
WHEN condition THEN statement
[WHEN condition THEN statement] ...
[ELSE statement]
END CASE;
```

示例：

```sql
create PROCEDURE test6(in num INT)
BEGIN
	declare content varchar(20) ;
	CASE num
		when 1 then SET content='one';
		when 2 then SET content='two';
		ELSE SET content='hhh';
	END CASE;
	SELECT content;
END
```

### while循环

```sql
create PROCEDURE test7(in num int)
BEGIN
	DECLARE total INT DEFAULT 0;
	WHILE num <10 DO 
	 SET total=total+num;
	 set num=num+1;
	END WHILE;
	SELECT total;
END
```

### repeat结构

```sql
create PROCEDURE test8()
BEGIN
	DECLARE num int DEFAULT 1;
  REPEAT 
		set num=num*2;
		UNTIL num>10 
  END REPEAT;
	SELECT num;
END
```

### loop和leave语句

loop是个死循环，需要leave退出：

```sql
create PROCEDURE test9()
BEGIN
 DECLARE num int DEFAULT 1;
 tag1:LOOP
		set num=num*2;
    IF num>10 THEN 
			LEAVE tag1 ;
		END IF;
 END LOOP tag1;

  SELECT num;
END
```

## 游标

游标是用来存储查询结果集的数据类型,在存储过程和函数中可以使用游标对结果集进行循环的处理。游标的使用包括游标的声明、OPEN、FETCH和CLOSE ,其语法分别如下：

```sql
-- 声明游标
DECLARE cursor_name CURSOR FOR select_statement;
-- 打开游标
OPEN cursor_name
-- 获取游标
FETCH cursor_name INTO var_name[,var_name]...
-- 关闭游标
CLOSE  cursor_name
```

**简单使用**

```sql
create procedure pro_test11 ()
begin
    declare e_id int(11) ;
    declare e_name varchar (50) ;
    declare e_age int(11) ;
    declare e_salary int(11) ;
    -- 声明游标
    declare emp_result cursor for select * from emp;
    -- 打开游标
    open emp_result;
    -- 获取游标
    fetch emp_result into e_id,e_name,e_age,e_salary;
    
    select concat('id=',e_ id,', name=',e_name, '，age=', e_age, '，薪资为: ',e_salary);
    -- 关闭游标
    close emp_result;
end
```

在上面的代码中，我们只获取了一行数据，如果做到类似java中的循环遍历呢，一种方法是声明count语句，然后循环。另外一种如下：

```sql
begin
    declare e_id int(11) ;
    declare e_name varchar (50) ;
    declare e_age int(11) ;
    declare e_salary int(11) ;
    declare has data int default 1;
    
    declare emp_result cursor for select * from emp;
    
    DECLARE EXIT HANDLER FOR NOT FOUND set has_data= 0 ;
    open emp_result;
    repeat
        fetch emp_result into e_id,e_name,e_age,e_salary;
        select concat('id=',e_id,'，name=' ,e_name, '，age=', e_age, '，薪资为: ' ,e_salary);
        until has_data = 0
    end repeat;
    close emp_result;
end

```

## 函数

```sql
CREATE FUNCTION function_name([param type ... ])
RETURNS type
BEGIN
...
END ;

```

示例：

```sql
create function fun1 (countryId int)
RETURNS int
begin
    declare cnum int ;
    select count(*) into cnum from city where country_id = countryId;
    return cnum ;
end

```

# 触发器

在insert/update/delete之前或之后,触发并执行触发器中定义的SQL语句集合。触发器的这种特性可以协助应用在数据库端确保数据的完整性,日志记录,数据校验等操作。

使用别名OLD和NEW来引用触发器中发生变化的记录内容,这与其他的数据库是相似的。现在触发器还只支持行级触发,不支持语句级触发。

| 触发器类型 | NEW和OLD                             |
| ---------- | ------------------------------------ |
| insert     | new 表示新增的数据                   |
| update     | old表示修改之前的，new表示修改之后的 |
| delete     | old表示删除的数据                    |

## 创建触发器

```sql
create trigger trigger_name
before/after insert/update/delete
on tb1_name
[ for each row ]
trigger_stmt ;

```

示例：

```sql
mysql> CREATE TRIGGER double_salary
    -> AFTER INSERT ON tb_emp6
    -> FOR EACH ROW
    -> INSERT INTO tb_emp7
    -> VALUES (NEW.id,NEW.name,deptId,2*NEW.salary);
```

## 删除触发器

```sql
drop trigger trigger_name
```

## 查看触发器

```sql
show trigger
```

# 事件

## 概述

自MySQL5.1.6起，增加了一个非常有特色的功能 - 事件调度器（Event Scheduler），可以用做定时执行某些特定任务（例如：删除记录、数据统计报告、数据备份等等），来取代原先只能由操作系统的计划任务来执行的工作。

> 值得一提的是MySQL的事件调度器可以精确到每秒钟执行一个任务，而操作系统的计划任务（如：Linux的cron）只能精确到每分钟执行一次。对于一些对数据实时性要求比较高的应用（例如：股票、赔率、比分等）就非常适合。

事件有时也可以称为临时触发器（temporal triggers），因为事件调度器是基于特定时间周期触发来执行某些任务，而触发器（Triggers）是基于某个表所产生的事件触发的，区别也就在这里。

## 开启事件

使用“事件”功能之前必须确保event_scheduler已开启：

```shell
mysql> SELECT @@event_scheduler; #方法一
+-------------------+
| @@event_scheduler |
+-------------------+
| OFF               |
+-------------------+
1 row in set (0.00 sec)

mysql> SHOW VARIABLES LIKE 'event%'; #方法一
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| event_scheduler | OFF   |
+-----------------+-------+
1 row in set (0.00 sec)
```

开启命令：

```sql
-- 开启功能命令：
SET GLOBAL event_scheduler = 1;
SET GLOBAL event_scheduler = ON;
-- 关闭功能命令：
SET GLOBAL event_scheduler = 0;
SET GLOBAL event_scheduler = OFF;
```

通过命令开启当数据库重启后会自动关闭；永久生效需要将`event_scheduler=1`写到my.cnf配置文件中。

## sql语法

### 创建事件

```sql
CREATE EVENT [IFNOT EXISTS] event_name
    　　 ON SCHEDULE schedule(调度时间设置)
    　　 [ON COMPLETION [NOT] PRESERVE]
    　　 [ENABLE | DISABLE | DISABLE ON SLAVE]
    　　 [COMMENT 'comment']
    　　 DO sql_statement;
```

* **ON COMPLETION [NOT] PRESERVE**: 配置事件执行完一次后的处理方式；当为on completion preserve 的时候,当event到期了,event会被disable,但是该event还是会存在; 当为on completion not preserve的时候,当event到期的时候,该event会被自动删除掉.
* **ENABLE、DISABLE、DISABLE ON SLAVE**:ENABLE表示该事件是开启的，也就是调度器检查事件是否必选调用；DISABLE表示该事件是关闭的，也就是事件的声明存储到目录中，但是调度器不会检查它是否应该调用；DISABLE ON SLAVE表示事件在从机中是关闭的。如果不指定这三个选择中的任意一个，则在一个事件创建之后，它立即变为活动的。
* **DO event_body**: 用于指定事件启动时所要执行的代码。可以是任何有效的SQL语句、存储过程或者一个计划执行的事件。如果包含多条语句，可以使用BEGIN…END复合结构

`schedule` 调度时间配置语法：调度时间配置包括`AT` 和 `EVERY`两种

```sql
AT timestamp [+ INTERVAL interval] ...
  | EVERY interval
    [STARTS timestamp [+ INTERVAL interval] ...]
    [ENDS timestamp [+ INTERVAL interval] ...]


-- INTERVAL中包含的时间单位如下:
{YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE |
 WEEK | SECOND | YEAR_MONTH | DAY_HOUR | DAY_MINUTE |
 DAY_SECOND | HOUR_MINUTE | HOUR_SECOND | MINUTE_SECOND}
```

示例1（某个时间点插入一条数据到表中）：

```sql
-- 事件执行完成之后就被删除了
CREATE EVENT insert_to_indextest 
	ON SCHEDULE AT TIMESTAMP '2021-09-14 10:04:00' 
	DO
		INSERT INTO index_test (id,`NAME`) VALUES(2,'event2');
```

示例2（当前时间之后1分钟再执行）

```sql
CREATE EVENT insert_to_indextest3 ON SCHEDULE AT  CURRENT_TIMESTAMP + INTERVAL 1 MINUTE 
DO
	INSERT INTO index_test (id, `NAME`)
VALUES
	(4, 'event4');
```

> 注意使用CURRENT_TIMESTAMP时，没有TIMESTAMP关键字

示例3（每隔10秒插入一条数据）

```sql
CREATE EVENT test_every ON SCHEDULE EVERY 10 SECOND ON COMPLETION PRESERVE DO
	INSERT INTO test
VALUES
	('aa');
```

### 修改

修改语法几乎和创建语句一样：

```sql
ALTER EVENT event_name
    　　 [ON SCHEDULE schedule]
    　　 [rename TO new_NAME]
    　　 [ON COMPLETION [NOT] PRESERVE]
    　　 [COMMENT 'comment']
    　　 [ENABLE | DISABLE]
    　　 [DO sql_statement]
```

示例：

```sql
ALTER EVENT test_every ON SCHEDULE EVERY 10 SECOND DO
	INSERT INTO test
VALUES
	('bb');
-- 修改名称
ALTER EVENT test_every rename TO test_every_new ;
```

### 禁用事件

```sql
ALTER EVENT test_every_new DISABLE;
ALTER EVENT test_every_new ENABLE;
```

### 删除

```sql
DROP EVENT [IF EXISTS] event_name
```





# mysql体系

![img](./mysql%E9%AB%98%E7%BA%A7/src=http%253A%252F%252Fseo-1255598498.file.myqcloud.com%252Ffull%252F05b627442aef5676dee67b33e8627c8782564b7d.jpg&refer=http%253A%252F%252Fseo-1255598498.file.myqcloud.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg)

* 连接层
  最上层是一些客户端和链接服务,包含本地sock通信和大多数基于客户端/服务端工具实现的类似于TCP/IP的通信。主要完成一些类似于连接处理、授权认证相关的安全方案。在该层上引入了线程池的概念,为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

* 服务层
  第二层架构主要完成大多数的核心服务功能,如SQL接口,并完成缓存的查询, SQL的分析和优化,部分内置函数的执行。所有跨存储引擎的功能也在这一层实现，如过程、函数等。在该层,服务器会解析查询并创建相应的内部解析树,并对其完成相应的优化如确定表的查询的顺序,是否利用索引等,最后生成相应的执行操作。如果是select语句,服务器还会查询内部的缓存,如果缓存空间足够大,这样在解决大量读操作的环境中能够很好的提升系统的性能。

* 引擎层
  存储引擎层，存储引擎真正的负责了`MySQL`中数据的存储和提取.服务器通过`API`和存储引擎进行通信。不同的存储引擎具有不同的功能.这样我们可以根据自己的需要,来选取合适的存储引擎。
* 存储层
  数据存储层，主要是将数据存储在文件系统之上,并完成与存储引擎的交互。和其他数据库相比, `MySQL`有点与众不同,它的架构可以在多种不同场景中应用并发挥良好作用。主要体现在存储引擎上,插件式的存储引擎架构,将查询处理和其他的系统任务以及数据的存储提取分离。这种架构可以根据业务的需求和实际需要选择合适的存储引擎。

## 存储引擎

存储引擎就是存储数据,建立索引,更新查询数据等等技术的实现方式。存储引擎是基于表的,而不是基于库的。所以存储引擎也可被称为表
类型。

Oracle , Sq|Server等数据库只有一种存储引擎。 `MySQL`提供了插件式的存储引擎架构。所以`MySQL`存在多种存储引擎,可以根据需要使用相应引擎,或者编写存储引擎。

`MySQL5.0`支持的存储引擎包含: `InnoDB` 、`MyISAM`、`BDB`、 MEMORY、 MERGE、 EXAMPLE、 `NDB Cluster`、ARCHIVE、 `CSV`
`BLACKHOLE`、FEDERATED等 ,其中`InnoDB`和`BDB`提供事务安全表,其他存储引擎是非事务安全表。

可以通过指定`show engines` ,来查询当前数据库支持的存储引擎:

![image-20210830203726044](./mysql%E9%AB%98%E7%BA%A7/image-20210830203726044.png)

创建新表时如果不指定存储引擎,那么系统就会使用默认的存储引擎, MySQL5.5之前的默认存储引擎是MyISAM , 5.5之后就改为了InnoDB。查看Mysq|数据库默认的存储引擎,指令:

```sql
SHOW VARIABLES like '%storage_engine%'
```

![image-20210830204112760](./mysql%E9%AB%98%E7%BA%A7/image-20210830204112760.png)

### **存储方式**	

InnoDB 存储表和索引有以下两种方式 ： 

*  使用共享表空间存储， 这种方式创建的表的表结构保存在.frm文件中， 数据和索引保存在 innodb_data_home_dir 和 innodb_data_file_path定义的表空间中，可以是多个文件。

* 使用多表空间存储， 这种方式创建的表的表结构仍然存在 .frm 文件中，但是每个表的数据和索引单独保存在 .ibd 中。

每个MyISAM在磁盘上存储成3个文件，其文件名都和表名相同，但拓展名分别是 ： 

* .frm (存储表定义)；

* .MYD(MYData , 存储数据)；

* .MYI(MYIndex , 存储索引)；

### MEMORY

​	Memory存储引擎将表的数据存放在内存中。每个MEMORY表实际对应一个磁盘文件，格式是.frm ，该文件中只存储表的结构，而其数据文件，都是存储在内存中，这样有利于数据的快速处理，提高整个表的效率。MEMORY 类型的表访问非常地快，因为他的数据是存放在内存中的，并且默认使用HASH索引 ， 但是服务一旦关闭，表中的数据就会丢失。

### MERGE

​	MERGE存储引擎是一组MyISAM表的组合，这些MyISAM表必须结构完全相同，MERGE表本身并没有存储数据，对MERGE类型的表可以进行查询、更新、删除操作，这些操作实际上是对内部的MyISAM表进行的。

​	对于MERGE类型表的插入操作，是通过INSERT_METHOD子句定义插入的表，可以有3个不同的值，使用FIRST 或 LAST 值使得插入操作被相应地作用在第一或者最后一个表上，不定义这个子句或者定义为NO，表示不能对这个MERGE表执行插入操作。

​	可以对MERGE表进行DROP操作，但是这个操作只是删除MERGE表的定义，对内部的表是没有任何影响的。

