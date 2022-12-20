---
title: Join Algorithm
date: 2022-07-03
categories: [DataBase, CMU15-445, 课程笔记]
tags:
- DataBase
- CMU15-445
- 课程笔记
- Join
---

# Join Algorithm

Join操作几乎是数据库查询中最常用的操作。它能够联合多张表的信息对数据进行查询和提取。Join操作有以下几种操作模式。

1. inner join：最常用的模式。**使用“join”时默认采用该模式**。需要指定连接的列，根据连接的列，找到两表之间相同的值的那一行然后连接在一起。**默认会将小的表作为左表，将右表的值往左表中加**。能够加速执行速度。原理下面会解释。

2. natural join：自然连接。原理与inner join基本一样，不指定列时会自动选择同名列然后进行join。**使用时两表不能超过两个同名列。**

3. left/right join：以左（右）表为基础，在右（左）表中找和左（右）表相同的，然后加进左（右）表中。左（右）表的数据都不会被删除，都会在结果中。

4. full outer join：全外连接。就是笛卡尔积。就是左表的每一个tuple都和右表的每一个tuple连接。两者都不会被删除任何数据。

<img src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_5.png" title="" alt="" data-align="center">

这里讲的原理围绕**自然连接natural join(inner equijoin)** 来讲。

输出有两种类型：直接输出表，各个列直接放的就是value。还可以放records id，读的时候再根据id去找。后者适合列存储数据库。

join算法评价指标：表1M个pages，m个tuples。表2 N个pages，n个tuples。需要用这四个数据表示出IO次数，根据IO次数来评估算法。

## Nested Loop Join

几乎所有数据库都提供的方式，很简单。

### Simple/Stupid

最简单的方式，也是最满的方式。就是用两重循环，第一重循环遍历表1的各个tuples，第二重循环遍历表2的各个tuples。如下所示。

```
foreach tuple r in R:    //Outer table
    foreach tuple s in S:    //inner table
        emit, if r and s match
```

**所以我们希望Outer table（外表，即左表）是数量小的那个表**，因为当m<n时，M+(m * N)<N+(n * M)。这也是这种方式<mark>IO次数</mark>，可以看到，很大。表R读进来一次，用了M次IO，表S读进来m次，用来m*N次IO。

### Block

就是加速计算的常用方法，一次性多读一些进cache并且反复利用读入cache的数据。

```javascript
foreach block Br ∈ R:
    foreach block Bs ∈ S:
        foreach tuple r ∈ Br :
            foreach tuple s ∈ Bs :
                emit, if r and s match
```

这种方式同样希望outer table是小表，不过希望的是M<N，因为这样 M + (M * N) < N + (N * M)。可以看到这样<mark>IO次数</mark>比上面的小了。

上面的方式在读Outer table时，一次只读了一个page大小的数据（即一个block只有一个page大小）。当内存有B个pages的空间时，可以用B-2个pages读outer table，一个page读inner table，一个page做输出。

```
foreach B - 2 blocks Br ∈ R:
    foreach block Bs ∈ S:
        foreach tuple r ∈ B - 2 blocks:
            foreach tuple s ∈ Bs :
                emit, if r and s match
```

这样<mark>IO次数</mark>又缩小了B-2倍，即M + ( M / (B-2) ∙ N)

### Index

如果表S在要合并的列上构建了索引，索引消耗为常数C，那么就只需要遍历M，在S的索引上每次搜索M中每个tuple在S中是否有对应。

IO次数为 M + (m * C)。

## Sort Merge Join

也是有两个阶段。就是利用有序性来避免反复重复的访问。<mark>IO次数</mark>如下所示。包括两个阶段的IO总和。

<img title="" src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_8.png" alt="" width="723" data-align="center">

### Sorting

两个表各自自己内部根据要合并的列进行排序，排序的方式可以用上一节课的**外部归并排序（External Merge Sort）**，如果内存够大就直接快速排序。总之就是根据相同的列各自进行排序。

### Merge

用两个cursor分别指向两个表，然后用下面的算法进行匹配来join。

<img title="" src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_7.png" alt="" width="660" data-align="center">

## Hash Join

当使用同一个hash_func时，对同一个key，总是会得到相同的值。所以利用这个特性，对相同的key进行hash就会进入相同的分区（bucket），所以只需要在分区内查找就好了。

### Basic

当内存能够支持放下整个表生成的hash table时，就用这种方式就好了。

#### Build

遍历表R的每一个tuple，根据join key进行hash，得到一个hashtable。

#### Probe

遍历表S的每一个tuple，对join key进行hash，然后在hashtable中找匹配的值，然后一起加入结果表中。

<img src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_9.png" title="" alt="" width="750">

### Bloom Fliter优化

很巧妙。Bloom Fliter是一个bitmap，可以用很小的空间，很快的速度查询到**是不是没有这个值**（即它说有不一定有，但是它说没有肯定没有）。

[Bloom Fliter原理](https://zhuanlan.zhihu.com/p/50587308)

所以表S遍历时，先去Bloom Fliter中查询，如果没有，就不用去Hash Table中查了。否则，再进一步去Hash Table中查。

### Grace Hash Join

当内存中放不下一整个Hash Table时，需要用这种方式。也分为两个阶段。

整体思想利用了当key相同时，会被hash到同一个分区，所以只需要比较两个hash table的同号分区就好了。

#### Build

需要先根据表R构建hash table，该hash table有b个buckets。**然后对表S用相同的hash函数构建另一个hashtable，和表R的hashtable的buckets数目相等**。

#### Probe

将两个hash table的同号分区进行比较，找出key相同的并输出结果。因为key相同的一定会被hash函数分到同号的分区（bucket）中。

<img title="" src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_12.png" alt="" width="862">

但是有个问题：**有可能太多的key被分配到一个bucket中，导致一个bucket装不下，这时就需要recursive partitioning**。就是对于一个bucekt中有多个pages的，再进行迭代的hash（用不同的hash函数），直到将bucket的page展开。另一个page在寻找的时候，也会根据元数据知道那些bucket需要进行二次hash来匹配。

<img src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_13.png" title="" alt="" width="836">

<mark>IO次数</mark>：

build时要用 2 * (M + N)，因为要读M + N个pages，写M + N个pages（根据两个表生成两个hash表）

probe时要用(M + N)，只需要读入这两个hash 表进行比较就可以了。

## 总结

1. 什么时候用 Nested Loop Join: 
   
   1. 当joinkey重复特别多时，比如所有的join key就一个值的时候用。即下面的两种方法失效的时候用。

2. 什么时候用Sort Merge Join:
   
   1. 一个表或者多个表已经在join key上有序了。
   
   2. 输出要求是有序的。
   
   3. 查询计划中的其他指令要让表有序。
   
   4. 总之就是减小排序损耗时sort merge是最快的。

3. 什么时候不能用 Sort Merge join:
   
   1. join key高度重复。

4. 什么时候用 Hash Join：
   
   1. 普通情况下Hash Join就最快。
