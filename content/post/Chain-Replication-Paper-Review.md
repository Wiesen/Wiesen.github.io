+++
date = "2016-03-19T21:33:09+08:00"
draft = false
title = "Chain Replication Paper Review"

tags = ["Distributed System"]
+++

本文是读完 Van Renesse R, Schneider F B. Chain Replication for Supporting High Throughput and Availability[C]//OSDI. 2004. 的总结。

Summary
---
Chain replication is a new approach to coordinating clusters of fail-stop storage servers.

Chain replication 采用 ROWAA (read one, write all available) 方法, 具有良好的 Scalability.

该方法目的是, **不以牺牲强一致性为代价来实现高吞吐和高可用**, 从而提供分布式存储服务. 

A Storage Service Interface
---
**Clients** 发送 query 或 update 操作 request

**The storage service** 为每个 request 生成 reply 发送会 client 告知其已经接收或已经处理完成, 从而 client 可以得知某 request 是否接收成功以及是否处理完成. 

Client request type:

- `query(objId, opts) -> value`: retrieve current value of *opts* of *objId*
- `update(objId, newVal, opts) -> value`: update *opts* of *objId* with *newVal*

Client's view of an object:
    
    State is:
    	Hist[objId]: history of all updates to objId
    	Pending[objId]: set of pending requests for objId
    Transitions are:
    	T1: Client request r arrives: 
    		Pending[objId] += {r}
    	// 一个 client request 接收失败 = server 忽略了该 client request
    	T2: Client request r ∈ Pending[objId] ignored: 
    		Pending[objId] -= {r}
    	T3: Client request r ∈ Pending[objId] processed: 
    		Pending[objId] -= {r}
    		if r = query(objId, opts) then 
    			reply according options opts based on Hist[objId]
    		else if r = update(objId, newVal, opts) then
    			Hist[objId] := Hist[objId] · r
    			reply according options opts based on Hist[objId]


Chain Replication Protocol
---
**1. Assumptions: 所有服务器均假设为 fail-stop**

- each server halts in response to a failure
- a server’s halted state can be detected by the environment

**2. Protocol Details**

在 chain replication 中, 所有 servers 根据 *objID* 线性排列从而组成一个链表.
<img src="http://7vij5d.com1.z0.glb.clouddn.com/the%20chain.png" width="400"/>

- 所有 update 操作由 HEAD 结点接收并开始处理, 然后按照FIFO顺序向链表中的下一个节点传递, 直到该 update 操作被 TAIL 节点处理.
- 所有 query 操作由 TAIL 结点接收并处理.
- 所有 query 操作 / update 操作的确认由 TAIL 结点处理 (即发送 reply 给 client).
	
**3. Coping with Server Failures**

论文中构建一个 *master* server, 其主要功能如下 (为区分本文将其余负责数据存储的 server 称为 *data* server):

- 检测其余 *data* servers 的失败
- 在链表新增或删除节点时, 通知 *data* servers 更新 predecessor 及 successor
- 告知 client 链表的 HEAD 节点和 TAIL 节点

论文中假设 *master* server 永不崩溃。而实际上该论文的 prototype 利用 Paxos 协调 *master* server 的各个 replicas 从而 behave in aggregate like a single process that does not fail, 以此避免单点故障.

下面仅讨论 *data* server 故障, 即如何在链表中的节点出现故障时保证存储服务的强一致性.主要分为以下头节点, 尾节点和中间节点三种情况.

论文中阐述了两个性质: 

