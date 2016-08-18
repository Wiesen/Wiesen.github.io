+++
date = "2016-08-06T17:32:46+08:00"
draft = false
title = "MIT 6.824 lab3 Fault-Tolerant Key/Value Service Implementation"

topics = ["Distributed System", "MIT 6.824"]
+++

lab3 是基于 lab2 实现的 Raft Consensus Algorithm 之上实现 KV Service。主要分为两部分：

1. Part A：Key/value service without log compaction，即实现基本的分布式存储服务。
2. Part B: Key/value service with log compaction，即在 Part A 基础上实现 log compaction。

代码分别可以 pass 各个 test case，但所有一起跑时有时会卡在 `TestPersistPartition()` 这里，初步猜测是 raft 的实现还有点 bug。

[Code Link](https://github.com/Wiesen/MIT-6.824/tree/master/2016/kvraft)

Part A：Key/value service
---
1. **KV Database Client API**

	key/value database 的 client API 必须满足以下要求：

	- 保证仅执行一次(at most once semantics)：API 必须为每个 Client 及 每个 Request 赋予唯一的 id；
	- 必须向使用该 API 的应用提供 sequential consistency：对于每个 Client，仅有一条 Request RPC 在显式执行(利用 lock 实现)；
 
	此外，API 应当一直尝试向 key/value server 发起 RPC 直到收到 positive reply；并且记住 leader id，从而尽可能避免失败次数。

2. **Raft Handler**

	Raft Server Handler 需要满足以下要求：

	- 保证每个 client 的每条 request (主要是写操作) 仅执行一次：对于重复的 request，仅向 client 回复操作结果；
	
		本文为每个 client 维护一个已执行的最大 requestId 值 (`map[int64]int`)，从而检测并过滤重复的 request

			 func (kv *RaftKV) isDuplicated(clientId int64, requestId int) bool {
				kv.mu.Lock()
				defer kv.mu.Unlock()
				if value, ok := kv.ack[clientId]; ok && value >= requestId {
					return true
				}
				kv.ack[clientId] = requestId
				return false
			 }

 	- 在向 raft 添加等待结果的同时，需要一直监听返回管道 `applyCh` ，以接收已经达成一致的 entry；
 	
   		我们开一个 goroutine `update()` 一直监听 `applyCh`，并且基于 entry 的 index 各维护一个管道 (`map[int]chan Result`)，存放在 raft servers 间达成一致的 entry，等待 handler 的读取或者直接丢弃。
		
			 func (kv *RaftKV) Update() {	// ignore snapshot
				for true {
					msg := <- kv.applyCh
					request := msg.Command.(Op)
					var result Result
					...						// set variable value
					result.reply = kv.Apply(request, kv.isDuplicated(clientId, requestId))
					kv.sendResult(msg.Index, result);
				}
			  }
				
			 func (kv *RaftKV) sendResult(index int, result Result) {
				kv.mu.Lock()
				defer kv.mu.Unlock()
				if _, ok := kv.messages[index]; !ok {
					kv.messages[index] = make(chan Result, 1)
				} else {
					select {
					case <- kv.messages[index]:
					default:
					}
				}
				kv.messages[index] <- result
			 }

  		key/value server 向 raft 添加一个 entry 后，阻塞读取其 index 对应的管道，直到接收到结果或者超时（本文设置为 1s）。当接收到的结果 clientId 或者 requestId 不一致时，表明 leader 已经发生了更替，由 Client 重新向 server 发起 RPC。

	- 有一个需要注意的地方是，**必须利用 `gob.Register()` 注册需要通过 RPC 发送的结构体**，这样结构体才能够被解析，否则发送过去就是一个 `nil`。
  
Part B: log compaction
---
随着运行时间的增加，raft server 的 log table 会越来越大，不仅会占用越多空间，而且一旦出现宕机则 replay 也需要越长时间。比如不加以管理，则势必影响服务的可用性。为了令其维持合理长度不至于无限增加，必须在适当的时候抛弃旧的 log entries。

这部分主要有以下实现点：

1. **Take snapshots independently**

	这里各个 raft server 各自独立进行 snapshot，而这并不会影响 raft 的一致性。因为数据始终从 leader 流向 follower，各个 raft server 只是将数据重新组织而已。

	这部分主要解决的问题是：

	- **(When)** 什么时候进行 snapshot：由 key/value server 检测所连接的 raft server 存储大小是否即将超过阈值，然后通知该 raft server 进行 snapshot；
	- **(What)** 在 snapshot 中存储什么信息：如 raft-extended paper Figure 12 里的描述，主要包括当前 `LastIncludedIndex`, `LastIncludedTerm` 以及 `data`，其中 `data` 是状态机状态，来自于 key/value server。一点需要注意的是，必须保存用于检测 duplicate client requests 的数据，因此 `data` 在这里包括整个数据库和检测重复的 map；
	- **(How)** 如何令 raft 在仅有最新一部分 log 的情况下保持正常运行：主要是将 `rf.logTable[0]` 作为起始 entry，一旦有 follower 请求更老的 entries，则应该发送 InstallSnapshot RPC；
  
	需要增加和修改的地方如下：

	- `update()`： 在每次接收到 raft server 返回的 entry 时，检测 raft state size，若临近阈值则将封装 `data`，然后通知 raft 从 `entry.index` 开始 snapshotting；
	- `StartSnapshot(data []byte, index int)`：删除 log table 中序号小于 `index` 的 entries，并且在 `data` 后添加最后包括的 entry 的相关数据，最后持久化保存；
	- 其余琐碎并且细节的修改还包括：对 entry 增加一个字段 `index`，记录其在所在 `term` 的序号，在需要对 log table 进行操作地方（尤其是涉及对 entry 在 table 中的位置），注意 log table 中的起始序号：`baseLogIndex := rf.logTable[0].Index` 再进行处理；

2. **InstallSnapshot RPC**

	当一个 follower 远远落后于 leader（即无法在 log table 中找到匹配的 entry），则应该发送 InstallSnapshot RPC。我们参考 raft-extended paper Figure 13 设置数据结构，但有所简化：由于 lab 里的 Snapshot 不大，因此没有设置 chunk，也就是不需要对 snapshot 进行切分。
 
	- **InstallSnapshot RPC sender**
	
		当 heartbeat timeout 时 leader 检测 follower 的 log table 是否与自身一致，如果 `rf.nextIndex[server] <= baseLogIndex`，则 leader 应该向 follower `server` 发送 InstallSnapshot RPC,否则 `AppendEntries`。这里发送设置好相应的参数即可，跟发送 `AppendEntries` 差不多，收到正确的 reply 则 更新 `matchIndex` 和 `nextIndex`。
		
		     baseLogIndex := rf.logTable[0].Index
			 if rf.nextIndex[server] <= baseLogIndex {
				...	 			// set variable value
				var reply InstallSnapshotReply
				if rf.sendInstallSnapshot(server, args, &reply) {
					if rf.role != Leader {
						return
					}
					if args.Term != rf.currentTerm || reply.Term > args.Term {
						if reply.Term > args.Term {
							...	// update term and role
						}
						return
					}
					rf.matchIndex[server] = baseLogIndex
					rf.nextIndex[server] = baseLogIndex + 1
					return
				}
			 }

	- **InstallSnapshot RPC handler**
 
		由于没有设置 chunk，所以这里的 handler 比 paper 中的 implementation 简化不少：
		
		- 首先检测 `args.Term < rf.currentTerm`，满足则直接返回自身 term 即可；
		- 其次检测 `rf.currentTerm < args.Term`,满足则更新自身 term 和 role;
		- 然后更新自身的 log table，把 LastIncluded entry 前的丢弃；
		- 最后将 snapshot 发送给 key/value server，以重置数据库数据，更新状态。

最后
---
写 lab3 的时候，对 lab2 的代码调整很大，发现了不少 bug 和逻辑不合理的地方，整体上对 raft 的理解更加深入了。
