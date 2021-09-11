---
title: MySQL运维
tags: 
 - mysql
categories:
 - 数据库
---

# 监控

MySQL 客户端连接成功后，通过 `show [session|global] status` 命令可以提供服务器状态信息。`show[session|global] status` 可以根据需要加上参数“session”或者“global”来显示 session 级（当前连接）的计结果和global 级（自数据库上次启动至今）的统计结果。如果不写，默认使用参数是“session”。

## 1. 查看索引使用情况

```sql
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 2     |
| Handler_read_key      | 3     |
| Handler_read_last     | 0     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 25    |
+-----------------------+-------+
7 rows in set (0.00 sec)
mysql> show global status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 200   |
| Handler_read_key      | 5358  |
| Handler_read_last     | 0     |
| Handler_read_next     | 582   |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 315   |
| Handler_read_rnd_next | 86247 |
+-----------------------+-------+
7 rows in set (0.00 sec)
```

- Handler_read_first：读取索引中第一个条目的次数。 如果这个值很高，则表明服务器正在进行大量的全索引扫描； 例如，SELECT col1 FROM foo，假设 col1 已编入索引。

```sql
mysql> flush status;
Query OK, 0 rows affected (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 0     |
| Handler_read_key      | 0     |
| Handler_read_last     | 0     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
-- address
mysql> select address from tb_seller;
+-----------+
| address   |
+-----------+
| 北京市    |
| 北京市    |
| 北京市    |
| 杭州市    |
+-----------+
4 rows in set (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 1     |
| Handler_read_key      | 1     |
| Handler_read_last     | 0     |
| Handler_read_next     | 4     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
```

- Handler_read_key：基于键读取行的请求数。 如果此值很高，则表明您的表已为查询正确编制了索引。

```sql
mysql> flush status;
Query OK, 0 rows affected (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 0     |
| Handler_read_key      | 0     |
| Handler_read_last     | 0     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
mysql> select * from tb_seller where address='北京';
Empty set (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 0     |
| Handler_read_key      | 1     |
| Handler_read_last     | 0     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
```

- Handler_read_last ：读取索引中最后一个键的请求数。 使用 ORDER BY，服务器将发出一个 first-key 请求，然后是几个 next-key 请求，而使用 ORDER BY DESC，服务器将发出一个 last-key 请求，然后是几个previous-key请求。 这个变量是在 MySQL 5.6.1 中添加的。

```sql
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 0     |
| Handler_read_key      | 0     |
| Handler_read_last     | 0     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
mysql> select * from tb_seller order by sellerid desc;
+----------+------+---------------+----------------------------------+--------+-----------+---------------------+
| sellerid | name | nickname      | password                         | status | address   | createtime          |
+----------+------+---------------+----------------------------------+--------+-----------+---------------------+
| itcast   | NULL | 传智播 客     | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市    | 2088-01-01 12:00:00 |
| huawei   | NULL | 华为小 店     | e10adc3949ba59abbe56e057f20f883e | 0      | 北京市    | 2088-01-01 12:00:00 |
| baidu    | NULL | 百度小 店     | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市    | 2088-01-01 12:00:00 |
| alibaba  | NULL | 阿里小 店     | e10adc3949ba59abbe56e057f20f883e | 1      | 杭州市    | 2088-01-01 12:00:00 |
+----------+------+---------------+----------------------------------+--------+-----------+---------------------+
4 rows in set (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 0     |
| Handler_read_key      | 1     |
| Handler_read_last     | 1     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 4     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
```

- Handler_read_next ：按键顺序读取下一行的请求数。 如果您正在查询具有范围约束的索引列或者您正在执行索引扫描，则该值会增加。参考Handler_read_key示例
- Handler_read_prev：按键顺序读取前一行的请求数。 这种读取方式主要用于优化ORDER BY ... DESC。参考Handler_read_last示例

 

- Handler_read_rnd ：根据固定位置读一行的请求数。如果你正执行大量查询并需要对结果进行排序该值较高。 你可能使用了大量需要MySQL扫描整个表的查询或你的连接没有正确使用键。这个值较高，意味着运行效率低，应 该建立索引来补救。 

- Handler_read_rnd_next：在数据文件中读下一行的请求数。如果你正进行大量的表扫描，该值较高。通常说 明你的表索引不正确或写入的查询没有利用索引。

 

