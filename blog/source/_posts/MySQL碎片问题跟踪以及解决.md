---
title: MySQL碎片问题跟踪以及解决
date: 2020-08-02 09:53:52
tags:
- mysql
- 碎片
categories:
- tech
description: 处理mysql碎片问题
---

### 什么是 mysql 碎片？
我们都知道 mysql 的默认每页大小是16k，而Linux系统的默认大小是4k，可以使用如下的命令得到页大小：

```shell script
getconf PAGE_SIZE
```

输出大小如下：
```shell script
4096
```

证明系统的页大小为4kb，但是我们知道mysql的页大小为16kb，这个可以使用如下的命令得到这个值：

```sql mysql
SHOW VARIABLES LIKE 'innodb_page_size';
```

```shell script
Variable_name    Value
innodb_page_size 16384
```

我们既然知道了两个页大小不一致，并且mysql为了避免一次性写入16kb的失败和回滚而采用的 `double buffer write` 技术
我们这里说的碎片就是因为存储数据的时候，数据大小的变化导致内存的分配而导致的碎片问题

### 碎片会有什么影响？
MySQL的碎片是由于在存在数据后，对数据进行修改、更新和删除操作，导致了数据页分裂而造成的数据碎片，
跟C语言里面malloc、free一样，总是频繁的操作会导致存在碎片问题致使引擎内存利用率下降，性能降低
比如你会发现：同样的一个查询之前好好地，为什么现在查询会变得很慢等等

+ 空间占用过大，但是数据大小没有增加多少
+ 查询、插入性能变差
+ 碎片存在导致了数据库的磁盘IO变成了随机IO，增加了磁盘的IO负担，性能变差

### 怎么判断mysql存在碎片？
通过如下SQL语句进行判断：

+ 
```sql mysql
SHOW TABLE STATUS LIKE 'table_name'
```
+ 
```sql mysql
SELECT * FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'data_base' AND TABLE_NAME = 'table_name'
```
如果看到DATA_FREE列大于0，则表示存在碎片，我们也要注意，不是说这列大于0，就需要进行清理
而是当其处于一个非理想状态下的较大值时候，我们才进行优化，释放空间，因为清理碎片会对表进行锁库，影响线上的功能，所以我们应该综合考虑清楚在处理

### 碎片怎么清理？
+ InnoDB
```sql mysql
ALTER TABLE table_name ENGINE=InnoDB;
```

+ MyISAM
```sql mysql
ALTER TABLE table_name ENGINE=InnoDB;
```








































