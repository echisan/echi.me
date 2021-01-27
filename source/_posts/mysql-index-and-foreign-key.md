---
title: mysql-索引与外键
date: 2019-09-30 11:42:18
tags:
  - mysql
  - innodb
---

## innodb存储引擎索引概述

innodb常见的索引
- B+树索引
- 全文索引
- 哈希索引

> B+树索引并不能找到一个给定键值的具体行。B+树索引能找到的只是被查找数据行所在的页。然后数据库通过页读入到内存，再在内存中进行查找，最后得到要查找的数据。


## 数据结构和算法

> 晚点去学习下B+树

## B+树索引

> B+索引在数据库中又一个特点是高扇出性，因此在数据库中，B+树的高度一般在2～4层。

> 数据库中B+树索引可以分为聚集索引和辅助索引，但其内部都是B+树，即高度平衡的，叶子节点存放着所有的数据。聚集索引与辅助索引不同的是，叶子节点存放的是否是一整行的信息

## 聚集索引

> innodb存储引擎表是索引组织表，即表中数据按主键顺序存放。而聚集索引(clustered index)就是按照每张表的主键构造一棵B+树，同时**叶子节点中存放的即为整张表的行记录数据**，也将聚集索引的叶子节点成为数据页。聚集索引的这个特性决定了索引组织表中数据也是索引的一部分。同B+树数据结构一样，每个数据页都通过一个双向链表来进行链接

## 辅助索引(Secondary Index,也称非聚集索引)

> 叶子节点并不包含行记录的全部数据。叶子节点除了包含键值以外，每个叶子节点中的索引行中还包含了一个书签(bookmark)

书签  
> 该书签用来告诉InnoDB存储引擎哪里可以找到与索引相对应的行数据。由于InnoDB是索引组织表，因此InnoDB的辅助索引的书签就是相应行数据的聚集索引键。

## B+树索引的分裂

没看懂

## Cardinality

如何查看索引是否是高选择性？

可以通过`show index`结果中的列`Cardinality`来观察。Cardinality值非常关键，表示索引中不重复记录的预估值。值得注意的是该值是一个预估值并不是准确的值。实际应用中`Cardinality/n_rows_in_table`应尽可能地接近1.

## Multi-Range Read优化

MySQL5.6开始支持Multi-Range Read(MRR)优化，MRR优化的目的为了减少磁盘的随机访问，并将随机访问转化为较为顺序的数据访问，这对IO-bound类型的SQL查询语句可带来性能极大的提升。MRR优化可适用于range，ref，eq_ref类型的查询。

MRR优化有以下几个好处
- MRR使数据访问变得较为顺序。在查询辅助索引时，首先根据得到的查询结果按照主键进行排序，并且按照主键排序的顺序进行书签查找
- 减少缓冲池中页被替换的次数
- 批量处理对键值的查询操作

对于InnoDB和MyISAM的范围查询和JOIN查询操作，MRR工作方式如下：
- 将查询得到的辅助索引键值存放于一个缓存中，这时缓存中的数据是根据辅助索引键值排序的
- 将缓存中的键值根据RowID进行排序
- 根据RowID的排序顺序来访问实际的数据文件

> 此外，若InnoDB或MyISAM缓冲池不是足够大，不能放下一张表中的所有数据，此时频繁的离散读操作还会导致缓存中的页被替换出缓冲池，然后又不断地读入缓冲池。若是按照主键顺序进行访问，则可以将此重复行为降为最低

## Index Condition Pushdown (ICP) 优化
ICP同样是MySQL5.6开始支持的一种根据索引进行查询的优化方式。

之前的MySQL版本不支持ICP的话，当进行索引查询时，首先根据索引来查找记录，然后再根据`WHERE`条件来过滤记录。

在支持ICP之后，`WHERE`操作放在了存储引擎层，在取出索引的同时根据`WHERE`条件过滤。在某些查询下，可以大大减少上层SQL层对记录的索取，从而提高数据库的整体性能。

## 哈希算法

### InnoDB中的哈希算法

对于缓冲池的哈希表来说，在缓冲池中的Page都有一个`chain`指针，他指向相同哈希函数值的页。

> 对于除法散列，m的取值为略大于2倍的缓冲池页的质数。例如：当前参数`innodb_buffer_pool_size`的大小为10M，则公有640个16kb的页。对于缓冲池内存的哈希表来说，需要分配640*2=1280个槽，但是由于128不是质数，需要取比1280略大的质数，应该是1399，所以启动时会分配1399个槽的哈希表。

那么innodb的缓冲池对于其中的页是怎么进行查找的？

innodb的表空间都有一个space`_id`，用户要查找的应该是某个表空间的某个连续16kb的页，即偏移量offset。innodb将`space_id`左移20位，然后加上这个`space_id`和offset，即关键字`K=space_id<<20 + space_id + offset`，然后通过除法散列到各个槽中去。