```sql
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 1     |
| Handler_read_key      | 1     |
| Handler_read_last     | 0     |
| Handler_read_next     | 4     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 0     |
+-----------------------+-------+
7 rows in set (0.00 sec)
mysql> select * from tb_seller where password='111';
Empty set (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 2     |
| Handler_read_key      | 2     |
| Handler_read_last     | 0     |
| Handler_read_next     | 4     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 5     |
+-----------------------+-------+
7 rows in set (0.00 sec)
mysql> select * from tb_seller order by password;
+----------+------+---------------+----------------------------------+--------+-----------+---------------------+
| sellerid | name | nickname      | password                         | status | address   | createtime          |
+----------+------+---------------+----------------------------------+--------+-----------+---------------------+
| baidu    | NULL | 百度小 店     | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市    | 2088-01-01 12:00:00 |
| huawei   | NULL | 华为小 店     | e10adc3949ba59abbe56e057f20f883e | 0      | 北京市    | 2088-01-01 12:00:00 |
| itcast   | NULL | 传智播 客     | e10adc3949ba59abbe56e057f20f883e | 1      | 北京市    | 2088-01-01 12:00:00 |
| alibaba  | NULL | 阿里小 店     | e10adc3949ba59abbe56e057f20f883e | 1      | 杭州市    | 2088-01-01 12:00:00 |
+----------+------+---------------+----------------------------------+--------+-----------+---------------------+
4 rows in set (0.00 sec)
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 3     |
| Handler_read_key      | 7     |
| Handler_read_last     | 0     |
| Handler_read_next     | 4     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 4     |
| Handler_read_rnd_next | 10    |
+-----------------------+-------+
7 rows in set (0.00 sec)
```

总结： 上面的指标只是笼统的告诉我们数据库的索引使用情况，我们并不能很精确的得出是否应该对数据库进行优化。

## 2.统计增删改查sql的次数

 

```sql
mysql> show status like 'Com_______';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Com_binlog    | 0     |
| Com_commit    | 0     |
| Com_delete    | 0     |
| Com_insert    | 0     |
| Com_repair    | 0     |
| Com_revoke    | 0     |
| Com_select    | 30    |
| Com_signal    | 0     |
| Com_update    | 0     |
| Com_xa_end    | 0     |
+---------------+-------+
10 rows in set (0.00 sec)
```

| 参数       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| Com_select | 执行 select 操作的次数，一次查询只累加 1。                   |
| Com_insert | 执行 INSERT 操作的次数，对于批量插入的 INSERT 操作，只累加一次。 |
| Com_update | 执行 UPDATE 操作的次数。                                     |
| Com_delete | 执行 DELETE 操作的次数。                                     |
| Com_commit | 执行commit的次数                                             |
|            |                                                              |
|            |                                                              |

## 查看mysql数据行的统计

 

```sql
mysql> show status like 'Innodb_rows_%';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Innodb_rows_deleted  | 0     |
| Innodb_rows_inserted | 664   |
| Innodb_rows_read     | 852   |
| Innodb_rows_updated  | 17    |
+----------------------+-------+
4 rows in set (0.00 sec)
```

- Innodb_rows_read  select 查询返回的行数。
- Innodb_rows_inserted  执行 INSERT 操作插入的行数。
- Innodb_rows_updated  执行 UPDATE 操作更新的行数。
- Innodb_rows_deleted  执行 DELETE 操作删除的行数。

## show profile分析SQL

Mysql从5.0.37版本开始增加了对 show profiles 和 show profile 语句的支持。show profiles 能够在做SQL优化时帮助我们了解时间都耗费到哪里去了。

通过 have_profiling 参数，能够看到当前MySQL是否支持profile：

 

```sql
mysql> select @@have_profiling;
+------------------+
| @@have_profiling |
+------------------+
| YES              |
+------------------+
1 row in set, 1 warning (0.00 sec)
```

默认profiling是关闭的，可以通过set语句在Session级别开启profiling：

 

```sql
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set, 1 warning (0.00 sec)
set profiling=1; //开启profiling 开关；
```

通过profile，我们能够更清楚地了解SQL执行的过程。

首先，我们可以执行一系列的操作，如下图所示：

 

```sql
select * from tn_seller;
select count(*) from emp;
```

