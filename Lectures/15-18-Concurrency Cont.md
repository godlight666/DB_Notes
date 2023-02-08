# Concurrency Control

## Timestamp Ordering

### timestamp allocate

1. System Clock：系统时间，缺点是夏令时的时候会回调一个小时，不是单调递增。
2. Logical Counter：用一个寄存器或者什么维护一个整数，单调递增。缺点是这个整数有上限，比如32位。
3. Hybrid：混合。

### 基本操作（Basic Timestamp）

1. 给每个值X都记录两个timestamp，一个是最后读该值的txn的timestamp（R-TS)，一个是最后写该值的timestamp（W-TS）。**在事务开始的时候**。
2. 读操作：读的事务的TS（Ti）<W-TS(X)，abort Ti 然后重启创建一个新的TS（就有更大的TS)。否则，允许事务TS读，并更新R-TS（x），**并且要创建一个本地的copy of X来确保Ti能够重复读到现在这个X**。
3. 写操作：如果TS(Ti) < W-TS(x) 或 Ts(Ti) < R-TS(X)，则abort Ti然后restart Ti（分配更大的timestamp）。否则，允许写，如果要可重复读，就也要创建本地X副本。
4. **写操作优化**：如果Ts(Ti) < R-TS(X)，abort并restart。**如果TS(Ti) < W-TS(x)，则跳过这个写操作继续执行下面的操作**。（因为对于外部来说，这个写操作总是要被覆盖的。
5. 特点：
   1. schedule是conflict serializable的。没有死锁。
   2. 对于长的事务，可能会导致starvation。就是一两个新的短的事务修改了数据，导致长事务被abort然后restart。
   3. 不是recoverable的。（一个事务在所有修改它读的数据提交后再提交，否则就不能保证重启之后还能读到相同的值）。
   4. 对于3，**其实就是会读到脏数据**（即没有commit的数据）。

### 乐观并发控制（Optimistic Concurrency Control）

 DBMS给每个事务都分配了一个单独的workspace，

将每个操作涉及的tuples都放到workspace中。读会把tuple复制到workspace中，写只是applied to workspace。Commit后，进行检查，再整体应用到global database中。

**每个tuple只有W-TS，没有R-TS**。

总共有三个阶段。

#### Work Phase

所有操作都在这个阶段执行。保存read set和write set，用在下一阶段。

#### Validation Phase

验证是否有冲突，能否提交到global database。**在这个阶段分配timestamp**。没分配前的timestamp都是无限大。

向前验证：检查所有并发执行的并且已经commit的txn（timestamp比自己小），看read/writeset是否有冲突（比如前一个修改了一个值，但是自己读到的还是旧的），如果有，就abort自己然后restart。

向后验证：检查所有并发执行的并且还没commit的txn（timestamp比自己大），看read/write是否有冲突，如果有就abort**自己这个txn**。

#### Write Phase

写入global database。

#### 适用的场景和优缺点

1. 相比于basic timestamp，前者是每进行一个操作就要检查这个操作的可行性，这个是到最后再来验证。
2. 在大量读请求的情况下，或者事务之间的数据操作交集不多的情况下（数据量很大而事务不多时，这个概率很大），就很适合用这种方法，不用请求锁。（**只是不用请求数据库的lock，底层的数据结构的操作还是要latch，不是同一个层面**）
3. 问题：
   1. 要复制到自己的workspace，消耗很大。
   2. validation和write阶段不能并行，是bottleneck。
   3. 但是在冲突很多的情况下，会导致事务做了大量的工作但是最后还被abort了。消耗比2PL大。
   4. 在validation阶段，多个事务同时查看一个事务本地保存的数据（read/Write set），还是要竞争这个数据结构的latch，因为这个事务可能还在执行，这个set可能会变。

## 幻像读（Phantom Problem）

发生在对一个范围tuple进行查询，tuple锁就无法控制了。解决方法有下面三种：

1. re-execute scan：在commit的时候，重新执行一遍看结果是不是这样
2. predicate lock：select的时候上共享锁，update，insert，delete的时候上排他锁。就是对一个条件上锁，锁住所有相关的tuple，负担很大，很少用。
3. **index locking：如果涉及的属性有索引，就直接在索引上加锁，这样跟该属性相关的tuple的增加或删除都会要获取这个索引的锁。如果还没有对应的值，就在索引的间隙上加锁，即间隙锁。如果不涉及任何index，就要保证任何已有的tuple不能改变成符合条件的tuple，任何符合条件的tuple不能被增加或者删除。**
