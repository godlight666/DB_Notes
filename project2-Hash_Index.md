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
