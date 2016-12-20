+++
date = "2016-12-15T21:31:20+08:00"
draft = true
title = "Leveldb: Storage SSTable"
topics = ["Database", "Leveldb"]
+++

Overall Architecture
---

<div style="text-align: center">
<img src="http://7vij5d.com1.z0.glb.clouddn.com/leveldb-architecture.png" width="500"/>
</div>

Leveldb 存储主要分为 SSTable 和 MemTable（MemTable && Immutable MemTable，此外还有一些辅助文件:Manifest, Log, Current。

SSTable
---

sstable 全名 sort-string table, bigtable 使用的存储技术. 顾名思义, sstable 中的数据都是有序的.
除了日志之外, leveldb 的数据统统存储在 sstable 中. 对于 sstable, 我们先来看下其布局.

每个 sstable 内部按 block 划分. 物理布局如图.

然后可以看看 Table Format 文档关于 table 存储格式。table 存储格式里面主要包括几个部分：

data block
meta block
meta index block
data index block
footer
footer 部分是放在最末尾的，里面包含了 data index block 以及 meta index block 的偏移信息，读取 table 时候从末尾读取。

首先我们看看 data block 是如何组织的。对于 DiskTable(TableBuilder) 就是不断地 Add(Key,Value). 当缓存的数据达到一定大小之后， 就会调用 Flush 这样就形成了一个 Block. 对于一个 Block 内部而言的话，有个很重要的概念就是 restart point. 所谓 restart point 就是为了解决 前缀压缩的问题的，所谓的 restart point 就是基准 key。假设我们顺序加入 abcd,abce,abcf. 我们以 abcd 为 restart point 的话，那么 abce 可以存储为 (3,e),abcf 存储为 (3,f). 对于 restart point 采用全量存储，而对于之后的部分采用增量存储。一个 restart block 可能存在多个 restart point, 将这些 restart point 在整个 table offset 记录下来，然后放在 data block 最后面。每个 data block 尾部还有一个 type 和 CRC32. 其中 type 可以选择是否 需要针对这个 data block 进行 snappy 压缩，而 CRC32 是针对这个 data block 的校验。

data index block 组织形式和 data block 非常类似，只不过有两个不同。1)data index block 从不刷新直到 Table 构造完成之后才会刷新，所以 对于一个 table 而言的话只有一个 data index block.2)data index block 添加的 key/value 是在 data block 形成的时候添加的，添加 key 非常取巧 ，是上一个 data block 和这个 data block 的一个 key seperator. 比如上一个 data block 的 max key 是 abcd, 而这个 data block 的 min key 是 ad. 那么这个 seperator 可以设置成为 ac.seperator 的生成可以参考 Comparator. 使用尽量短的 seperator 可以减小磁盘开销并且提高效率。而对于添加的 value 就是 这个 data block 的 offset. 同样在 data index block 也会存在 restart point.

然后看看进行一个 key 的 query 是如何进行的。首先读取出 data index block(这个部分可以常驻内存)，得到里面的 restart point 部分。针对 restart point 进行二分。因为 restart point 指向的 key 都是全量的 key. 如果确定在某两个 restart point 之间之后，就可以遍历这个 restart point 之间范围分析 seperator. 得到想要查找的 seperator 之后对应的 value 就是某个 data block offset. 读取这个 data block 和之前的方法一样就可以查找 key 了。对于遍历来说，过程是一样的。

这里我们稍微分析一下这样的工作方式的优缺点。对于写或者是 merge 来说的话，效率相当的高，所有写都是顺序写并且还可以进行压缩。影响写效率的话一个重要参数就是 flush block 的参数。 但是对于读来说的话，个人觉得过程有点麻烦，但是可以实现得高效率。对于 flush block 调节会影响到 data index block 和 data block 占用内存大小。如果 flush block 过大的话， 那么会造成 data index block 耗费内存小，但是每次读取出一个 data block 内存很大。如果 flush block 过小的话，那么 data index block 耗费内存很大，但是每次读取 data block 内存很小。 而 restart point 数量会影响过多的话，那么可能会占用稍微大一些的内存空间，但是会使得查找过程更快 (遍历数更少).



