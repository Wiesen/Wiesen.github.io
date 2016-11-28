+++
date = "2016-11-28T21:31:20+08:00"
draft = false
title = "Leveldb: Introduction"
topics = ["Database", "Leveldb"]
+++

Introduction
---

Leveldb 库提供持久层 kv 存储，其中 keys 和values 可以是任意字节数组。目前有C++，golang 的实现。

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

Overall Architecture
---

![architecture](http://7vij5d.com1.z0.glb.clouddn.com/leveldb-architecture.png)

Leveldb 存储主要分为 SSTable 和 MemTable，前者为不可变且存储于持久设备上，后者位于内存上并且可变。其中有两个 MemTable，一个为当前写入 MemTable，另一个为等待持久化的 Immutable MemTable。此外还有一些辅助文件，后面详述。

**Write Operation**：当用户需要插入一条 kv 到 Leveldb 中时，首先会被存储到 log 中；然后插入到 MemTable 中；当 MemTable 达到一定大小后会转化为 read-only Immutable MemTable，并且创建一个新的 MemTable；同时开启一个新的后台线程将 Immutable MemTable 的内容 dump 到磁盘中，创建一个新的 SSTable；

**Deletion Operation**：在 Leveldb 中视为特殊的 Write operation，写入一个 deletion marker。

**Read Operation**：当 leveldb 收到一个 Get 请求时，首先会在 MemTable 进行查找；然后在 Immutable MemTable 查找；最后在 SSTable 中查找（从 level 0 到 higher level），直到匹配到一个 kv item。


Files
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
    |      +--- memenv  <=== Env的一个具体实现(Env是leveldb封装的运行环境)
    |
    +--- table          <=== 磁盘数据结构
    |
    +--- db             <=== db的所有实现
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
