+++
date = "2017-01-27T21:31:20+08:00"
draft = false
title = "Leveldb: Log"
tags = ["Database", "Leveldb"]
+++

Leveldb 存储主要分为 SSTable（磁盘） 和 MemTable（内存，包括 MemTable 和 Immutable MemTable），此外还有一些辅助文件:Manifest, Log, Current。

本篇 blog 主要分析 Log：log 在 Leveldb 中的作用，一条log 在内存中的形式是什么，以怎样的方式被组织。

## Log (`db/log_format.h&cc`,`db/log_writer.h&.cc`,`db/log_reader.h&.cc`)

![log commit](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_log_commit.png)

### Introduction

日志（logging）分两种：

1. 诊断日志（diagnostic log）：即log4j、glog、log4cxx等常用日志库提供的日志功能
2. 交易日志（transaction log）：即**数据库的 write-ahead log**、文件系统的journaling等，用于记录状态变更，通过 replay log 可以逐步恢复每一次修改之后的状态。

LevelDb 中的 log 是后者，主要作用是系统故障时进行数据恢复。**利用 WAL（write-ahead log），当故障时 Memtable 中的数据没来得及 Dump 成 SSTable 文件，LevelDB 可根据 log 文件恢复内存数据，保持数据（data和metadata）的持久性。

### [Format](http://www.cnblogs.com/cobbliu/p/6243550.html)

![log format](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_log.png)

log record 以 block 为单位组织（32k）。写日志时，一致性考虑，并没有以 block 为单位写，而是每次更新均对 log 文件进行 IO，根据 WriteOption::sync 决定是否做强制 sync。读取时以 block 为单位做 IO 以及校验。log 的写入是顺序写，读取只会在启动时发生，不会是性能的瓶颈。

Log  entry header 包含三个字段，checksum 是对“类型”和“数据”字段的校验码，为了避免处理不完整或者是被破坏的数据，当 leveldb 读取记录数据时候会对数据进行校验，如果 checksum 相同则说明数据完整无破坏，可以继续后续流程。length 表示 payload 长度，type 字段则指出了每条 log record 和 log block 之间的结构关系，主要有五种类型：ZERO/FULL/FIRST/MIDDLE/LAST，具体结构关系如上图所示。

![log record payload](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_log_payload.png)

上图为 log record payload 的具体结构，主要为 PUT_OP 和 DELETE_OP 两种操作，读操作不写 log。

Leveldb 采用 write batch 来优化并发写。对每一个写操作，先通过传入的键值对构造一个 WriteBatch 对象。多个并发写的 write batch 最后会被合并成一个，再写入 log 和 memtable。

### Declaration

![log reader](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_log_reader.png)

![log writer](http://7vij5d.com1.z0.glb.clouddn.com/leveldb_log_writer.png)

### Analysis

1. 写入 （`Writer::AddRecord() log_writer.cc`）

    对 log 的每次写入作为 record 添加：

    1. 如果当前 block 剩余的 size 小于 record 头长度，填充 trailer，开始下一个 block。
    2. 根据当前 block 剩余的 size 和写入 size，划分出满足写入的最大 record，确定 record type。
    3. 写入 record（Writer::EmitPhysicalRecord()）
    4. 循环1-3,直至写入处理完成
    5. 根据 option 指定的 sync 决定是否做 log 文件的强制 sync

2. 读取（`Reader::ReadRecord() db/log_reader.cc`）

    log 的读取仅发生在 db 启动的时候，每次读取出当时写入的一次完整更新：

    1. 第一次读取， 根据指定的 `initial_offset`_跳过 log 文件开始的 init_data(`Reader::SkipToInitialBlock()`)，如果从跳过的 offset 开始，当前 block 剩余的 size 小于 record 的头长度（是个 trailer），则直接跳过这个 block。当前实现中指定的initial_offset_为 0。
    2. 从 log 文件中读一个 record（`Reader::ReadPhysicalRecord()`)
    3. 根据读到 record 的 type 做进一步处理
    4. 循环 1-3 直至读取出当时写入的一个完整更新。

### Reference

1. [leveldb源码分析之读写log文件](http://luodw.cc/2015/10/18/leveldb-08/)
2. [LevelDB源码剖析之Env与log::Writer](http://mingxinglai.com/cn/2013/01/leveldb-log-and-env/) 
3. leveldb实现解析 by 淘宝-核心系统研发-存储 那岩
4. [LevelDB: Read the Fucking Source Code](http://www.grakra.com/2017/06/17/Leveldb-RTFSC/) 


