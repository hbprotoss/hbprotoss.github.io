<!-- 
.. link: 
.. description: 
.. tags: Database
.. date: 2013/11/16 19:31:34
.. title: InnoDB引擎索引学习笔记
.. slug: innodbyin-qing-suo-yin-xue-xi-bi-ji
-->

最近在研究学习MySQL，本文记录下索引相关（主要B-Tree索引）的特性。

---

## 索引是什么

数据库索引，是数据库管理系统中一个排序的数据结构，以协助快速查询、更新数据库表中数据。

按所用的数据结构分，有B-Tree索引（B+ Tree）、哈希索引、R-Tree索引等。按数据块的顺序和索引节点的逻辑顺序是否一致可以分为聚集索引和非聚集索引。聚集索引由于物理块连续，在范围扫描的时候可以减少磁头寻道时间，因而比非聚集索引高效。

## InnoDB索引结构

InnoDB引擎的索引主体结构是B+Tree（这种数据结构的描述可以参考CSDN博主v_JULY_v的[从B树、B+树、B*树谈到R 树](http://blog.csdn.net/v_JULY_v/article/details/6530142 "")）。

其中，主键索引(Primary Index)是聚集索引，中间节点是用于查询的边界值，叶子节点就是数据节点，叶子节点之间通过指针形成双向链表。如图：

![primary_index.jpg](../galleries/InnoDB/primary_index.jpg "")


辅助索引(Secondary Index)同样为B-Tree索引，只是叶子节点不是数据节点，而是对主键的引用。

![secondary_index.jpg](../galleries/InnoDB/secondary_index.jpg "")

由于此特性，InnoDB引擎要求数据表必须有一个主键，如果没有主键，在数据表创建的时候会自动生成一个唯一非空索引替代。这个特性也决定了InnoDB表的主键最好是一个自增的ID序列，这样在后续插入的过程中不会造成数据块的唯一，提升性能。

当所建立的索引为多列索引的时候，例如KEY(a,b,c)，中间节点依旧以最左边的一列a进行索引，只是在叶子节点上保存的不仅仅是a，而是(a,b,c)多字段排序后的序列。

## 性能实测

下面结合实际例子阐述现象和内部结构的关系

首先建立一个测试表：

```sql
CREATE TABLE test (
     id INT NOT NULL AUTO_INCREMENT,
     a INT NOT NULL,
     b INT NOT NULL,
     c INT NOT NULL,
     PRIMARY KEY (id),
     KEY(a,b,c)
)ENGINE=InnoDB;
```

数据项为(a,b,c)在0~200之间的所有组合项，共计8000000行记录。为了消除几次实验中以query cache为代表的各种server cache对实验数据的影响，每次实验后均重启mysql服务。

### 索引对查询效率的影响

首先是建立KEY(a,b,c)索引前后的性能对比，能很明显地看到索引大幅提升查询速度。没有索引的时候需要做全表扫描，建立索引之后通过B-Tree搜索。

```
没有索引（除主键，下同）
[SQL]select * from test where a = 0 and b = 1;
Affected rows: 0
Time: 0.324s

建立KEY(a,b,c)
[SQL]select * from test where a = 0 and b = 1;
Affected rows: 0
Time: 0.004s

```

由于MySQL优化器的优化，where条件中的各个子条件顺序对索引的使用没有影响。

```
[SQL]select * from test where b = 1 and a = 0;
Affected rows: 0
Time: 0.002s
```

当使用的列非严格的索引最左前缀时，可以想象，只有a加速了索引搜索，找到所有符合a条件的记录的之后只能顺序扫描检查c是否符合条件。所以相比没有索引时要快，但是慢于使用严格索引最左前缀时的速度。

```
[SQL]select * from test where a = 0 and c = 1;
Affected rows: 0
Time: 0.043s
```

当没有使用到a列作为条件时完全没法使用索引，甚至比没有索引时还要慢。因为建表的时候a在前，b、c在后，并且数据节点有序。在没有索引的测试中，InnoDB引擎到a不为0时自动停止搜索。而在这个例子中需要扫描全表找出满足符合条件的b、c，达到完全没法忍受的16秒。

```
[SQL]select * from test where b = 0 and c = 1;
Affected rows: 0
Time: 16.023s
```

### 索引对排序效率的影响

只要能用到索引，并且ORDER BY后面的列在索引中，则顺序还是倒序排序对性能无影响。因为叶子节点是个双向链表，倒序可以从后向前遍历，与顺序的代价相同。

```
[SQL]select * from test where a = 0 ORDER BY b;
Affected rows: 0
Time: 0.085s

[SQL]select * from test where a = 0 ORDER BY b DESC;
Affected rows: 0
Time: 0.089s
```

但是，当有两个字段排序的顺序不同时，叶子节点的有序性就无法用在排序中，必须读出所有叶子节点之后再按指定的字段进行一趟排序，所以速度就慢了下来。

```
[SQL]select * from test where a = 0 ORDER BY b DESC, c DESC;
Affected rows: 0
Time: 0.089s

[SQL]select * from test where a = 0 ORDER BY b ASC, c DESC;
Affected rows: 0
Time: 0.184s
```

同样，下面的例子中对索引左边的字段a进行范围选择的时候也没法运用索引，因为多个取值的a无法保证b和c字段有序，必须重新排序

```
[SQL]select * from test where a <= 0 ORDER BY b;
Affected rows: 0
Time: 0.103s
```

所以，尽可能做到覆盖索引（即where，order by中用到的字段被索引覆盖），能利用索引加速搜索和排序。

---

参考资料：

《高性能MySQL》，电子工业出版社，2010年1月