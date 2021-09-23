# 历史表迁移

## 概述

当业务运行一段时间后，会出现有些表数据量很大，可能对系统性能产生不良的影响，常见的如订单表、登录log表等，这些数据很有时效性，比如我们一般很少去查上个月的订单，最多也就是报表统计会涉及到。

并且，表中的数据是不可逆的，只有插入操作没有删除或者修改操作，表示在过去一段时间内完成的事实业务数据。比如登录表，表示一段时间内用户的登录信息，登录一次就会在数据库中记录一条数据。

## 迁移策略

1. 第一次备份全表的数据导入到备份表中，也就是全备。
2. 以后每天只将增加的量自动迁移到备份表中，这里通过自增列的值相比较得到增量，如第一次全备导入备份表，这时自增列的最大值为max(idx)，通过这个值到从库中查询，只要大于这个值的所有记录形成增量。
3. 备份完成后，分批删除主库上的数据。

### 具体实施（同库迁移）

需求：同一数据库中，表A只保留最近一个月数据，其余数据迁移到表B中

1. 复制A表，重命名成B表

   ```sql
   create table B select * from A;
   ```

2. 创建事件

   ```sql
   create event month_A_back_B on schedule every 1 MONTH  ON COMPLETION PRESERVE DO
   begin
   -- 复制A表中一个月之前的数据到B表
   insert into B select * from A where time< SELECT DATE_ADD(CURDATE(),interval -1 MONTH);
   -- 删除A表中一个月之前的数据
   delete from A where time < SELECT DATE_ADD(CURDATE(),interval -1 month);
   -- 重建A表的索引，提升效率
   alter table A drop index k; 
   alter table A add index(k);
   end
   ```

   

# 索引重建

当你对InnoDB进行修改操作时，例如删除一些行，这些行只是被标记为“已删除”，而不是真的从索引中物理删除了，因而空间也没有真的被释放回收。InnoDB的Purge线程会异步的来清理这些没用的索引键和行，但是依然没有把这些释放出来的空间还给操作系统重新使用，因而会导致页面中存在很多空洞。

有些用户可能会使用 OPTIMIZE TABLE 或者 ALTER TABLE <table> ENGINE=InnoDB 来重建这些表，但是这样会导致表的拷贝，如果临时空间不足甚至不能进行一次 OPTIMIZE TABLE 操作。

并且如果你用的是共享表空间方式，OPTIMIZE TABLE 会导致你的共享表空间文件持续增大，因为整理的索引和数据都追加在数据文件的末尾。

> InnoDB类型的表是无法使用optimize table命令

重建索引：

```sql
alter table T drop index k;
alter table T add index(k);
```

重建主键索引

```sql
alter table T drop primary key;
alter table T add primary key(id);
```

或者：

```sql
alter table T engine=InnoDB;
```

可以释放空洞，这是由于在转换数据引擎（即使没有真正转换）的时候，会将表中的所有数据读取，再重新写入，这个过程中，会释放空洞（效率慢）