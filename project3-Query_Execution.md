---
title: Project3 - Query Execution
date: 2022-07-10
categories: [DataBase, CMU15-445, 实验笔记]
tags:
- DataBase
- CMU15-445
- 实验笔记
---

# Query Execution

## 结构解析

这个实验的难点之一在于了解Executor相关的代码的结构，主要通过调用其他的类的方法来实现Executors。

## Executors实现

### SEQUENTIAL SCAN

处理语句例子：SELECT col_a, col_b FROM test_1 WHERE col_a < 500，output scheme

遍历整个表，Next()根据输出的格式和输出的条件每次返回一个tuple（**该tuple是原始数据的副本，并不是原始数据**）。

1. 通过catalog中的tableinfo获得目标table的iterator，利用该iterator遍历目标表，获得原始数据（实现了++）

2. 创建一个vector，用来保存要返回的tuple的数据。

3. 通过plan获得output scheme（目标格式），遍历scheme中的每一个column，利用Evaluate生成目标value然后加入结果vector中。

4. 利用vector生成目标tuple。

5. 利用plan中的predicate的evaluate方法检查该tuple是否符合输出规则。

可以看到，都是Evalueate（）方法，column的返回值是用来生成该列的value，predicate的返回值是用来判断是否符合条件，**他们的作用并不相同而且差别很大**。

比如：// SELECT col_a, col_b FROM test_1 WHERE col_a < 500，output scheme用来限制只要col_a和col_b，predicate用来限制col_a < 500

```cpp
for (const Column &column : GetOutputSchema()->GetColumns()) {
      res.push_back(
          column.GetExpr()->Evaluate(&(*iterator_), &(table_info_->schema_)));  // 第二个参数schema是Tuple原有的格式
    }
*tuple = Tuple(res, GetOutputSchema());  // 调用Tuple的构造函数

iterator_++;  // iter_指向下一个Tuple
if (predeicate_ != nullptr && !predeicate_->Evaluate(tuple, GetOutputSchema()).GetAs<bool>()) {
  continue;
}
```

### INSERT

// INSERT INTO empty_table2 VALUES (100, 10), (101, 11), (102, 12)

// INSERT INTO empty_table2 SELECT col_a, col_b FROM test_1 WHERE col_a < 500

来源有两个，直接的数据或者子Executor。调用一次Next就把所有有效的tuple插入。

1. 前者利用原始数据vector<Value>生成tuple，后者直接获得tuple。

2. 调用table的insert插入tuple。

3. 通过catalog获得跟表绑定的所有index，遍历每个index，利用index和tuple生成每个index对应的key，然后在对应的index中插入key，value是tuple对应的rid。

```cpp
// 来源是原始数据
std::vector<Value> values = plan_->RawValuesAt(row_index_);
      temp_tuple = Tuple(values, &(target_table_->schema_));
      target_table_->table_->InsertTuple(temp_tuple, &temp_rid, txn_);
      lock_mgr_->LockExclusive(txn_, temp_rid);
      for (auto &index : table_indexs_) {
        Tuple key_tuple =
            temp_tuple.KeyFromTuple(target_table_->schema_, index->key_schema_, index->index_->GetKeyAttrs());
        index->index_->InsertEntry(key_tuple, temp_rid, txn_);
        // 不确定tuple用temp_tuple还是key_tuple
        txn_->AppendIndexWriteRecord(IndexWriteRecord(temp_rid, target_table_->oid_, WType::INSERT, temp_tuple,
                                                      temp_tuple, index->index_oid_, catalog_));
      }
```

### UPDATE

// UPDATE test_3 SET colB = colB + 1;

没什么特别的，调用GenegrateUpdatedTuple生成要更新的tuple，然后调用table的updateTuple更新tuple。最后删除原始的index，加入新的index。

同样是调用一次Next（）就完成所有update，next不返回tuple。

### DELETE

// DELETE FROM test_1 WHERE col_a == 50;

