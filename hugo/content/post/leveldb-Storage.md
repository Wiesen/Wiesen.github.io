+++
date = "2016-12-20T21:31:20+08:00"
draft = false
title = "Leveldb: Storage MemTable"
topics = ["Database", "Leveldb"]
+++

<div style="text-align: center">
<img src="http://7vij5d.com1.z0.glb.clouddn.com/leveldb-architecture.png" width="500"/>
</div>

Memtable (`db/skiplist.h db/memtable.h&memtable.cc`)
---

1. Introduction

    MemTable 是 leveldb 的 kv 数据在内存中的存储结构。当 Memtable 写入的数据占用内存到达指定大小 (Options.write_buffer_size)，则自动转换为 Immutable Memtable，同时生成新的 Memtable 供写操作写入新数据。后台的 compact 进程会负责将 immutable memtable dump to disk 生成 sstable。

    Memtable 类只是一个接口类，其底层实现依赖于两大核心组件 Arena 和 SkipList，Arena 内存分配器统一管理内存，SkipList 用于实际 KV 存储。

    根据上一篇 blog 中 LSM tree 的性质，table 应当保持有序性。而对一个排序结构执行插入操作开销很大（随机写），通常性能瓶颈集中在这里。一些数据结构如链表, AVL 树, B 树, skiplist 等都加速了随机写。leveldb 的 memtable 实现没有使用复杂的 B 树，采用更轻量级的 [skiplist](https://en.wikipedia.org/wiki/Skip_list)。
    
    skiplist 是一种可以代替平衡树的数据结构。其从概率上保证数据平衡，结构和实现比平衡树简单。概率上时间复杂度近似 O(logN)，与平衡树相同，但空间上比较节省，一个节点平均只需要 1.333 个指针。


2. Interface

    1. 成员变量

            typedef SkipList<const char*, KeyComparator> Table;
            KeyComparator comparator_;
            int refs_;    // 引用计数
            Arena arena_; // 内存池
            Table table_; // Table 就是 SkipList<const char*, KeyComparator>

    2. 共有函数
 
            explicit MemTable(const InternalKeyComparator& comparator);
            void Ref();
            void Unref();
            size_t ApproximateMemoryUsage();
            Iterator* NewIterator();
            void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value);
            bool Get(const LookupKey& key, std::string* value, Status* s);
        
    3. 私有函数
            
            ~MemTable();  // Private since only Unref() should be used to delete it
            // No copying allowed
            MemTable(const MemTable&);
            void operator=(const MemTable&);

3. Analysis

    MemTable 的对象构造必须显式调用，重点在于初始 Arena 和 SkipList 两大核心组件，且提供引用计数初始为 0，使用时必须先 Ref()，实际对象销毁在 Unref() 引用计数为 0 时。
    
    此外，LevelDB 中禁止类被复制的方法都是声明拷贝构造函数和赋值操作符为 private，并且只提供声明，而不提供定义。（析构函数同理）（C++11 中可以使用 = delete)

    memtable 对 key 的查找和遍历是 MemTableIterator，而 MemTableIterator 实际上是 SkipList iterator wrapper。NewIterator 即是返回一个 MemTableIterator，用于有序遍历 memtable 的存储数据。

    写入 `void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value)`:
    
     1. 将传入的参数封装为 `Internalkey`，然后与 `value` 一起编码成上述 entry
     2. `SkipList::Insert()`

    读取 `bool Get(const LookupKey& key, std::string* value, Status* s)`:
     
     1. 从传入的 `LookupKey` 中获取 `memtable_key`
     2. `MemTableIterator::Seek()` 返回 `MemTableIterator`
     3. seek 失败，返回 data not exist。seek 成功，则判断数据的 ValueType：a) `kTypeValue` 则返回对应的 `value` 数据; b) `kTypeDeletion` 则返回 data not exist    

    没有 Delete，前面说过，memtable 的 delete 是 lazy 的，实际上是 add 一条 `ValueType` 为 `kTypeDeletion` 的记录。

Reference
---
[Leveldb 源码分析 --1](http://blog.csdn.net/sparkliang/article/details/8567602)

[LevelDB : MemTable](http://mingxinglai.com/cn/2013/01/leveldb-memtable/)

[leveldb skiplist 实现分析](http://luodw.cc/2015/10/16/leveldb-05/)

[MemTable 与 SkipList-leveldb 源码剖析 (3)](http://www.pandademo.com/2016/03/memtable-and-skiplist-leveldb-source-dissect-3/)
    
    
