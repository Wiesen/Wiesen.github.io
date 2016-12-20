+++
date = "2016-12-19T21:31:20+08:00"
draft = false
title = "Leveldb: Basic Settings"
topics = ["Database", "Leveldb"]
+++

（待完善……）

1. Slice(`include/leveldb/slice.h`)

    为操作数据的方便，将数据和长度包装成 Slice 使用，直接操控指针以避免不必要的数据拷贝

        class Slice {
            … 
            private: 
                const char* data_; 
                size_t size_; 
        };

2. Optin(`include/leveldb/option.h`)

    leveldb 中启动时的一些配置，通过 Option 传入，get/put/delete 时，也有相应的 ReadOption/WriteOption。

3. Env(`include/leveldb/env.h` `util/evn_posix.h`)

    考虑到移植以及灵活性，leveldb 将系统相关的处理（文件/进程/时间之类）抽象成 Env，用户可以自己实现相应的接口，作为 Option 传入。默认使用自带的实现。

4. varint(`util/coding.h`)

    leveldb 采用了 protocalbuffer 里使用的变长整形编码方法，节省空间。

5. ValueType(`db/dbformat.h`)

    leveldb更新（put/delete）某个key时不会操控到db中的数据，每次操作都是直接新插入一份kv数据，具体的数据合并和清除由后台的compact完成。

    所以，每次 put 都会添加一份 KV 数据，即使该 key 已经存在；而 delete 等同于 put 空的 value。为了区分 live 数据和已删除的 mock 数据，leveldb 使用 ValueType 来标识：

        enum ValueType { 
            kTypeDeletion = 0x0, 
            kTypeValue = 0x1 
        };

6. SequenceNumber(`db/dbformat.h`)

    leveldb中的每次更新（put/delete) 操作都拥有一个版本，由 SequnceNumber 来标识，整个db有一个全局值保存着当前使用到的SequnceNumber。SequnceNumber 在 leveldb 有重要的地位，key 的排序，compact 以及 snapshot 都依赖于它。
    
    `typedef uint64_t SequenceNumber`: 存储时，SequnceNumber 只占用56 bits, ValueType 占用8 bits，二者共同占用 64bits（uint64_t).

    |   0-56      |   56-64   |
    | ------------|:---------:|
    |SequnceNumber|  ValueType|

7. user_key & memtable_key & internal_key

    - user_key: 用户使用的 key
    - memtable_key: memtable 中使用的 key
    - internal_key: sstable 中使用的 key

8. InternalKey(`db/dbformat.h`) & ParsedInternalKey (`db/dbformat.h`)

    InternalKey: userkey ＋ 元信息（8 bytes, SequnceNumber|ValueType), ParsedInternalKey 为 InternalKey 分拆得到的结构体

        class InternalKey {
            …
            private:
                std::string rep_;
        }
        
        struct ParsedInternalKey { 
            Slice user_key; 
            SequenceNumber sequence; 
            ValueType type; 
        };

10. LookupKey(`db/dbformat.h`)

    db 内部为查找 memtable/sstable 方便而设置的类：
   
        class LookupKey { 
            … 
            private: 
                const char* start_;
                const char* kstart_;
                const char* end_;
        };
 
    |start_           |               kstart_ - end_                           |
    | ----------------|:------------------------------------------------------:|
    |internal_key_size|  internal_key: userkey_data + SequenceNumber|ValueType)|
    |    (varint32)   |          (InternalKey_size: char[] + uint64)           |

    对 memtable lookup 时使用 `memtable_key` [`start_`,`end_`], 对 sstable lookup 时使用 `internal_key` [`kstart_`, `end_`]。
    
    <img src="http://7vij5d.com1.z0.glb.clouddn.com/leveldb_key.png" width="450"/>

10. memtable entry

    MemTable 中 entry 存储格式（实际上是字符串）：

    <img src="http://7vij5d.com1.z0.glb.clouddn.com/leveldb_memtable_entry.png" width="600"/>

11. Comparator(`include/leveldb/comparator.h` `util/comparator.cc`)
12. InternalKeyComparator(`db/dbformat.h`)
13. WriteBatch(`db/write_batch.cc`)
14. FileMetaData(`db/version_edit.h`)
15. block(`table/block.cc`)
16. BlockHandle(`table/dbformat.h`)
17. FileNumber(`db/dbformat.h`)
18. filename(`db/dbformat.cc`)
19. Compact(`db/db_impl.cc` `db/version_set.cc`)
20. Iterator(`include/leveldb/iterator.h`)

Reference
---
leveldb实现解析 by 淘宝-核心系统研发-存储 那岩 

[MemTable 与 SkipList-leveldb 源码剖析 (3)](http://www.pandademo.com/2016/03/memtable-and-skiplist-leveldb-source-dissect-3/)
    