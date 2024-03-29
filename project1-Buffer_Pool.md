---
title: [Project1 - Buffer Pool]
date: 2022-06-15
categories: [DataBase, CMU15-445, 实验笔记]
tags:
- DataBase
- CMU15-445
- 实验笔记
---
# Buffer Pool实现

DBMS的Storage主要就是解决两个问题：

1. 怎么将数据库数据以文件的形式存储在磁盘中。
2. 如何控制内存和磁盘之间的数据交换。Buffer Pool用来解决这个问题。

Buffer Pool 是数据库用来管理内存空间的结构。后面的数据存储都构建在Buffer Pool上层。

通过Buffer Pool分配pages进行写入，并将修改后的pages写入到磁盘。读取数据时，通过将磁盘数据读入Buffer Pool分配的page中（即内存），再进行相关操作。

Buffer Pool也会根据置换算法（这里用的LRU）来将磁盘中的Page读入或写出。

## 整体结构

1. Buffer Pool 在内存中，由一个个frame组成，buffer pool的整个空间是一直在的，即这块内存空间数据库不会释放让操作系统回收，不管有没有有效page。
2. 内存中还有一个Page Table，它负责维护pageid->frame id的映射，从而能够访问page。（还有一个page directory，是负责pageid->location of disk）。
3. 一个LRU Replacer，负责标记哪个frame的page可以先置换出去。

## LRU Replacer

关键数据结构：

```cpp
// 用队列来维护replacer中的frame，新来的从头部插入，最旧的在末尾
std::list<frame_id_t> queue_;
// 用来存放该frame在queue中的位置，方便当Pin调用时将该frame从replacer队列中移除
std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> map_;
```

关键实现逻辑：

1. frame：**frame表示的是buffer_pool中格子的索引**，可以用来存放不同的page，格子总是那么多。
   1. 所以Victim()表示找到该frame并且新读入的page可以写入该frame
   2. Pin表示该frame中的page有用，新来的page不要用该frame
   3. Unpin表示该frame中的page用不上，可以被取代，该frame可写入。
2. LRU的逻辑：很巧妙，**多次Unpin并不表示该page被新使用了，所以Unpin时如果该frame在replacer中并不需要将该frame提到queue的头部**，该page被访问或者被使用的时候都会调用Pin，而这时候会将该frame移出，所以每次被使用的时候都会移出replacer，**新加入的就会比后面的更晚被使用，因为如果后面的有更新被使用的都从queue中被移除了**。

## Buffer Pool Manager

关键数据结构：

```cpp
/** Array of buffer pool pages. 即buffer_pool数组，数组中各个元素指向各个page。该数组的索引即为frame_id*/
Page *pages_;
/** Page table for keeping track of buffer pool pages. 存放page_id到frame_id的映射*/
std::unordered_map<page_id_t, frame_id_t> page_table_;
/** Replacer to find unpinned pages for replacement. 用来实现page替换算法，上面的LRU是它的子类*/
Replacer *replacer_;
/** List of free pages. 记录空着的没用的frame*/
std::list<frame_id_t> free_list_;
```

关键实现逻辑：

1. 索引：指针数组pages_就是buffer_pool，**它的索引就是frame_id**，表示buffer_pool中的位置。
2. 放入buffer_pool：就是先从free_list_中找是否有空着的frame，然后再从replacer中找根据替换原则找到下一个可替换写入的frame，找到该frameid后将要new或者读取的page的指针放入该frame中。
3. 离开buffer_pool：如果该page要被删除，则将该page的数据清除，包括元数据，然后将该frame放入free_list中。如果只是Unpin，则放入replacer等待被替换就好。
4. 实际实现中，**buffer_pool已经初始化好了各个page的数据结构，读入和卸载都只是修改page的数据和元数据，并不是新建或者消除一个Page对象**。
5. 跟硬盘数据的交互有一个disk_manager对象来进行磁盘端的读取，删除和写入。

## Parallel Buffer Pool Manager

关键数据结构：

```cpp
// 指针数组，存放指向各个BufferPool的指针
BufferPoolManagerInstance **bpms_;
// BufferPool的数目
size_t num_instances_;
```

关键实现逻辑：

1. 其实就一个逻辑，根据pageid轮流调用不同的bufferpool，从而分担workload，提高并行度。即将page放在第pageid%num_instances个bufferpool中，类似哈希表。
2. 每个buffer_pool都有一个mutex，即latch，在bufferpool的每个函数上加锁来避免多线程带来的问题.

   ```cpp
   // 加锁，并自动释放
   std::scoped_lock lock{latch_};
   ```

## 总结

整体实现还是比较简单，整体就是要弄清楚下面三个问题：

1. LRU中，多次Unpin并不表示该page被新使用了，所以Unpin时如果该frame在replacer中并不需要将该frame提到queue的头部。新加入的就会比后面的更晚被使用，因为如果后面的有更新被使用的都从queue中被移除了。
2. 指针数组pages_就是buffer_pool，**它的索引就是frame_id**，表示buffer_pool中的位置。
3. buffer_pool已经初始化好了各个page的数据结构，读入和卸载都只是修改page的数据和元数据，并不是新建或者消除一个Page对象。
