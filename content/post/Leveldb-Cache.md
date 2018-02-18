+++
date = "2017-02-25T21:31:20+08:00"
draft = false
title = "Leveldb: Block Cache"
tags = ["Database", "Leveldb"]
+++

Block Cache (`util/cache.cc`)：
	
1. Introduction

	Cache 的目的：减少磁盘IO，加快 CURD 速度。

    leveldb 中的 cache 分为 Table Cache 和 Block Cache 两种，其中 Table Cache 中缓存的是 sstable 的索引数据，Block Cache 缓存的是 Block 数据（可选打开）。
    
	leveldb 中支持用户自己实现 block cache 逻辑，作为 option 传入。默认使用的是内部实现的 LRU。

	1. 基于简单以及效率考虑，leveldb 中实现了一个简单的 hash table（LRUHandle），采用定长数组存放 node，链表解决 hash 冲突。每次 insert 后，如果 node 数量大于数组的容量（期望短的冲突链表长度），就将容量扩大2倍，做一次 rehash；
	2. LRU 的逻辑由 LRUCache 控制， insert 和 lookup 时更新链表即可；
	3. 由于 levelDB 为多线程，每个线程访问 cache 时都会对 cache 加锁。为了保证多线程安全并且减少锁开销，又将 LRUCache 再做 shard（ShardedLRUCache）。

	整体来看：ShardedLRUCache 内部有 16 个 LRUCache（定长），查找 Key 时根据 key 的高四位进行 hash 索引，然后在相应的 LRUCache 中进行查找或更新。当 LRUCache 使用率大于总容量后, 根据 LRU 淘汰 key.

	![cache](http://7vij5d.com1.z0.glb.clouddn.com/leveldb-cache.jpg)

2. Declaration

		// A single shard of sharded cache.
		class LRUCache {
		 public:
		  LRUCache();
		  ~LRUCache();
		
		  // Separate from constructor so caller can easily make an array of LRUCache
		  void SetCapacity(size_t capacity) { capacity_ = capacity; }
		
		  // Like Cache methods, but with an extra "hash" parameter.
		  Cache::Handle* Insert(const Slice& key, uint32_t hash,
		                        void* value, size_t charge,
		                        void (*deleter)(const Slice& key, void* value));
		  Cache::Handle* Lookup(const Slice& key, uint32_t hash);
		  void Release(Cache::Handle* handle);
		  void Erase(const Slice& key, uint32_t hash);
		
		 private:
		  void LRU_Remove(LRUHandle* e);
		  void LRU_Append(LRUHandle* e);
		  void Unref(LRUHandle* e);
		
		  // Initialized before use.
		  size_t capacity_;
		
		  // mutex_ protects the following state.
		  port::Mutex mutex_;
		  size_t usage_;
		  uint64_t last_id_;
		
		  // Dummy head of LRU list.
		  // lru.prev is newest entry, lru.next is oldest entry.
		  LRUHandle lru_;
		  HandleTable table_;
		};

		struct LRUHandle {
		  void* value;//存储键值对的值
		  void (*deleter)(const Slice&, void* value);//这个结构体的清除函数，由外界传进去注册
		  LRUHandle* next_hash;//用于hash表冲突时使用
		  LRUHandle* next;//当前节点的下一个节点
		  LRUHandle* prev;//当前节点的上一个节点
		  size_t charge;      // 这个节点占用的内存
		  size_t key_length;//这个节点键值的长度
		  uint32_t refs;//这个节点引用次数，当引用次数为0时，即可删除
		  uint32_t hash;      // 这个键值的哈希值
		  char key_data[1];   // 存储键的字符串，也是C++柔性数组的概念。
		  Slice key() const {
		    // For cheaper lookups, we allow a temporary Handle object
		    // to store a pointer to a key in "value".
		    if (next == this) {
		      return *(reinterpret_cast<Slice*>(value));
		    } else {
		      return Slice(key_data, key_length);
		    }
		  }//为了加速查询，有时候一个节点在value存储键。
		};
		
		class HandleTable {
		 public:
		  HandleTable() : length_(0), elems_(0), list_(NULL) { Resize(); }
		  ~HandleTable() { delete[] list_; }
		
		  LRUHandle* Lookup(const Slice& key, uint32_t hash) {}
		  LRUHandle* Insert(LRUHandle* h) {}
		  LRUHandle* Remove(const Slice& key, uint32_t hash) {}
		
		 private:
		  // The table consists of an array of buckets where each bucket is
		  // a linked list of cache entries that hash into the bucket.
		  uint32_t length_;//哈希数组的长度
		  uint32_t elems_;//哈希表存储元素的数量
		  LRUHandle** list_;//哈希数组指针，因为数组存储的是指针，所以类型必须是指针的指针

		
		  // Return a pointer to slot that points to a cache entry that
		  // matches key/hash.  If there is no such cache entry, return a
		  // pointer to the trailing slot in the corresponding linked list.
		  LRUHandle** FindPointer(const Slice& key, uint32_t hash) {}
		  void Resize() {}
		};	
		
	

3. Analysis

	Cache类：一个抽象类，规范外部接口，用户可定义自己的 block cache，默认是调用 `NewLRUCache` 返回一个 SharedLRUCache 对象。

		Cache* NewLRUCache(size_t capacity) {
		  return new SharedLRUCache(capacity);
		}

	SharedLRUCache类：内部有16个（`kNumShards`） LRUCache 缓冲区，该类主要作用仅是计算 hash 值，选择 LRUCache ，然后调用 LRUCache 的函数。

	LRUCache 类：真正的缓冲区数据结构，每个缓冲区都维护了一个 LRU 双向链表和 hash 表。链表的元素类型为 LRUHandle；hash 表中的元素是 `LRUHandle *`，用于快速判断数据是否在缓冲区中。

	LRUHandle 类：cache 中负责存储 key/value 的数据结构，是LRU 双向链表中的元素类型。`char key_data[1];`为存储 key 的字符串，也是C++柔性数组的概念。`void* value;`为存储 value
	
	HandleTable 类：为了保证哈希链的查找速度，尽量使平均哈希链长度为<=1。当元素的个数大于桶的数量时就重新 hash（resize 为当前 hash 表长度的2倍并 rehash），保证 hash 的时间复杂度为 O(1)。


Reference
---

1. [LevelDB源码剖析之Cache缓冲区与hash表](http://mingxinglai.com/cn/2013/01/leveldb-cache/)
2. [leveldb源码分析之Cache](http://luodw.cc/2015/10/24/leveldb-11/)
3. [leveldb](https://dirtysalt.github.io/leveldb.html#orgheadline188)
 