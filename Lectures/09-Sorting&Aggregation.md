---
title: Sorting & Aggregation in Database
date: 2022-07-03
categories: [DataBase, CMU15-445, 课程笔记]
tags:
- DataBase
- CMU15-445
- 课程笔记
- 外部排序
---

# Sorting & Aggregation

## External Merge Sort

### Why we need sort

1. 关系模型是无序的，有时候查询要求结果按照某种顺序排列（order by）

2. 除了上面这种最直接的需求，还有很多间接的需求，需要利用有序的特性
   
   1. 用来去重（distinct）
   
   2. 有序的加入B+ tree index会更快
   
   3. 聚合Aggregation（group by）
   
   4. 。。。

### how to sort

1. 可以放入内存时，直接用各种排序算法就好（比如快排）

2. 往往内存中一次性放不下，需要使用外部排序算法，比如**External Merge Sort**

### External Merge Sort

外部归并排序。

主要有两个阶段：

1. 将整体数据划分为一个个内存能够放下的RUNs，并在每个RUN内部对数据根据规则进行排序。

2. 将两个（或多个）排序好的RUN进行merge，<mark>就是用两个（或多个）指针，取小的写入目标pages，并且写满了一部分（一个page）就将该部分flush进磁盘</mark>（或者其他的大的存储空间）

例如，当用2-way merge时流程和cost如下图所示：先将每个RUN（这里是一个page大小）内部排好序，然后每次合并2个RUNs（内部有多个page），并迭代下去。所以PASS的数目时LOG2。

<img title="" src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_2.png" alt="" width="741" data-align="center">

<mark>但是上图的局限性在于</mark>，当buffer_pool中可以读入大于3个page（2个输入1个输出）时，并没有充分利用，因为只能用上两个RUNs，所以希望能够充分利用多个buffer page，可以一次性合并多个RUN。当一次性可以读入B个pages时，时间复杂度就变为如下图所示。

在第一阶段，即Pass #0时，<mark>将N个pages（总数据）一次性读入B个pages，B个pages内部进行内存内的排序，变为一个RUN（B个pages）。从而将总数据分为N/B个RUN</mark>，每个RUN内部有序。

在第二阶段，就开始合并各个RUNs成为一个更大的RUN，由于需要留一个page空间作为输出（写满一个page就可以flush，所以不管RUN多大留一个就可以），并且对于每个RUN，每次只需要往内存中读入一个page就行。<mark>所以每次可以同时合并B-1个RUN成为一个RUN。由于总共有N/B个RUN，所以总共需要LOG(B-1)(N/B)轮合并</mark>。

<img title="" src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_3.png" alt="" width="683" data-align="center">

## Aggregations

就是GROUP BY，只需要将一样的放在一起就行。有两种实现方式

### Sorting

很直接的，就是根据GROUP BY选中的列进行排序，一样的就会在一起了。

优化的方式就是先根据其他语句过滤掉不用的行（WHERE），不用的列（比如不在select中的），根据对应列进行排序就是。

### Hashing

如果不要求顺序（比如distinct或GROUP BY），用hash的损耗会比sorting小的多。

如果内存装的下（即一整个hash table都可以放在内存中），直接就用hash去重或者分组（hash到同一个）。

如果内存装不下，就需要用下面的方法，分为两个阶段：<mark>先用一个hash分区，分成能放入内存的区。然后再用另一个hash来得到结果。这个也是分治的思想。</mark>

#### 阶段一 Partition

思想就是先用一个hash来将所有的数据分区。由于同一个key得到的hash就是一样的，所以会分在同一个区。当Buffer pool有B个pages时，用B-1个pages来装分区的，用剩下的一个来存放结果。所以第一个hash会得到B-1个分区。

#### 阶段二 Rehash

这个阶段需要对上面得到的B-1个分区进行操作。对每个分区进行第二个hash（与阶段一的hash不同），生成这个分区的hash表，然后根据查询逻辑将这个分区表的结果放入结果数组中。并且如下所示，有时对一个key的多个value会追加到一个tuple中，方便进行运算。

<img title="" src="https://raw.githubusercontent.com/godlight666/picgo/main/screenshot_4.png" alt="" width="563" data-align="center">
