+++
date = "2017-01-31T21:31:20+08:00"
draft = false
title = "Leveldb: Storage MemTable"
tags = ["Database", "Leveldb"]
+++

Leveldb 存储主要分为 SSTable（磁盘） 和 MemTable（内存，包括 MemTable 和 Immutable MemTable），此外还有一些辅助文件:Manifest, Log, Current。

本篇 blog 主要分析 Memtable：Leveldb一条记录在内存中的形式是什么，记录以怎样的方式被组织。

Memtable (`db/skiplist.h`,`db/memtable.h & memtable.cc`)
---

<div style="text-align: center">
<img src="http://7vij5d.com1.z0.glb.clouddn.com/leveldb-architecture.png" width="500"/>
</div>

### Introduction

MemTable 是 leveldb 的 kv 数据在内存中的存储结构。当 Memtable 写入的数据占用内存到达指定大小 (Options.write_buffer_size)，则自动转换为 Immutable Memtable（只读），同时生成新的 Memtable 供写操作写入新数据。后台的 compact 进程会负责将 immutable memtable dump to disk 生成 sstable。

通过 Memtable 和 Immutable Memtable，Leveldb 可在持久化到磁盘上的同时保持对外服务可用。这种特性是由 Leveldb 的适用场景催生的：Append 写。

### Structure & Operation

![leveldb memtable key](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_memtable_key.png)

memtable key 的组成参照 **Leveldb: Basic Settings**。memtable 存储同一 key 的多个版本的数据。KeyComparator 首先按照递增顺序比较 user key，然后安装递减顺序比较sequence number，这两个足以唯一确定一条 entry。

MemTable 提供内存中 KV entry 的 Add 和 Get 操作接口。 Memtable 实际上并不存在 Delete 操作，删除某个 Key 的 Value 在 Memtable 内是作为插入一条记录实施的，但是会打上一个 Key 的删除标记，真正的删除操作是 Lazy 的，会在以后的 Compaction 过程中删除这个 KV。

Memtable 类只是一个接口类，其底层实现依赖于两大核心组件 Arena 和 SkipList。其中 Arena 内存分配器统一管理内存，SkipList 用于实际 KV 存储（Memtable 的所有数据操作接口都是对 SkipList 的封装）。

### Component

1. Arena： memtable 内存管理
   1. 封装内存操作以便进行内存使用统计：memtable 有阈值限制（write buffer size），通过统一的接口来分配不同大小的内存；

   2. 保证内存对齐：Arena 每次按 klockSize(`static const int kBlockSize = 4096;`)单位向系统申请内存，提供地址对齐内存，方便记录内存的使用并且提高内存使用效率；

   3. 解决频繁分配小块内存降低效率，直接分配大块内存浪费内存的问题，同时避免个别大内存使用影响：当memtable申请内存时，若 `size <= kBlockSize / 4` 则在当前的内存block中分配，否则直接向系统申请（new）；

   4. 由Arena的析构函数统一释放所有的内存：和 leveldb 特定的应用场景相关的，比如一个memtable使用一个Arena，当memtable被释放时，由Arena统一释放其内存，不需要额外提供内存释放接口。

   5. Arena 类内存管理方法：

     1. 采用 vector 来存储每次分配的内存，每一次分配的内存为一个块（block），block默认大小为4096kb；

     2. 当达到阈值时将 dump to disk，完成后由析构函数完成所有内存的一次性释放，由于业务的特殊性因此无需提供内存释放的接口

        ![arena](http://7vij5d.com1.z0.glb.clouddn.com/arena.png))

2. SkipList：memtable 的实际数据结构

   1. 根据上一篇 blog 中 LSM tree 的性质，table 应当保持有序性。而对一个排序结构执行插入操作开销很大（随机写），通常性能瓶颈集中在这里。一些数据结构如链表, AVL 树, B 树, skiplist 等都加速了随机写。leveldb 的 memtable 实现没有使用复杂的 B 树，采用更轻量级的 [skiplist](https://en.wikipedia.org/wiki/Skip_list)。
   2. skiplist 是一种可以代替平衡树的数据结构：skiplist 从概率上保证数据平衡（依赖于系统设计时的随机假设），结构和实现比平衡树简单。概率上时间复杂度近似 O(logN)，与平衡树相同，但空间上比较节省，一个节点平均只需要 1.333 个指针。
   3. skiplist 拥有比平衡树更好的并发性能：平衡树在更新时可能会触发 rebalancing，改操作牵涉到大量节点，争锁的代价相对较高。而 skiplist 的操作需要更新的部分比较少，加锁的粒度和数量都更少，所以不同线程争锁的代价相对较低
   4. Leveldb 中，Skiplist 中的操作不需要任何锁或者 node 的引用计数，利用内存屏障来进行线程同步，原因主要为以下两个：
      1. Skiplist 中 node 内保存的是 InternalKey 与对应 value 组成的数据，SequenceNumber 的全局唯一保证了不会有相同的 node 出现，因此保证了不会有node更新的情况.
      2. delete 操作等同于 put 操作，因此不会需要引用计数记录 node 的生存周期。

### Declaration

![memtable](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_memtable.png)

![skiplist](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_skiplist.png)

### Coding & Function Analysis

构造和析构：MemTable 的对象构造必须显式调用，重点在于初始 Arena 和 SkipList 两大核心组件。MemTable class 由 `Unref()` 完成实际的内存释放操作，因此 Leveldb 中禁用了 copy ctor 和 assignment operator（C++ class基本功，正确管理内存和其他资源）。

操作：memtable 对 key 的查找和遍历是 MemTableIterator，而 MemTableIterator 实际上是 SkipList iterator wrapper。NewIterator 即是返回一个 MemTableIterator，用于有序遍历 memtable 的存储数据。

1. 写入： `void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value)`:
  1. Encode：将传入的参数封装为 `Internalkey`，然后与 `value` 一起编码成前述（Leveldb setting） entry
  2. Insert 到实际数据结构：`SkipList::Insert()`

2. 读取： `bool Get(const LookupKey& key, std::string* value, Status* s)`:
  1. 从传入的 `LookupKey` 中获取 `memtable_key`
  2. `MemTableIterator::Seek()` 返回 `MemTableIterator`
  3. 对 `MemTableIterator` 的 key 进行还原，并对 key 后面的8个字节解码，以此判断消息是什么类型
  4. a) `kTypeValue` 表有效数据，返回对应的 `value` 数据; b) `kTypeDeletion` 表无效数据，设置 Status 为 NotFound 

3. 没有删除：前面说过，memtable 的 delete 是 lazy 的，实际上是 add 一条 `ValueType` 为 `kTypeDeletion` 的记录。


### Reference

1. [Leveldb 源码分析 --1](http://blog.csdn.net/sparkliang/article/details/8567602)
2. [LevelDB : MemTable](http://mingxinglai.com/cn/2013/01/leveldb-memtable/)
3. [leveldb skiplist 实现分析](http://luodw.cc/2015/10/16/leveldb-05/)
4. [MemTable 与 SkipList-leveldb 源码剖析 (3)](http://www.pandademo.com/2016/03/memtable-and-skiplist-leveldb-source-dissect-3/)
5. [LevelDB: Read the Fucking Source Code](http://www.grakra.com/2017/06/17/Leveldb-RTFSC/) 

