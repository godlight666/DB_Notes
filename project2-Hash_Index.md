---
title: Project2 - Extendible Hash Index
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

根据对key的hash快速索引找到存储目标value的page。读取page的data并根据key找到目标。

### 关键数据结构：

```cpp
// 存储当前的global depth
uint32_t global_depth_{0};
// 存储各个bucket的localdepth
uint8_t local_depths_[DIRECTORY_ARRAY_SIZE];
// 存储各个bucket所在的pageid
page_id_t bucket_page_ids_[DIRECTORY_ARRAY_SIZE];
```

### 关键实现逻辑：

1. <mark>hash索引</mark>：先对key进行hash，再取得到的值的后*global_depth*位，这就相当于mod directory的当前长度。**我这里实现取低位，即least-significant bits（LSB）**。
2. IncrGlobalDepth: 将dir数组扩充为原来的两倍，**由于新的一半都是旧的一半的兄弟（即在上一个global_depth时索引相等）**，所以直接让新的一半的local_depth和bucket_page都复制旧的一半就好了。
3. DecrGlobalDepth: **直接减小global_depth就可以**，就会失去对另一半的访问，相当于释放了。没有对数组进行操作。
4. SetBucketPageId：
   1. 首先需要检查要设置的bucket_idx是否超过了dir的范围，如果是则需要增大global_depth。
   2. 而且由于调用时，肯定是需要将该bucket指向与原本不同的page，所以需要增大自己和当前兄弟（指向同一个旧page）的localdepth（IncrLocalDepth中直接实现了这部分逻辑）。
   3. 然后设置该bucket的pageid时，**需要根据新的localdepth，找到当前的所有兄弟并将新pageid赋给这些兄弟**。
5. IncrLocalDepth: 增大自己**和当前兄弟（指向同一个旧page）的localdepth**。
6. GetOldBro: 找到（global_depth_-1）时的兄弟bucket，在merge时调用。**只需要取后global_depth_位（bucket_idx直接就是）并将local_depth_位取反就找到了**。
7. CanShrink: 表示dir可以缩小规模，如果是则直接DecrGlobalDepth。只需要判断是不是每个localdepth都小于global_depth就行。
8. <mark>Merge</mark>: 在hash table的RemoveMerge中调用，即一个page被删空了，可以和他的oldbro进行合并，指向同一个page。**需要让每一个指向旧page(已经空了)的bucket都指向新page，并且让所有指向新page的localdepth-1，因为他们之前都是兄弟**。在最后用Canshrink检查dir是否可以收缩，如果是则globaldepth减小。

## Hash Table Bucket Page

存放在page的data中。用来存储数据pair，无序，直接就是一个个pair放在数组中。访问的时候直接遍历bucket寻找。

### 关键数据结构：

```cpp
// 位图，该bit为1表示该位置曾经被写入过，作用不大
char occupied_[(BUCKET_ARRAY_SIZE - 1) / 8 + 1];
// 位图，该bit为1表示该位置的值有效（即可读取使用）
char readable_[(BUCKET_ARRAY_SIZE - 1) / 8 + 1];
// 存储pair，key
MappingType array_[BUCKET_ARRAY_SIZE];
```

### 关键实现逻辑

1. IsReadable和SetReadable:前者判断该位是否为1，后者将该位置1。**Readable就表示该位置的pair是有效的存在的，否则就是被删除了，不应该被读取**。

2. GetValue: 直接遍历array_，对所有Readable的位置进行key的比较，如果相等则加入到结果数组中。

3. Insert: 需要遍历所有的readable判断是否有key和value都一样的。同时找到合适的插入位置。

4. Remove: 找到目标pair的位置，**然后在readable中将该位置置为0就好，并不需要实际的删除**。

5. IsFull和IsEmpty: 两者类似，都先判断一整个char（8位）。**需要注意的是，全1为-1，全0等于0，有1不一定大于0，因为用的是补码表示**。

## Extendible Hash Table

