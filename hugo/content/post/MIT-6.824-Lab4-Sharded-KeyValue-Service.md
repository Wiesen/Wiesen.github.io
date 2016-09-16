+++
date = "2016-09-09T17:32:46+08:00"
draft = false
title = "MIT 6.824 Lab 4 Sharded KeyValue Service Implementation"

topics = ["Distributed System", "MIT 6.824"]
+++

lab4 是基于 lab2 和 lab3 实现的 Raft Consensus Algorithm 之上实现 Sharded KeyValue Service。主要分为两部分：

- Part A：The Shard Master
- Part B: Sharded Key/Value Server

除了最后一个 challenge test case `TestDelete()` 以外，目前代码其余都可以 pass。但偶尔会 fail 在 unreliable test case，目前的定位是 raft 的实现还有点 bug。

Code Link:

- [PART A](https://github.com/Wiesen/MIT-6.824/tree/master/2016/shardmaster)
- [PART B](https://github.com/Wiesen/MIT-6.824/tree/master/2016/shardkv)

Architecture
---

lab4 的架构是典型的 M/S 架构（a configuration service and a set of replica groups)，不过实现十分基础，很多功能没有实现：1) shards 之间的传递很慢并且不允许 concurrent client acess；2) 每个 raft group 中的 member 不会改变。

configuration service

- 由若干 shardmaster 利用 raft 协议保证一致性的集群；
- 管理 configurations 的顺序：每个 configuration 描述 replica group 以及每个 group 分别负责存储哪些 shards；
- 响应 Join/Leave/Move/Query 请求，并且对 configuration 做出相应的改变；


replica group

- 由若干 shardkv 利用 raft 协议保证一致性的集群；
- 负责具体数据的存储（一部分），组合所有 group 的数据即为整个 database 的数据；
- 响应对应 shards 的 Get/PutAppend 请求，并保证 linearized；
- 周期性向 shardmaster 进行 query 获取 configuration，并且进行 migration 和 update；


Part A：The Shard Master
---

1. **RPC Join/leave/Move/Query**

	Client:

	- at most once semantics：每个 Client 及 每个 Request 赋予唯一的 id；
	- sequential consistency：对于每个 Client，仅有一条 Request RPC 在显式执行；

	Server:

	- 异步：开一个 goroutine 来监听和处理 accept entry，与各 RPC 中的 start entry 分离，利用 channel 进行同步和通信；
	- at most once semantics：主要针对写操作，本文在 accept operation 处进行 duplicated check (也可以在 start operation 处再加一层)，对于重复的 request，仅向 client 回复操作结果；

2. **Rebanlancing**
    
    lab4 对 shards 分配的要求是: divide the shards as evenly as possible among the groups, and should move as few shards as possible to achieve that goal.
    
    最直接的做法是: 首先计算每个 replica group 分配多少个 shards ( `quota = num_of_shards / num_of_group`, `remain = num_of_shards - num_of_group * quota`，然后依据 `num_of_server` 对 replica group 进行排序，server 数量多的 group 分配 shards 个数 `quota + 1`，其余分配 `quota` )，最后把 shards 从多的 group 移动到少的 group。

    然而，在 go 中实现上面的做法并不是那么 clean，很多特性在语言层面上并不支持，比如: 1) get map.keys(); 2) 根据特定的字段进行排序等，自行实现实在麻烦。
 
    因此，由于 lab 中只有 join/leave 会造成 imbalance，并且 group 是逐个 join/leave。所以 simple first，首先统计 prior config 中每个 group 有多少个 shards，并且计算 present config 中平均每个 group 分配多少个 shards (`quota = num_of_shards / num_of_group`)，然后循环将 max_shards_group/leaving_group 中的 shards 分配给 joining_group/min_shards_group.

Part B: Sharded Key/Value Server
---

1. **RPC PutAppend/Get/TransferShard**
    
    > Hint: Think about how the shardkv client and server should deal with ErrWrongGroup. Should the client change the sequence number if it receives ErrWrongGroup ? Should the server update the client state if it returns ErrWrongGroup when executing a Get / Put request?   
     
    绝大部分与 Part A 一样，除了 ErrWrongGroup 的异常处理。

    当 server 发现 ErrWrongGroup 时，不需要 update client state；当 client 收到 ErrWrongGroup 时，不需要改变 operation sequence number，换个 group 继续 request 即可。

3. **Migrating Shard**
    
    > Hint: When groups move shards, they need to move some duplicate detection state as well as the key/value data. Think about how the receiver of a shard should update its own duplicate detection state. Is it correct for the receiver to entirely replace its state with the received one?
    
    - **假设**: configuration 由 cfg1 —> cfg2，shard S1 由 G1 -> G2
    - **How to migrate - push or pull**: 这时应该由 G1 push S1 还是由 G2 pull S1？一般来说，G2 完成 reconfig 操作后，随时会有针对 migrated shard 的 get/put/append 操作。如果由 G1 push S1，那么就无法保证 migrated shard 什么时候完成，出现错误。所以应该由 G2 在进行 reconfig 时进行 pull;
    - **Response to migration**: 由于 lab 中 group 是逐个 join/leave，同一个 group 不会同时迁入和迁出 shards，亦即是迁入和迁出 shards 的 groups 不会相交。因此当 G1 处于 cfg2 时，G1 已经不再负责 S1，之后可随时响应 migration 迁出 S1。亦即是：当 G1.cfg.num > G2.cfg.num 才能响应 migration，否则必须等待 G1 更新完毕 (避免互相请求出现死锁)。

2. **Reconfiguration**
    
    > Hint: Process re-configurations one at a time, in order.
    
    > Hint: When group G1 needs a shard from G2 during a configuration change, does it matter at what point during its processing of log entries G2 sends the shard to G1?

    - **Detect**: 开一个 goroutine 来周期性地向 shardmaster query 最新的 configuration (由 group leader 完成)，当发现 configuration 更新时 (one by one in order)，即向其余 group 获取需要的 shards (migrating shard)；
    - **When to reconfigurate**: Reconfiguration operation 仅仅是 group leader 从 shardmaster 获取到的 operation proposal，新的 configuration 什么时候生效是由各 group 自身决定，本文的实现是当 group 获取到需要的 shards 后再进行 reconfiguration update。因此会出现当 client 根据新的 configuration 向 shardkv 发起 request 时，各个 group 都还没完成 configuration update。不过这并没什么影响，我们可以让 client 一直 loop 向每个 replica group 发起 request (当然是一直被拒)，直到目标 group 完成 reconfiguration update;
    - **Consistency**: Reconfiguration operation 会影响到 PutAPpend/Get，因此同样需要利用 raft 保证 group 内的一致性，确保集群内完成了之前的操作后同时进行 Reconfiguration；
    - **Replace What**: 包括 config，StoreShard (data) 以及 Ack (duplicate detection)，其中 Ack 不是 replace entirely，仅当 kv.ack[clientId] < migration.Ack[clientId] 才更新；
    
Reference
---
> [bluesea147/6.824/shardmaster](https://github.com/bluesea147/6.824/blob/master/src/shardmaster/server.go)

> [bluesea147/6.824/shardkv](https://github.com/bluesea147/6.824/blob/master/src/shardkv/server.go)
   
    