---
title: 关于 MySQL InnoDB 引擎的索引知识
date: 2018-06-01 08:00:00
tags: 
- 技术
- MySQL
---

这边文章在草稿箱放了整整１年半啊.

## InnoDB
从 MySQL 5.5.5 版本开始 InnoDB 是 Mysql 的默认存储引擎。

## 索引
### 支持的索引
- 支持全文索引
- 支持 B+ 树索引
- 支持 hash 索引

其中 hash 索引是完全自动的，不能人工干预，InnoDB 引擎会监控表上各个索引页的查询，如果观察到建立 hash 索引可以带来速度提升，则自动建立 hash 索引

**B+ 树索引**
就是我们所说的索引。
B+ 树索引不能直接找到具体数据的行，只能找到数据所在的页，数据库通过把也度入到内存，在内存中查找，最后得到数据。

### 原理
#### 树
二叉树：每个节点有两个子节点，数据量的增大必然导致高度的快速增加。
搜索二叉树：左子树小于根，右子树大于根。
平衡二叉树：2 棵子树的高度差为 1（查找效率共，但维护成本也高）。
B树：是一种自平衡的树。概括来说是一个一般化的二叉查找树，可以拥有多于 2 个子节点（也是 MyISAM  参考的数据结构，ISAM：索引顺序存取方法）。

B+ 树： 通过二叉查找树，再由平衡二叉树，B 树演化来的。因此，B+ 树也是一种平衡树。

![](https://ws1.sinaimg.cn/large/006CUA5Vgy1fzme91elfej30m80a8q4h.jpg)

⤴️ 高度为2 每页可存放3条记录的B+树。

B+ 树元素自底向上插入。所有记录节点都是按键值的大小顺序存放在同一层叶子节点上，由各叶子节点指针进行连接。

#### B+ 树的插入操作

节点未满：直接插入。

节点已满：这时候需要分裂。当一个结点满时，分配一个新的结点，并将原结点中 1/2 的数据复制到新结点，最后在父结点中增加新结点的指针；B+ 树的分裂只影响原结点和父结点，而不会影响兄弟结点，所以它不需要指向兄弟节点的指针

#### B+ 树的删除操作

直接删除：如果不是 Page Index 节点，则直接删除。
删除 Page Index 节点： 直接删除，使用右边的节点作为 Page Index， 同事更新父节点。
合并：删除后，节点数量小于 50%，则和兄弟节点合并。同时更新父节点。

#### B+ 树的查找

![B+ 树的查找示意图](https://ws1.sinaimg.cn/large/006CUA5Vgy1fzmearrmt8j30fi04vwet.jpg)


通常向下读取一个节点的动作可能会是一次磁盘IO操作，不过非叶节点通常会在初始阶段载入内存以加快访问速度。同时为提高在节点间横向遍历速度，真实数据库中可能会将图中蓝色的CPU计算/内存读取优化成二叉搜索树。


### InnoDB 的索引

####  聚集索引
InnoDB使用的是聚集索引，就是按照每张表的主键构建一颗B+树，叶子节点中存放的整张表的行记录数据（数据页）。各数据页之间使用双向链表连接。

聚集索引即是主键。
每张表只能有用一个聚集索引。
数据在逻辑上是顺序的，物理上不是顺序存储的。
聚集索引的排序、查找、范围查找非常快。

#### 辅助索引

也称非聚集索引。叶子节点不包含行记录的全部数据。叶子节点除了包含键值以外，还包含一个数钱，用来告诉引擎在哪里找到数据（指向到聚集索引）。
辅助索引不影响数据在聚集索引中的组织，因此，一张表可以包含多个辅助索引。

假设辅助索引树高3层，聚集索引树为3层。辅助索引 B+ 树中检索 name，需要先经过3次 IO 到达其叶子节点获取对应的主键。接着再经过3次 IO 使用主键在聚集索引 B+ 树中再执行一次 B+ 树检索操作，最终到达叶子节点即可获取整行数据。

聚簇索引的优势:
- 行数据和叶子节点存储在一起，这样主键和行数据是一起被载入内存的，找到叶子节点就可以立刻将行数据返回。
- 保证不管这个主键B+树的节点如何变化，辅助索引树都不受影响。

## 正确建立索引
- 最左前缀匹配
- = 和 in 可以乱序， MySQL 的查询优化器会帮你优化成索引可以识别的形式
- 尽量选择区分度高的列作为索引（Cardinality 值），区分度的公式是  `count(distinct col)/count(*)` ，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1
    - `show index from user` 可查看 Cardinality 值（采样统计，非实时）
    - ![Cardinality 值](https://ws1.sinaimg.cn/large/006CUA5Vgy1fzme9yvis3j31lg09mq58.jpg)

- 索引列不能参与计算
- 尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。