调用上面实现的directory和bucket实现Extendible Hash Table.

### 关键数据结构

```cpp
// 存放directory的page
page_id_t directory_page_id_;
// buffer pool对象，用来管理page
BufferPoolManager *buffer_pool_manager_;
// key比较器
KeyComparator comparator_;
// Readers includes inserts and removes, writers are splits and merges
ReaderWriterLatch table_latch_;
```

### 关键实现逻辑

1. 初始化：首先创建一个page用来存放directory_page，并将pageid赋值给*directory_page_id*。接着创建一个bucke_page，让dir中的bucket0指向它。

2. 从buffer_pool中获得page：**从buffer_pool中得到的page是缓冲池中的page，并不是hash table中directory page或者bucket page的父类，hash table的各个page应该放在page的data部分，所以要用GetData()**。
   
   ```cpp
   // 将Page*转化为HashTableDirectoryPage *就不行，因为前者并不是后者的父类，也不是相同的结构
   HashTableDirectoryPage * dir_page = reinterpret_cast<HashTableDirectoryPage *>(buffer_pool_manager_->FetchPage(directory_page_id_));
   // 强制转换的指针是data部分，而不是整个buffer pool 中管理的page
   HashTableDirectoryPage * dir_page = reinterpret_cast<HashTableDirectoryPage *>(buffer_pool_manager_->FetchPage(directory_page_id_)->GetData());
   ```

3. GetValue: 没有什么特别的。根据key，在dir中找到bucket对应的page，读如然后调用该bucekt_page的GetValue就好。

4. Insert: 根据key找到要插入的bucket_page，检查有没有重复的，检查有没有位置，如果可以就插入，如果满了调用SplitInsert。

5. SplitInsert: 创建一个新的bucekt_page，并将要插入的pair插入。接着调用dir提供的SetBucketPage(会同步更新兄弟的page和depth)。然后遍历旧的bucket_page，将上面的pairs分流到旧page和新page中。 注意，**由于已经更新了dir，所以分流时只需要关注该pair要进入的page是哪一个就行**，不用管bucket_idx。注意，SplitInert一定要再次检查page是否满或者是否有相同的，**即后续操作和检查操作一定要在一组锁中，不能分别在两个阶段的锁中**，否则会出现多线程的问题！

6. Remove: 找到目标pair删除即可。

7. RemoveMerge: 删除后bucket_page为空时，且localdepth>0且old_bro的localdepth和自己相等时调用。首先将自己的bucket_page删除。然后调用dirpage提供的merge方法，使自己和兄弟共同指向oldbro的bucket_page并更新localdepth。同样要注意，**检查和merge操作要在一组锁中，不能分成两阶段，否则多线程出错**。并且，**还需要检查新合并的page是否也为空，如果是则需要递归的检查是否可以进行shrink**。否则可能出现删空了但是globaldepth不为0的情况。

## 其他注意点

1. FetchPage后一定要及时的UnpinPage，否则可能导致buffer_pool不够用。

2. table_latch主要控制对于directory的读写，bucket_page的读写控制需要用Page提供的rwlatch。即需要获得Page*来进行读写锁控制，然后再将Page的data转化为bucketpage类型。
   
   ```cpp
   Page *father_page = FetchBucketPage(bucket_page_id);
   father_page->RLatch();
   HASH_TABLE_BUCKET_TYPE *bucket_page = reinterpret_cast<HASH_TABLE_BUCKET_TYPE *>(father_page->GetData());
   bool ret = bucket_page->GetValue(key, comparator_, result);
   father_page->RUnlatch();
   // 释放page
   UnpinPage(bucket_page_id, false);
   ```

3. **锁应该嵌套的获取和释放**。即先获取的锁应该后释放，而不能先获取的锁先释放。否则会发生死锁。

## 总结

1. 从buffer_pool中得到的page是缓冲池中的page，并不是hash table中directory page或者bucket page的父类，hash table的各个page应该放在page的data部分，所以要用GetData()。
   
   ```cpp
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
