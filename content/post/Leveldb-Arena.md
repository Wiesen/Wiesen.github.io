+++
date = "2017-01-20T21:31:20+08:00"
draft = true
title = "Leveldb: Memable"
tags = ["Database", "Leveldb"]
+++



Arena：memtable 内存管理
​	

1. Introduction

  Leveldb 中使用 Arena 对 memtable 进行简单的内存管理，主要出于以下三个目的：

  1. 封装内存操作以便进行内存使用统计：memtable 有阈值限制（write buffer size），通过统一的接口来分配不同大小的内存；
  2. 保证内存对齐：Arena 每次按 klockSize(`static const int kBlockSize = 4096;`)单位向系统申请内存，提供地址对齐内存，方便记录内存的使用并且提高内存使用效率；
  3. 解决频繁分配小块内存降低效率，直接分配大块内存浪费内存的问题，同时避免个别大内存使用影响：当memtable申请内存时，若 `size <= kBlockSize / 4` 则在当前的内存block中分配，否则直接向系统申请（new）。

  Arena 类内存管理方法：

  1. 采用 vector 来存储每次分配的内存，每一次分配的内存为一个块（block），block默认大小为4096kb；
  2. 当达到阈值时将 dump to disk，完成后由析构函数完成所有内存的一次性释放，由于业务的特殊性因此无需提供内存释放的接口

  ![arena](http://7vij5d.com1.z0.glb.clouddn.com/arena.png))

2. Declaration

  	class Arena {
  	public:
  	  Arena();
  	  ~Arena();
  	
  	  // Return a pointer to a newly allocated memory block of "bytes" bytes.
  	  char* Allocate(size_t bytes);
  	
  	  // Allocate memory with the normal alignment guarantees provided by malloc
  	  char* AllocateAligned(size_t bytes);
  	
  	  // Returns an estimate of the total memory usage of data allocated
  	  // by the arena (including space allocated but not yet used for user
  	  // allocations).
  	  size_t MemoryUsage() const {
  	    return blocks_memory_ + blocks_.capacity() * sizeof(char*);
  	  }
  	
  	 private:
  	  char* AllocateFallback(size_t bytes);
  	  char* AllocateNewBlock(size_t block_bytes);
  	
  	  // Allocation state
  	  char* alloc_ptr_;
  	  size_t alloc_bytes_remaining_;
  	
  	  // Array of new[] allocated memory blocks
  	  std::vector<char*> blocks_;
  	
  	  // Bytes of memory in blocks allocated so far
  	  size_t blocks_memory_;
  	
  	  // No copying allowed
  	  Arena(const Arena&);
  	  void operator=(const Arena&);
  	};

3. Analysis

  成员变量：

  1. `char* alloc_ptr_;`				// 内存的偏移量指针，即指向未使用内存的首地址
    2. `size_t alloc_bytes_remaining_;`// 还剩下的内存数
    3. `std::vector<char*> blocks_;`// 关键：存储每一次分配的内存指针
      4. `size_t blocks_memory_;`	// 到目前为止分配的总内存

  共有函数（接口）

  1. `inline char* Arena::Allocate(size_t bytes)`：

     ```c++

     inline char* Arena::Allocate(size_t bytes) {
       // The semantics of what to return are a bit messy if we allow
       // 0-byte allocations, so we disallow them here (we don't need
       // them for our internal use).
       assert(bytes > 0);
       if (bytes <= alloc_bytes_remaining_) {
         char* result = alloc_ptr_;
         alloc_ptr_ += bytes;
         alloc_bytes_remaining_ -= bytes;
         return result;
       }
       return AllocateFallback(bytes);
     }

     char* Arena::AllocateFallback(size_t bytes) {
       if (bytes > kBlockSize / 4) {
         // Object is more than a quarter of our block size.  Allocate it separately
         // to avoid wasting too much space in leftover bytes.
         char* result = AllocateNewBlock(bytes);
         return result;
       }

       // We waste the remaining space in the current block.
       alloc_ptr_ = AllocateNewBlock(kBlockSize);
       alloc_bytes_remaining_ = kBlockSize;

       char* result = alloc_ptr_;
       alloc_ptr_ += bytes;
       alloc_bytes_remaining_ -= bytes;
       return result;
     }
     ```

     1. 该函数返回bytes字节的内存
     2. 如果预先分配的内存(alloc_bytes_remaining_)大于bytes,也就是说，预先分配的内存能够满足现在的需求，那就返回预先分配的内存，而不用向系统申请
     3. 如果(alloc_bytes_remaining_ < bytes)，也就是说，预先分配的内存不能不够用，那就调用AllocateFallback来分配内存（分`size <= kBlockSize / 4`进行区别处理）

  2. `char* Arena::AllocateAligned(size_t bytes)`：

     	char* Arena::AllocateAligned(size_t bytes) {  const int align = sizeof(void*);    // We'll align to pointer size
     	  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
     	  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);
     	  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
     	  size_t needed = bytes + slop;
     	  char* result;
     	  if (needed <= alloc_bytes_remaining_) {
     	    result = alloc_ptr_ + slop;
     	    alloc_ptr_ += needed;
     	    alloc_bytes_remaining_ -= needed;
     	  } else {
     	    // AllocateFallback always returned aligned memory
     	    result = AllocateFallback(bytes);
     	  }
     	  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0);
     	  return result;
     	}

     1. 首先获取一个指针的大小`const int align = sizeof(void*)`
     2. 然后将我们使用的char * 指针地址转换为一个无符号整型(reinterpret_cast<uintptr_t>(result):It is an unsigned int that is guaranteed to be the same size as a pointer.)
     3. 通过与操作来获取`size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1)`;
     4. 当前指针模4的值，有了这个值以后，我们就容易知道，还差 `slop = align - current_mod` 个字节内存才能对齐，即`result = alloc_ptr + slop`

  3. `size_t MemoryUsage() const`：仅仅是计算内存使用情况，不赘述

  ctor & dtor：

  	Arena::Arena() {
  	  blocks_memory_ = 0;
  	  alloc_ptr_ = NULL;  // First allocation will allocate a block
  	  alloc_bytes_remaining_ = 0;
  	}
  	
  	Arena::~Arena() {
  	  for (size_t i = 0; i < blocks_.size(); i++) {
  	    delete[] blocks_[i];
  	  }
  	}

  1. `Arena::Arena()`：初始化各个成员变量
  2. `Arena::~Arena()`：释放各个 vector block 的内存

