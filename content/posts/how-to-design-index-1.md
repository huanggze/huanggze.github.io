---
title: "如何设计 MySQL 索引（一）：基础概念"
date: 2023-01-14T11:48:06+08:00
toc: true
categories: ["mysql"]
---

## 1. B+ Tree

MySQL中的InnoDB引擎使用B+Tree结构来存储索引，可以尽量减少数据查询时磁盘IO次数，同时树的高度直接影响了查询的性能，一般树的高度维持在 3~4 层[^1]。B+Tree中，只有叶子节点存储数据。

![](https://segmentfault.com/img/remote/1460000038921159)

从【根节点】到每个【叶子节点】的距离是相等的，也就是访问任何一个叶子节点，需要的 IO 是一样的，即索引树的高度 + 1 次 IO 操作。

## 2. 数据页

当某个索引很大，大到达几十个G，远远超过了内存的容量时，MySQL不可能把整个索引全部加载到内存。MySQL 的做法是从磁盘以【页】为单位加载数据到内存。MySQL通过将索引拆分成页，一页为其最小数据单元，每次读取数据时，加载其中某几页数据。写也是按页，刷新一页的数据到磁盘。【页】是InnoDB存储引擎管理数据库的最小磁盘单位，一个页的大小一般是16KB[^2][^3]。


![](/images/how-to-design-index-1.png)

![](/images/how-to-design-index-2.png)

![](/images/how-to-design-index-3.png)

## 3. 主键索引、辅助索引

主键索引：数据按照主键顺序存储（逻辑上是连续，物理上不连续），存储整行数据。如果没有显示的指定主键，MySQL 会将所有的列组合起来构造一个row_id作为primary key。由于叶子节点包含了完整的数据记录，所以主键索引又叫聚簇索引（clustered index）；

辅助索引：也称为二级索引（secondary index），索引中除了存储索引列外，还存储了主键id，对于user_name的索引idx_user_name(user_name)而言，其实等价于idx_user_name(user_name, id)，MySQL会自动在辅助索引的最后添加上主键id，**不需要要用户自己显式写出来**。

## 4. 页分裂

如果自定义主键 id 不是自增的（乱序），有发生页分裂的可能。考虑以下情况，如果页内存储的键值为 [20, 21, 22, 23, 24, 25, 26]，如果新增键值 27 时页满需要发生分裂。

![](https://pic3.zhimg.com/80/v2-750b0a4f535435653c13cdcb0c853d06_1440w.webp)

创建一个新页，确定在 23 的位置分裂，移动记录至新页。

![](https://pic3.zhimg.com/80/v2-eb81b65c29711b609e2076af48c17146_1440w.webp)

![](https://pic1.zhimg.com/80/v2-0d05a58b7f1f1856894985e015db49c0_1440w.webp)

## 5. 页合并

当你删了一行记录时，实际上记录并没有被物理删除，记录被标记（flaged）为删除并且它的空间变得允许被其他记录声明使用。[&]

![](https://pic1.zhimg.com/80/v2-6a9fd05c70648eabfb153f9c38b081d4_1440w.webp)

当页中删除的记录达到MERGE_THRESHOLD（默认页体积的50%），InnoDB会开始寻找最靠近的页（前或后）看看是否可以将两个页合并以优化空间使用。刚好下一页使用了不到一半的空间，前一页又有足够的删除数量，它们能够进行合并。

![](https://pic3.zhimg.com/80/v2-f3497d7e821abdfd51797a80bf0e68c2_1440w.webp)

合并操作使得前一页数据保留，并且容纳来下一页的数据。原来下一页变成一个空页，可以接纳新数据：

![](https://pic2.zhimg.com/80/v2-ed174fc87f4037da0ed81cd1268653b5_1440w.webp)

![](https://pic3.zhimg.com/80/v2-345d5647e15879a7342895d07fe1906e_1440w.webp)

## 6. 文件排序

在 MySQL 中的 ORDER BY 有两种排序实现方式：利用有序索引排序（最优，using index）和文件排序（using filesort）。

> 只有当ORDER BY中所有的列必须包含在相同的索引，并且索引的顺序和order by子句中的顺序完全一致，并且所有列的排序方向（升序或者降序）一样才有，（混合使用ASC模式和DESC模式则不使用索引）[^4]。

![](/images/how-to-design-index-4.png)

当排序数据量不超过 sort_buffer_size 系统变量所设置的容量时，MySQL 将会在内存使用快速排序算法进行排序（内部排序）；当排序数据量超过 sort_buffer_size 容量时，MySQL 将会借助临时磁盘文件使用归并排序算法进行排序（外部排序）在进行真正排序时，MySQL 又会根据数据单行长度是否超过 max_length_for_sort_data 而决定使用主键排序还是全字段排序，优先选择全字段排序，以减少回表次数[^5]。

![](https://dl-harmonyos.51cto.com/images/202204/33082ec07b3468cf1823500588c010202b686c.png)

## 7. 复合索引

指包含多个数据列的索引，与之概念相对的是单列索引，仅包含一个数据列。

## 参考资料

[^1]: [阿里面试官：MySQL如何设计索引更高效？](https://segmentfault.com/a/1190000038921156)
[^2]: [MySQL索引-页结构](https://www.cnblogs.com/caoxb/p/15526053.html)
[^3]: [InnoDB 数据页](https://juejin.cn/post/6974225353371975693)
[^4]: [mysql中的文件排序(filesort)](https://www.cnblogs.com/chafanbusi/p/10648026.html)
[^5]: [mysql 内存排序_MySQL 排序的艺术：你真的懂 Order By 吗？](https://blog.csdn.net/weixin_39997400/article/details/114331759)