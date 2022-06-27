---
title: Project #1 - Buffer Pool实现
categories: [DataBase, CMU15-445, 实验笔记]
tags:
- DataBase
- CMU15-445
- 实验笔记
---

# Extendible Hash Index实现

[Project #2 - Extendible Hash Index | CMU 15-445/645 :: Intro to Database Systems (Fall 2021)](https://15445.courses.cs.cmu.edu/fall2021/project2/)

实现一个hash表用来存储数据。有key和value，hash函数对key生效，使用的xxHash。

整体结构：一个hash table对象由以下结构组成

1. 一个directory_page，存储着hashtable元数据，以及索引数组，索引数组存储着该key所在的bucket所在的page的pageid。**它的空间由project 1的buffer_pool分配的Page中的data分配**
2. 很多个bucket_page，存储着数据pair，包括key和value。一个key可以对应多个不同的value，访问时会返回该key的所有value。无需。**它的空间也由project 1的buffer_pool分配的Page中的data分配**
3. 上面两个page都和buffer pool中的page不同，**前者并不是后者的子类。而是buffer pool中的page的data部分存放着这两种page的数据**。

所以根据结构，我们需要先实现directory_page类和bucket_page类，然后再实现hash_table类，通过调用前两个类来实现这个类。

## Hash Table Directory Page



## 总结

1. 从buffer_pool中得到的page是缓冲池中的page，并不是hash table中directory page或者bucket page的父类，hash table的各个page应该放在page的data部分，所以要用GetData()。

   ```c++
   // 将Page*转化为HashTableDirectoryPage *就不行，因为前者并不是后者的父类，也不是相同的结构
   HashTableDirectoryPage * dir_page = reinterpret_cast<HashTableDirectoryPage *>(buffer_pool_manager_->FetchPage(directory_page_id_));
   // 强制转换的指针是data部分，而不是整个buffer pool 中管理的page
   HashTableDirectoryPage * dir_page = reinterpret_cast<HashTableDirectoryPage *>(buffer_pool_manager_->FetchPage(directory_page_id_)->GetData());
   ```

   

2. unsigned int 不能用 >= 0 来循环，因为永远大于0

3. int可以转成unsigned int，反之则能，会出错（测试中是总是会小1）。

4. c/c++中，char 全为1的时候等于-1，而不是255，因为是有符号整数，8位 ，表示范围是-128-127，补码表示。顺便复习一下补码的知识。[二进制的原码、反码、补码](https://zhuanlan.zhihu.com/p/99082236)对于负数，补码相当于求该负数的同余正数（即-2%256=254)，所以-2的补码就是254。符号位刚好也成了1。

5. 在bucket page的IsFull()和IsEmpty()中都用到了4的知识，一定要注意！

6. 关于锁，一定要注意锁的使用，避免死锁发生：

   1. 锁应该嵌套的获取和释放。即先获取的锁应该后释放，而不能先获取的锁先释放。
   2. 在splitinsert中，hadsame的检查一定要和分流在同一个锁中，不能进行分段。

7. Merge和shrink后，有可能还可以进行merge和shrink，即merge后的指向的新page有可能还是空的。所以需要递归继续合并。否则可能删除完后最后globaldepth还是非0。