执行完上述命令之后，再执行show profiles 指令， 来查看SQL语句执行的耗时：

 

```sql
mysql> show profiles;
+----------+------------+--------------------------+
| Query_ID | Duration   | Query                    |
+----------+------------+--------------------------+
|        1 | 0.00009750 | select @@profiling       |
|        2 | 0.00012350 | select * from tn_seller  |
|        3 | 0.00019625 | select * from tb_seller  |
|        4 | 0.00015625 | select count(*) from emp |
+----------+------------+--------------------------+
4 rows in set, 1 warning (0.00 sec)
```

通过show profile for query query_id 语句可以查看到该SQL执行过程中每个线程的状态和消耗的时间：

 

```sql
mysql> show profile for query 3;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000038 |
| checking permissions | 0.000006 |
| Opening tables       | 0.000012 |
| init                 | 0.000013 |
| System lock          | 0.000006 |
| optimizing           | 0.000004 |
| statistics           | 0.000009 |
| preparing            | 0.000007 |
| executing            | 0.000002 |
| Sending data         | 0.000068 |
| end                  | 0.000004 |
| query end            | 0.000005 |
| closing tables       | 0.000005 |
| freeing items        | 0.000008 |
| cleaning up          | 0.000011 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)
```

 Sending data 状态表示MySQL线程开始访问数据行并把结果返回给客户端，而不仅仅是返回给客户端。由于在Sending data状态下，MySQL线程往往需要做大量的磁盘读取操作，所以经常是整各查询中耗时最长的状态。

在获取到最消耗时间的线程状态后，MySQL支持进一步选择all、cpu、block io 、context switch、page faults等明细类型类查看MySQL在使用什么资源上耗费了过高的时间。例如，选择查看CPU的耗费时间 ：

 

```sql
mysql> show profile cpu for query 3;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| starting             | 0.000038 | 0.000015 |   0.000017 |
| checking permissions | 0.000006 | 0.000003 |   0.000003 |
| Opening tables       | 0.000012 | 0.000005 |   0.000006 |
| init                 | 0.000013 | 0.000006 |   0.000006 |
| System lock          | 0.000006 | 0.000002 |   0.000003 |
| optimizing           | 0.000004 | 0.000002 |   0.000002 |
| statistics           | 0.000009 | 0.000004 |   0.000004 |
| preparing            | 0.000007 | 0.000003 |   0.000003 |
| executing            | 0.000002 | 0.000001 |   0.000002 |
| Sending data         | 0.000068 | 0.000030 |   0.000033 |
| end                  | 0.000004 | 0.000001 |   0.000002 |
| query end            | 0.000005 | 0.000002 |   0.000002 |
| closing tables       | 0.000005 | 0.000002 |   0.000003 |
| freeing items        | 0.000008 | 0.000004 |   0.000003 |
| cleaning up          | 0.000011 | 0.000004 |   0.000005 |
+----------------------+----------+----------+------------+
15 rows in set, 1 warning (0.00 sec)
```

## trace分析优化器执行计划

MySQL5.6提供了对SQL的跟踪trace, 通过trace文件能够进一步了解为什么优化器选择A计划, 而不是选择B计划。

打开trace ， 设置格式为 JSON，并设置trace最大能够使用的内存大小，避免解析过程中因为默认内存过小而不能够完整展示。

 

```sql
SET optimizer_trace="enabled=on",end_markers_in_json=on;
set optimizer_trace_max_mem_size=1000000;
```

 

```sql
mysql> select * from emp;
-- 查看执行计划
mysql> select * from information_schema.optimizer_trace\G;
*************************** 1. row ***************************
                            QUERY: select * from emp
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `emp`.`id` AS `id`,`emp`.`name` AS `name`,`emp`.`age` AS `age`,`emp`.`salary` AS `sal                                                                                                                               ary` from `emp`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "table_dependencies": [
              {
                "table": "`emp`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "rows_estimation": [
              {
                "table": "`emp`",
                "table_scan": {
                  "rows": 12,
                  "cost": 1
                } /* table_scan */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`emp`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 12,
                      "access_type": "scan",
                      "resulting_rows": 12,
                      "cost": 3.4,
                      "chosen": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 12,
                "cost_for_plan": 3.4,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": null,
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`emp`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "refine_plan": [
              {
                "table": "`emp`"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
          INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.00 sec)
```

# 运维

