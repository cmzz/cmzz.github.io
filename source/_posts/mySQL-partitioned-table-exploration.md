---
title: MySQL 分区表探索
tags: 
- 技术
- MySQL
date: 2017-12-08 08:00:00
---

## 背景

如果需要定时清理一张普通大表里的历史数据。

可以使用一个或多个带 where 条件的 delete 语句去删除（where条件是时间）。 如果表数据量较大，这对数据库的造成了很大压力。即使我们把这些旧数据删除了，但是底层的数据文件并没有变小。

> 为什么没有变小？
> 当删除数据 时，MYSQL 并不会立即回收表空间。被已删除数据的占据的存储空间，以及索引位会空在那里，等待新的数据来弥补这个空缺。
> 强行回收： OPTIMIZE TABLE

面对这类问题，其实最有效的方法就是在使用分区表。分区表最大的优点就是可以非常高效的进行历史数据的清理。

## 关于分区表
分区表不是在存储引擎层完成的。这是 MySQL 支持的功能（5.1开始）

原理：
将表索引分解为多个更小、更可管理的部分。

从逻辑上讲，只有一个表或者索引，但是物理上这个表或者索引可能由数十个物理分区组成。

分区表最大的优点就是可以非常高效的进行历史数据的清理。

每个分区都是独立的对象，可以独自处理，也可以作为一个更大对象的一部分进行处理（如果分区表很大，亦可以将分区分配到不同的磁盘上去）。

在执行查询的时候，优化器会根据分区定义过滤哪些没有我们需要数据的分区，这样查询就无须全表扫描所有分区，只查找包含需要数据的分区即可。


## 检查分区功能是否启用

    mysql> SHOW PLUGINS \G;

    *************************** 43. row ***************************
       Name: partition
     Status: ACTIVE
       Type: STORAGE ENGINE
    Library: NULL
    License: GPL


## 分区类型
MySQL目前只支持 水平分区（水平分区就是将不同的行分配到不同的物理文件中）。

* 范围分区（RANGE）
  行数据基于一个给定的连续区间的值被放入分区。

* 列表分区（LIST）
  和 RANGE 分区类似，只不过面向的是离散的值。

* 哈希分区（HASH）
  根据用户自定义的表达式返回的值来区分放入那个分区。

* KEY分区
  根据 MySQL 数据库提供的哈希函数来进行分区。

* COLUMNS 分区
  可以对多个列的值进行分区。（MySQL 5.5+ 开始支持）。

## RANGE 分区

这是最常用的一种分区类型。最常见的是基于时间字段（基于分区的列最好是整型）来分区。
分区的列可以允许 null 值，如果分区的列值是 null，则会选择第一个分区。


    CREATE TABLE range_partition_test (
        id INT,
        pdate INT
    )
    PARTITION BY RANGE (pdate) (
        PARTITION p1 VALUES LESS THAN ( 201702 ),
        PARTITION p2 VALUES LESS THAN ( 201703 ),
        PARTITION p3 VALUES LESS THAN ( 201704 ),
        PARTITION p4 VALUES LESS THAN (MAXVALUE)
    );


`MAXVALUE` 是一个无穷大的值，所以p4 分区即为默认的分区。


在执行查询的时候，带上分区字段，这样可以使用分区剪裁功能。

    mysql> select * from range_partition_test;
    +------+--------+
    | id   | pdate  |
    +------+--------+
    |    1 | 201701 |
    |    2 | 201702 |
    |    3 | 201703 |
    |    4 | 201704 |
    |    5 | 201705 |
    +------+--------+


    mysql> explain partitions select * from range_partition_test where pdate between 201702 and 201703;
    +----+-------------+----------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
    | id | select_type | table                | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
    +----+-------------+----------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
    |  1 | SIMPLE      | range_partition_test | p2,p3      | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where |
    +----+-------------+----------------------+------------+------+---------------+------+---------+------+------+----------+-------------+


只查询了p2,p3分区。


## LIST 分区

LIST 分区和 RANGE 分区类似。
区别在于 LIST 是枚举值列表的集合，RANGE 是连续的区间值的集合。二者在语法方面非常的相似。
建议 LIST 分区列是非 null 列，否则插入 null 值如果枚举列表里面不存在 null 值会插入失败（和 RANGE 分区不一样）。

    CREATE TABLE list_partition_test (
        id INT,
        pdate INT
    )
    PARTITION BY LIST (pdate) (
        PARTITION p1 VALUES IN (1,3,5,7,9),
        PARTITION p2 VALUES IN (2,4,6,8,0)
    );


## Hash 分区

HASH 分区的目的是讲数据均匀的分不到预先定义的各个分区中。保证各个分区的记录数量大体上都是一致的。

在实际工作中经常遇到像会员表的这种表。并没有明显可以分区的特征字段。但表数据有非常庞大。这时候可以使用 HASH 分区。

基于给定的分区个数，将数据分配到不同的分区，HASH分区只能针对整数进行 HASH。


    CREATE TABLE hash_partition_test (
        id INT,
        pdate INT
    )
    PARTITION BY HASH(id)
    PARTITIONS 4;


* 上面的分区对象(id)也可以是一个表达式，表达式的结果必须是整数值。
* HASH 分区可以不用指定 PARTITIONS 子句，则默认分区数为1。
* 不允许只写 PARTITIONS，而不指定分区数。
* HASH 分区的底层实现其实是基于 MOD 函数。


## KEY 分区

KEY 分区和 HASH 分区相似。不同之处在于：

* KEY 分区允许多列，而 HASH 分区只允许一列。
* 如果在有主键或者唯一键的情况下，key 中分区列可不指定，默认为主键或者唯一键，如果没有，则必须显性指定列。
* KEY 分区对象必须为列，而不能是基于列的表达式。
* KEY 分区和 HASH 分区的算法不一样，对于 innodb 引擎，采用的是 MD5 值来分区。


## COLUMNS 分区

可以直接使用非整型的数据进行分区。分区根据类型直接比较而得，不需要转化为整型。同时，可以对多个列值进行分区。


    CREATE TABLE listvardou (
        id INT,
        pdate INT
    )
    PARTITION BY LIST COLUMNS(id,pdate)
    (
        PARTITION a VALUES IN ( (1, 201701), (1, 201702), (1, 201703)),
        PARTITION b VALUES IN ( (2, 201702) )
        PARTITION b VALUES IN ( (3, 201703) )
    );



## 总结

* RANGE 分区，LIST 分区，HASH 分区，KEY 分区对象返回的只能是整数值，如果不是整型，则需要使用函数将其转化为整型。
* 数据表非常大以至于无法全部都放到内存，或者只在表的最后部分有热点数据，其他均为历史数据的情况下，可以选用分区表。
* 分区表数据更容易维护（可独立对分区进行优化、检查、修复及批量删除大数据可以采用drop分区的形式等）。
* 分区表的数据可以分布在不同的物理设备上，从而高效地利用多个硬件设备。
* 可以备份和恢复独立的分区，非常适用于大数据集的场景。
* 分区的主要目的是用于数据库的高可用性管理。