从child_executor获得tuple然后调用table的MarkDelete，再删除对应的indexs就行了。

每次next()删除完所有的tuples。

### NESTED LOOP JOIN

1. 初始化left_child_executor和right_child。

2. 对每一个left_child中的tuple都遍历一次right table

3. 利用plan中的predicate的**EvaluateJoin**判断这两个tuple是否符合join的需求，如果是，则遍历output schema中的每一个column，使用column中的**EvaluateJoin**生成join后的value，加入到结果数组中。

4. 使用结果数组生成结果tuple。

5. 每次Next返回一个tuple。

### HASH JOIN

用一个表建hash 表，再遍历另一个表的每一个tuple，寻找相同key的再进行join

1. 构建能够用来进行join的hash。我的方式是key是join的key（通过plan获得joinkey的expression，然后用exper的evaluate方法获得joinkey），value是vector<Tuple>，存储了该key的多个tuple。使用unorderedmap需要实现==和hash函数。

```cpp
namespace bustub {
struct JoinKey {
  Value col_val_;
  explicit JoinKey(Value val) : col_val_(val) {}
  bool operator==(const JoinKey &other) const { return (col_val_.CompareEquals(other.col_val_) == CmpBool::CmpTrue); }
};
}  // namespace bustub
namespace std {
template <>
struct hash<bustub::JoinKey> {
  std::size_t operator()(const bustub::JoinKey &join_key) const {
    size_t cur_hash = 0;
    if (!join_key.col_val_.IsNull()) {
      cur_hash = bustub::HashUtil::CombineHashes(cur_hash, bustub::HashUtil::HashValue(&join_key.col_val_));
    }
    return cur_hash;
  }
};
```

2. 在Init步骤中，利用left table建立hash table，由于条件说明，该table可以在内存中放下。

3. 然后在next步骤中，每次遍历一个right table中的tuple，尝试进行join并返回目标tuple（和上面nested loop join 一样）。

### AGGREGATION

实现聚合函数如count，min，sum，max，相关的语句有group by和having。

使用了一个hash表来存储，这个hash 表的key是自定义的，**组成为group by中的列对应的values**，即vector<value>。value是聚合函数的值，比如count就每次对应的key+1，min就每次对应的key取最小值。

1. Init中遍历table中的每一tuple，makekey和makevalue将pair加入hash表中。

2. Next中遍历hash表的每一值，生成tuple。

3. 使用plan中的having（可能为空）的evaluate**判断是否符合having的筛选**。

### Distinct

使用一个unorderedset记录下每一个加入的tuple（同样要实现一个类，该类的值为tuple的各个value，需要实现==和hash），然后遍历每一tuple，查看set中是否有记录，没有记录就可以返回并且加入set中。

### limit

限制数量就可以。定义个变量记录已经返回的数量。















1. ExecutionEngine: 将Plan转化为对应的Executor，但是在这里需要处理Executor的错误。

2. Update的时候，Engine只执行一次Next，否则result set的size不为0。所以update里面一次Next就要用循环获取所有的child返回的tuple来进行更新。这点与其他的操作不一样。

3. Nested Loop Join: 内表要遍历多次，调用内表Executor的Init()来使内表指向开头。

4. Hash Join: 在Init阶段就用left table构建hash表，key为给定的key， value为tuple 的vector形式。需要注意的是，左表中可能有key相同的多个值，所以遍历右表时需要遍历左表形成的hash表中多个同一个key的pair。

5. 对于没有实现 = 的类，成员变量的初始化需要在构造函数的": "部分中进行。比如aggregation Executor中hashtable的初始化。

6. Aggregation: 主要是要注意不同Expression的使用，group和aggregate都是用evaluate构造hash table的key和value，然后用having的evaluateAggregate筛选，再用各个列的evaluate构造返回的tuple。重点理解having在执行中的顺序。

7. 注意explicit关键字