- **Update Propagation Invariant (更新传递不变性)**. 意即对于编号 *i* 和 *j*, 若有: ![fomula](http://latex.codecogs.com/gif.latex?i%20%5Cle%20j) (例如 *i* 是 *j* 的 predecessor), 则有:  successor 的 update 操作序列 是 predecessor 的前缀 —— ![fomula](http://latex.codecogs.com/gif.latex?Hist%5Ej_%7BobjID%7D%20%5Cpreceq%20Hist%5Ei_%7BobjID%7D) (该性质根据链表节点间 update 操作由 FIFO 传递得出)
- **Inprocess Requests Invariant (上下文请求不变性)**. 每个 server *i* 维护一个列表 <img src="http://www.forkosh.com/mathtex.cgi?Sent_i}">, 其中存储着 server *i* 已经处理并传递给 successor 节点但可能未被 tail 节点处理的 update requests. 当 tail 节点处理了一个 update request *r* 后会发送确认 *ack(r)* 给 predecessor, server *i* 接收 *ack(r)* 后将 *r* 从 <img src="http://www.forkosh.com/mathtex.cgi?Sent_i}"> 中删除, 然后依次向前传递. 据此, 若有: ![fomula](http://latex.codecogs.com/gif.latex?Hist%7B%5Ei_%7BobjID%7D%7D%20%3D%20Hist%7B%5Ej_%7BobjID%7D%7D%20%5Coplus%20Sent_i) (根据 tail 节点接收到的 request 必定已被其所有 predecessors 接收到这一事实得出)

 **Head 节点故障停止**: 将 Head 节点从链表中移除, 其 successor 节点称为新的 Head 节点. 旧 Head 节点已经传递的 update 操作继续传递, 而丢失的 update 操作可视为 server 忽略了该 update (如前所述等同于server 接收该 client request 失败), 因此对应的 client request 将无法接收到 reply, 此时 client 会 resend request. 不影响存储服务的强一致性.

 **Tail 节点故障停止**: 将 Tail 节点从链表中移除, 其前继 Tail- 节点称为新的 Tail 节点. 由于 ![fomula](http://latex.codecogs.com/gif.latex?Tail%5E-%20%3C%20Tail%20%5Cto%20Hist%7B%5E%7Btail%7D_%7BobjID%7D%7D%20%5Cpreceq%20Hist%7B%5E%7BTail%5E-%7D_%7BobjID%7D%7D), 从用户的角度看即数据变新变多了, 因此并不影响存储服务的读一致性.

 **中间节点故障停止**: 将故障节点 S 从链表中移除, 然后 *master* server 首先通知故障节点的后继 S+ 节点新的链表配置, 然后通知前继 S- 节点连接后继 S+ 节点并要求其处理 <img src="http://www.forkosh.com/mathtex.cgi?Sent^-}"> 中的 update request, 后序 update 操作继续传递下去, 因此也不影响存储服务的强一致性.

 **扩展链表**: 当越来越多故障节点被移除, 链表将会缩短, 同时容错性下降.因此当链表变短时应当向链表中添加新的 servers. 理论上可以在链表的任何位置添加新 server, 但最简单的方法是在结尾添加 T+ 节点. 首先 T+ 通过复制 <img src="http://www.forkosh.com/mathtex.cgi? Hist{^T_{objID}}"> 到 <img src="http://www.forkosh.com/mathtex.cgi? Hist{^{T^+}_{objID}}"> 完成初始化, 然后 T+ 节点一边处理由 T 节点 forward 过来的 query request 一边处理 <img src="http://www.forkosh.com/mathtex.cgi?Sent^T}">, 最后 T+ 正式作为新的 tail 节点.

Comparision to Primary/Backup Protocols
---
Chain replication 是 primary/backup 方法的一种改进, 实际上是一种副本管理的状态机方法.

在 primary/backup 方法中, 有一个 *primary* server, 负责序列化 client requests (从而保证强一致性), 然后将序列化的 client requests 或 resulting updates 分散发送到各个 *backup* servers, 等待接收非故障 *backups* 的确认信息, 最后发送 reply 给 client. 当 *primary* server 故障停止, 其中一个 *backup* server 将提升为 *primary* server.

相比 primary/backup 方法, Chain replication 的不同如下:

- client requests 由两个 replicas 处理, 即 head 节点负责序列化处理 update request, tail 节点并行处理 query requests, 降低了 query requests 的 latency.
- Chain replication 只能串行传递 update requests, 因此发送 reply 的 latency 与 the sum of server latencies 成比例.

当出现 server 故障停止时, Chain replication 和 primary/backup 方法的主要时延都是检测 server failure, 其次是 recovery.

Simulation Experiments
---
论文在四种情况下进行实验:

- Single Chain, No Failures
- Multiple Chains, No Failures
- Effects of Failures on Throughput
- Large Scale Replication of Critical Data

Additional: Application
---
在 [CRAQ](https://www.usenix.org/legacy/events/usenix09/tech/full_papers/terrace/terrace.pdf) 论文中介绍了基于 chain replication 的 CRAQ 系统, 该系统扩展了 chain replication protocol, 使链表上的所有节点均可处理 query 操作, 提高系统的吞吐, 同时仍然提供强一致性保证.

微软云计算平台 [Windows Azure](http://sigops.org/sosp/sosp11/current/2011-Cascais/printable/11-calder.pdf)、[FDS](http://sigops.org/sosp/sosp11/current/2011-Cascais/printable/11-calder.pdf) 都使用 chain replication protocol 提供强一致性保证.

----------
**Reference**

> https://www.youtube.com/watch?v=nEbD-qutsKo

> http://blog.csdn.net/yfkiss/article/details/13772669

> http://blog.xiaoheshang.info/?p=883