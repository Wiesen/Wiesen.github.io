+++
date = "2016-11-28T21:31:20+08:00"
draft = false
title = "Leveldb: Introduction"
topics = ["Database", "Leveldb"]
+++

看了不少 blog 分析 leveldb，但很少看到有人从设计原因和策略上进行总结。所以这个系列对 leveldb 的实现做一点设计分析，争取将内部实现逻辑串联起来，至于源码注释之类的网上一扒拉就有很多啦。

Introduction
---

Leveldb 库提供持久层 kv 存储，其中 keys 和values 可以是任意字节数组。目前有 C++，golang 的实现。

作者 Jeff Dean, Sanjay Ghemawat 同时也是设计实现 BigTable 的作者。在 BigTable 中有两个关键部分：master server 和 tablet server。

- master server 负责存储 meta-data，并且调度管理 tablet server；
- tablet server 负责存储具体数据，并且响应读写操作。

Leveldb 可视为 BigTable 中 tablet server 的简化实现。

Features, Limitations, Performance
---

详见 [leveldb homepage](https://github.com/google/leveldb)

How to use
---

详见 [leveldb 使用文档](https://github.com/google/leveldb/blob/master/doc/index.html)

Background
---
冯诺依曼体系结构的计算机系统主要为两点：**存储 + 计算**。数据库即为存储方面，依赖于存储硬件特性。

当前磁盘物理结构特性导致 (磁头寻道，旋转延迟)：**随机读写慢，连续读写快**，相差三个数量级。内存和 SSD 同样表现，只不过原因是：连续读写可预判，因此会被优化 (相差量级也没磁盘那么大)。

上述说明：基于目前这样的硬件特性，我们设计存储系统时要**尽量避免随机读写，设计为顺序读写**。

然而，顺序读和顺序写是相互矛盾的：

- 为了优化读效率，最好是将以有序方式组织数据从而写入相邻的块，以维护顺序读，而这增加了随机写；
- 为了优化写效率，最好是所有的写都是增量写，也就是顺序写，而这增加了随机读。

题外话：常见的优化读操作性能的设计有：二分、hash、B+ 树等。这些方法使用各种查找结构来组织数据，有效提升读操作性能(最少提供了O(logN))，但是增加了随机写操作，大大降低了写操作性能。

因此，如果一个业务**对写操作的吞吐量十分敏感，或者写操作数量远远大于读操作**，那么应该采取的措施是：**优化写操作，也就是尽量将写操作设计为增量写**，从而尽可能地减少寻道，以达到磁盘理论写入速度最大值。

这个策略常用于日志或者堆文件这种业务场景中，因为它们是完全顺序的（包括读写），所以可以提供很好的写操作性能。

Leveldb Core Idea
---

以 **LSM(Log Structured Merge) Tree**组织数据从而将逻辑场景中的写请求转换为顺序写，辅以 **Bloom Filter** 和 **Shard LRU cache** 等策略优化随机读操作从而保证读效率。

1. [**LSM tree**](https://en.wikipedia.org/wiki/Log-structured_merge-tree)
    
    关于论文的部分分析详见[链接](https://blog.acolyer.org/2014/11/26/the-log-structured-merge-tree-lsm-tree/)
    
    <div style="text-align: center">
    <img src="http://7vij5d.com1.z0.glb.clouddn.com/memtable.png" width="500"/> <img src="http://7vij5d.com1.z0.glb.clouddn.com/compaction.png" width="500"/>
    </div>

    LSM tree 针对**写入速度瓶颈问题**而提出的方法，其基本思想是：**将随机写转换为顺序写，交换读和写的随机 IO**。主要组成结构为：MemTable（内存中）+ SSTable (Sorted String Table)

    首先数据更新增量写入 MemTable，其后保存为有序文件 SSTable 到磁盘中（read only），每个文件保存一段时间内的数据更新。为了均衡读写效率，SSTable 文件是一种分层次（level）管理。

    LSM Tree 的主要措施有:

    - 对更新进行批量 & 延时处理 (超过大小阈值后将内存中的数据 dump 到磁盘中)：减少寻道，提高写操作性能；
    - 周期性地利用归并排序对磁盘文件执行合并操作 (compaction)：移除已删除和冗余数据，减少文件个数，保证读操作性能；

2. [**Bloom Filter**](https://en.wikipedia.org/wiki/Bloom_filter)
    
    <div style="text-align: center">
    <img src="http://7vij5d.com1.z0.glb.clouddn.com/bloomfilter.png" width="550"/>
    </div>
    
    由于 LSM Tree 会产生大量文件，因此 LevelDb 利用 bloomfilter 来避免大量的读操作。

    bloomfilter 以概率性算法高效检索一个 key 是否在一个 SSTable (核心为 hash，若判断为不存在则必定不存在，若判断为存在则可能不存在)。

3. [**Shard LRU Cache**](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_Recently_Used_.28LRU.29)

    leveldb 中的 cache 分为 Table Cache 和 Block Cache 两种，其中 Table Cache 中缓存的是 sstable 的索引数据，Block Cache 缓存的是 Block 数据（可选打开）
        
    由于 levelDB 为多线程，每个线程访问缓冲区时都会对缓冲区加锁。为了保证多线程安全并且减少锁开销，leveldb 定义了一个 SharedLRUCache。

    ShardedLRUCache 内部有 16 个 LRUCache，查找 Key 时根据 key 的高四位进行 hash 索引，然后在相应的 LRUCache 中进行查找或更新。当 LRUCache 使用率大于总容量后, 根据 LRU 淘汰 key.

Overall Architecture
---

<div style="text-align: center">
<img src="http://7vij5d.com1.z0.glb.clouddn.com/leveldb-architecture.png" width="500"/>
</div>

Leveldb 存储主要分为 SSTable 和 MemTable（即 LSM Tree），前者为不可变且存储于持久设备上，后者位于内存上并且可变。其中有两个 MemTable，一个为当前写入 MemTable，另一个为等待持久化的 Immutable MemTable。此外还有一些辅助文件，后面详述。

**Write Operation**：当用户需要插入一条 kv 到 Leveldb 中时，首先会被存储到 log 中 **(Write Ahead Log, WAL, 保证数据持久性)**；然后插入到 MemTable 中；当 MemTable 达到一定大小后会转化为 read-only Immutable MemTable，并且创建一个新的 MemTable；同时开启一个新的后台线程将 Immutable MemTable 的内容 dump 到磁盘中，创建一个新的 SSTable；

**Deletion Operation**：在 Leveldb 中视为特殊的 Write operation，写入一个 deletion marker。

**Read Operation**：当 leveldb 收到一个 Get 请求时，首先会在 MemTable 进行查找；然后在 Immutable MemTable 查找；最后在 SSTable 中查找（从 level 0 到 higher level），直到匹配到一个 kv item 或者为 NULL。


File Layout
---

详见 [leveldb 实现文档](https://github.com/google/leveldb/blob/master/doc/impl.html)


Source Code Structure
---
    leveldb-1.4.0  
    |
    +--- port           <=== 提供各个平台的基本接口
    |
    +--- util           <=== 提供一些通用工具类
    |
    +--- helpers
    |      |
    |      +--- memenv  <=== Env 的一个具体实现(Env 是 leveldb 封装的运行环境)
    |
    +--- table          <=== sstable 相关的数据格式定义和操作实现
    |
    +--- db             <=== 主要逻辑的实现
    |
    +--- doc
    |     |
    |     +--- table_format.txt   <=== 磁盘文件数据结构说明
    |     |
    |     +--- log_format.txt     <=== 日志文件（用于宕机恢复未刷盘的数据）数据结构说明
    |     |
    |     +--- impl.html          <=== 一些实现
    |     |
    |     +--- index.html         <=== 使用说明
    |     |
    |     +--- bench.html         <=== 测试数据
    |
    +--- include
         |
         +--- leveldb           <=== 所有头文件

----------
Reference：

> [LevelDB 之 LSM-Tree](http://zouzls.github.io/2016/11/23/LevelDB%E4%B9%8BLSM-Tree/)

> [和我一起学习 leveldb](http://brg-liuwei.github.io/tech/2014/10/15/leveldb-0.html)

> [Log Structured Merge Trees(LSM) 原理](http://www.open-open.com/lib/view/open1424916275249.html)