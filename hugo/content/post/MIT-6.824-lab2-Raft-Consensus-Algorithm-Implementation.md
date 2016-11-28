+++
date = "2016-06-10T21:32:46+08:00"
draft = false
title = "MIT 6.824: lab2 Raft Consensus Algorithm Implementation"

topics = ["Distributed System", "MIT 6.824"]
+++

Raft 将一致性问题分为了三个相对独立的子问题，分别是：

- **Leader election**：当前 leader 崩溃时，集群中必须选举出一个新的 leader；
- **Log replication**：leader 必须接受来自 clients 的 log entries，并且将其 replicate 到集群机器中，强制其余 logs 与其保持一致；
- **Safety**：Raft 中最关键的 safety property 是 State Machine Safety Property，亦即是，当任一机器 apply 了某一特定 log entry 到其 state machine 中，则其余服务器都不可能 apply 了一个 index 相同但 command 不同的 log。

差不多依据上述划分，6.824 中 Raft 的实现指导逻辑还是挺清晰的，其中 safety property 由 Leader election 和 Log replication 共同承担，并且将 Persistence 作为最后一部分。实现过程主要分为：

- Leader Election and Heartbeats：首先令 Raft 能够在不存在故障的情景下选举出一个 leader，并且稳定保持状态；
- Log Replication：其次令 Raft 能够保持一个 consistent 并且 replicated 的 log；
- Persistence：最后令 Raft 能够持久化保存 persistent state，这样在重启后可以进行恢复。

其中，本文主要参考 Raft paper，其中的 **figure 2** 作用很大。本文实现**大量依赖 channel 实现消息传递和线程同步**。

[Code Link](https://github.com/Wiesen/MIT-6.824/blob/master/2016/raft/raft.go)

Leader Election and Heartbeat
---
实现 Leader Election，主要是需要完成以下三个功能：

1. **Role Transfer：state machine & election timer**

    首先实现 Raft 的 sate machine，所有 server 都应当在初始化 Raft peer 时开启一个单独的线程来维护状态，令其在 Follower，Candidate，Leader 三个状态之间进行转换。
    
    有一点**注意**的是，`nextIndex[]` 和 `matchindex[]` 需要在 election 后进行 reinitialize。
    
       func (rf *Raft) changeRole() {
			for true {
				switch rf.role {
				case Leader:
					for i := range rf.peers {
						rf.nextIndex[i] = rf.logTable[len(rf.logTable)-1].Index + 1
						rf.matchIndex[i] = 0
					}
					go rf.doHeartbeat()
					<-rf.chanRole
				case Candidate:
					chanQuitElect := make(chan bool)
					go rf.startElection(chanQuitElect)
					<-rf.chanRole
					close(chanQuitElect)
				case Follower:
					<-rf.chanRole
				}
			}
		}
    
    此外，定时器 election timer 的作用十分关键，所有 server 都应当在初始化 Raft peer 时开启一个单独的线程来维护 election timer。
     
    当 election timer 超时时，机器将会转换至 Candidate 状态。另外在以下三种情况下将会 reset 定时器：收到合法的 heartbeat message；投票给除自身以外的 candidate；自身启动election（本文视为转换为 Candidate 状态）。
     
    同样有一点需要注意，需要确保不同机器上的 timer 异步，也就是不会同时触发，否则所有机器都会自投票导致无法选举 leader。在 golang 中通过以时间作为种子投入到随机发生器中：`rand.Seed(time.Now().UnixNano())`
    
        func (rf *Raft) startElectTimer() {
        	floatInterval := int(RaftElectionTimeoutHigh - RaftElectionTimeoutLow)
        	timeout := time.Duration(rand.Intn(floatInterval)) + RaftElectionTimeoutLow
        	electTimer := time.NewTimer(timeout)
        	for {
        		select {
        		case <- rf.chanHeartbeat: // received valid heartbeat message
        			rf.resetElectTimer(electTimer)
        		case <- rf.chanGrantVote: // voted for other server 
        			rf.resetElectTimer(electTimer)
        		case <-electTimer.C:      // fired election  
        			rf.chanRole <- Candidate
        			rf.resetElectTimer(electTimer)
        		}
        	}
        }
        
2. **RequstVote RPC 的 sender 和 handler**

    完成 Raft 的 sate machine后，开始实现 Raft 中的 RequestVote 操作，使得能够选举出一个 leader。
    
    机器处于 Candidate 状态时应当启动 election：`currentTerm++` -> `votedFor = me` -> `sendRequestVote()`。其中 `sendRequestVote()` 应当异步，也就是并发给其余机器发送 RequstVote。
     
    当出现以下情况，当前 election 过程终结：
    
    - 获得大多数机器的投票 -> 转换为 Leader 状态；
    - 接受到的 reply 中 `term T > currentTerm` -> 更新 `currentTerm`，转换为 Follower 状态；
    - election timer 超时 -> 终结当前 election，并且启动新一轮 election，保持 Candidate 状态；
     
    一个机器接收到 RequstVote RPC，需要决定是否投票：
    
    - 如果 RequstVote RPC 中的 `term T < currentTerm`，直接返回 false 拒绝投票即可；
    - 如果 RequstVote RPC 中的 `term T > currentTerm`，则需要更新 `currentTerm`，转换为 Follower 状态；
    - 如果该机器在 `currentTerm` 已经投票，则直接返回 false 拒绝投票；
    - 否则在满足 "Candidate's log is at least as up-to-date as receiver's log" 时返回 true 投票，并且 reset election timeout；
    
    所谓 **“up-to-date”** 简单来说就是：比较两个 log 中的最后一条 entry 的 `index` 和 `term`：
    
    - 当两个 log 的最后一条 entry 的 term 不同，则 **later** term is more up-to-date；
    - 当两个 log 的最后一条 entry 的 term 相同，则 whichever log is **longer** is more up-to-date
    
            // RequestVote RPC handler.
            func (rf *Raft) RequestVote(args RequestVoteArgs, reply *RequestVoteReply) {
              if rf.currentTerm > args.Term {
              	reply.Term = rf.currentTerm
              		reply.VoteGranted = false
              	return
              }
              if rf.currentTerm < args.Term {
              	rf.currentTerm = args.Term
              	rf.votedFor = -1
              	rf.chanRole <- Follower
              }
              reply.Term = args.Term
              if rf.votedFor != -1 && rf.votedFor != args.CandidateId {
              	reply.VoteGranted = false
              } else if rf.logTable[len(rf.logTable)-1].Term > args.LastLogTerm {
              	reply.VoteGranted = false 	//! different term
              } else if len(rf.logTable)-1 > args.LastLogIndex && rf.logTable[len(rf.logTable)-1].Term == args.LastLogTerm { 
              	reply.VoteGranted = false   //! same term but different index
              }else {
              	reply.VoteGranted = true
              	rf.votedFor = args.CandidateId
              	rf.chanGrantVote <- true
              }
            }

3. **Heartbeat 的 sender 和 handler**

    最后实现 Raft 中的 Heartbeat 操作，能够令 Leader 稳定保持状态。
     
    机器处于 Leader 状态时应当启动 heartbeat，而其实际上就是不含 log entry 的 AppendEntries（只需要检测某一机器的 log 是否最新）。
     
    其中 `sendHeartbeat()` 应当异步，也就是并发给其余机器发送 heartbeat。每一个 heartbeat 线程利用 timer 来周期性地触发操作。
        
        func (rf *Raft) sendHeartbeat(server int) {
            for rf.getRole() == Leader {
        				rf.doAppendEntries(server)  // later explain in Log Replication
        				heartbeatTimer.Reset(RaftHeartbeatPeriod)
        				<-heartbeatTimer.C
        	    }
        }
     
    一个机器接收到不含 log entry 的 AppendEntries RPC（也就是 heartbeat）时，需要决定是否更新自身的 term 和 leaderId：
    
    - 如果 AppendEntries RPC 中的 `term T < currentTerm`，则 reply 中返回 `currentTerm` 让发送方更新；
    - 如果 RequstVote RPC 中的 `term T > currentTerm`，则需要更新 `currentTerm` 和 `leaderId`，转换为 Follower 状态，并且 reset election timeout；
     
            // temporary AppendEntries RPC handler.
            func (rf *Raft) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) {
	            if args.Term < rf.currentTerm {
	            	reply.Term = rf.currentTerm
	            	reply.Success = false
	            	return
	            }
	            //! update current term and only one leader granted in one term
	            if rf.currentTerm < args.Term {
	            	rf.currentTerm = args.Term
	            	rf.votedFor = -1
	            	rf.leaderId = args.LeaderId
	            	rf.chanRole <- Follower
	            }
	            rf.chanHeartbeat <- true
	            reply.Term = args.Term
            }

Log Replication
---
完成了 Leader election 后，下一步是令 Raft 保持一个 consistent 并且 replicated 的 log。

1. **AppendEntries RPC sender**

    在 Raft 中，只有 leader 允许添加 log，并且通过发送含有 log 的 AppendEntries RPC 给其余机器令其 log 保持一致。
    
        func (rf *Raft) Replica() {
        // replica log
        for server := range rf.peers {
        	if server != rf.me {
        		go rf.doAppendEntries(server)
        	}
        }
    
    接下来主要是 `doAppendEntries(server int)` 的实现细节。
     
    当 Leader 的 log 比某 server 长时，亦即是 `rf.nextIndex[server] < len(rf.logTable)`，则需要发送 entry
     
    有一点要**注意**的是：Leader 在对某 server 进行上述的连续发送时间或者等待 reply 的时间可能会大于 heartbeat timeout，因此触发 AppendEntries RPC。然而 Leader 本身正在等待 reply，这时候重复发送是多余的。 
     
    本文检测 Leader 是否在向某 server 发送 AppendEntries RPC，若是则直接退出不再重复操作：这里设计为 Leader 为每一 server 维护一个带有**一个缓存**的管道，这样某 server 进行 AppendEntries RPC 前可以确认是否正在处理。
     
    最后如果 RPC failed 则直接退出，否则收到 AppendEntries RPC 的 reply 时：
    
    - 首先判断是否 `reply.Term > args.Term`，若是则 Leader 需要更新自身 currentTerm，并且转换为 Follower 状态。
    - 若 `reply.Success` 会表明达成一致，更新 `rf.matchIndex[server]` 和 `rf.nextIndex[server]`；否则仅更新 `rf.nextIndex[server]`（这里涉及到 3 中的优化）。
    
            func (rf *Raft) doAppendEntries(server int) {
				//! make a lock for every follower to serialize
				select {
				case <- rf.chanAppend[server]:
				default: return
				}
				defer func() {rf.chanAppend[server] <- true}()
			
				for rf.getRole() == Leader {
					rf.mu.Lock()
					var args AppendEntriesArgs
					...				// set variable value
					if rf.nextIndex[server] < len(rf.logTable) {
						args.Entry = rf.logTable[rf.nextIndex[server]:]
					}
					rf.mu.Unlock()

					var reply AppendEntriesReply
					if rf.sendAppendEntries(server, args, &reply) {
						if rf.role != Leader {
							return
						}
						if args.Term != rf.currentTerm || reply.Term > args.Term {
							if reply.Term > args.Term {
								...	// update term and role
							}
							return
						}
						if reply.Success {
							rf.matchIndex[server] = reply.NextIndex - 1
							rf.nextIndex[server] = reply.NextIndex
						} else {
							rf.nextIndex[server] = reply.NextIndex
						}
					}
				}
			}

2. **AppendEntries RPC handler** (2016.8.5 Update)

    一个机器接收到含有 log entry 的 AppendEntries RPC，前一部分跟处理 heartbeat 一样，剩下的部分则是决定是否更新自身的 log。根据 Raft paper 中的 figure 2 按部就班实现就好了。

	Update：处理 log entries array 还是需要注意一下：首先决定是否更新 log table，当满足条件决定更新后，主要存在三种更新情况：（1）更新旧 index；（2）更新旧 index，且添加新 index；（3）仅添加新 index。（如下图）

	![raft-appendentries](http://7vij5d.com1.z0.glb.clouddn.com/raft-appendentries.bmp)

	本文方法是：首先记录下当前开始处理的 `basicIndex := args.PrevLogIndex + 1`（即 `args.Entry` 中的起始 index），然后遍历 `args.Entry`，一旦 index 已达 logTable 末端或者某个 entry 的 term 不匹配，则清除 logTable 中当前及后续的 entry，并且将 `args.Entry` 的当前及后续 entry 添加到 logTable中。

3. **AppendEntries RPC optimization** (2016.8.5 Update)

    这一部分主要是优化 AppendEntries RPC，当发生拒绝 append 时减少 master 需要发送 RPC 的次数。虽然 Raft paper 中怀疑这个优化操作是否有必要，但 lab2 中有些 unreliable test case 就需要你实现这个优化操作…
     
    首先需要在 struct AppendEntriesReply 中添加两个数据：`NextIndex int`,  `FailTerm int`，分别用于指示 conflicting entry 所在 term，以及在该 term 中存储的第一个 entry 的 index；
     
    在 handler 的优化如下：如果 log unmatched，存在两种情况：

    - `len(rf.logTable) <= args.PrevLogIndex`：则直接返回 `NextIndex = len(rf.logTable)` 即可；
    - `rf.logTable[args.PrevLogIndex].Term != args.PrevLogTerm`：首先记录 `FailTerm`，然后从当前 conflicting entry 开始向前扫描，直到 index 为 0 或者 term 不匹配，然后记录下该 term 的第一个 index；
     
    在 sender 的优化如下：当 append 失败时，如果 `NextIndex > matchIndex[server]`，则 `nextIndex[server]` 直接退至 `NextIndex`，减少了需要尝试 append 的 RPC 次数；否则 `nextIndex[server]` 退回到 `matchIndex[server] + 1`。
    
    **STEP 4. Apply committed entries to local service replica**
    
    对于 Leader 而言，需要不断检测当前 term 的 log entry 的 replica 操作是否完成，然后进行 commit 操作。
     
    当超过半数的机器已经完成 replica 操作，则 Leader 认为该条 log entry 可以 commit。
     
    一旦当前 term 的某条 log entry L 是通过上述方式 commit 的，则根据 Raft 的 **Log Matching Property**，Leader 可以 commit 先于 L 添加到 log 的所有 entry。
    
        func (rf *Raft) updateLeaderCommit() {
			rf.mu.Lock()
			defer rf.mu.Unlock()
			defer rf.persist()
        	oldIndex := rf.commitIndex
        	newIndex := oldIndex
        	for i := len(rf.logTable)-1; i>oldIndex && rf.logTable[i].Term==rf.getCurrentTerm(); i-- {
        		countServer := 1
        		for server := range rf.peers {
        			if server != rf.me && rf.matchIndex[server] >= i {
        				countServer++
        			}
        		}
        		if countServer > len(rf.peers) / 2 {
        			newIndex = i
        			break
        		}
        	}
        	if oldIndex == newIndex {
        		return
        	}
			rf.commitIndex = newIndex

        	//! update the log added in previous term
        	for i := oldIndex + 1; i <= newIndex; i++ {
        		rf.chanCommitted <- ApplyMsg{Index:i, Command:rf.logTable[i].Command}
				rf.lastApplied = i
        	}
        }
     
    对于 Follower 而言，则：`commitIndex = min(leaderCommit, index of last new entry)`。 

Persistence
---
最后一步是持久化保存 persistent state，不过在 lab2 仅是通过 `persister` object 来保存，并没有真正使用到磁盘。实现了这一部分后可以稳定地 pass 掉关于 Persist 的 test case。

1. **Read persist**

    在 Make Raft peer 时读取 persister。

2. **Write persist**

    修改了 Raft 的 persistent state 后应当及时写至 persister，主要是在以下几个地方插入 write：
    
    - 启动了 election 修改了自身的 persistent state后；
    - 受到了 RequestVote RPC 和 AppendEntries RPC 的 reply，得知需要更新自身的 persistent state 后；
    -  RequestVote RPC handler 和 AppendEntries RPC handler 处理完毕后；
    -  Leader 收到 client 的请求命令，添加到自身的 log 后。

Defect
---
本文还有些不足，有待后续优化。主要是**不能稳定地** pass `TestUnreliableAgree()` + **不能** pass `TestFigure8Unreliable()`

To Be Continue...

2016-8-11 Update
---
最近几天 debug 了之前的代码，发现了若干个问题。目前的代码实现已经**比较稳定**地 pass 所有的 test case，上面的代码段也已经修改了。但仍然存在一个 bug: 在过 TestUnreliableAgree() 时 fail to reach agreement。这个 bug 的触发几率很低，还没想明白哪里出问题。

1. **LEADER ELECTION: Role Transfer (state machine)**

 之前实现 server state machine 的方法是利用 channel 来进行状态修改操作，并且会新开一个 goroutine 进行该状态死循环，而没有其他状态的处理逻辑。本文由一个 goroutine 阻塞读取该 channel，从而处理 server 的状态转换。然而，channel 同步是存在延时的，只有在当前 goroutine 被挂起或者休眠等时，才会转去处理。

 这样的方法存在一个 bug：新 Leader 产生后，旧 Leader 收到消息应该更新自身的 term 并且转换为 Follower；然而由于 channel 同步并非立即执行，旧 Leader 在自身状态被重新赋值前仍然会执行 Leader 的代码；这时候就会出现两个 Leader 同时处理 RPC。

 因此，修改的内容是：去掉 channel 同步方法，当需要进行状态转换时，立即修改 server 的状态，终止该 server 当前状态的执行。

2. **LOG REPLICATION: AppendEntries RPC**

 之前不能 pass `TestUnreliableAgree()` 和 `TestFigure8Unreliable()` 时丝毫没有提示，只能等程序运行时间过长然后被杀死。后来思考了一下问题原因：**程序运行过慢，跟不上 test case 要求的速度，所以后续的测试代码也根本没有执行**。

 然后发现程序执行慢的原因是：Leader 发送 AppendEntries RPC 每次仅携带一条 log entry，导致 server 无法快速 catch up。所以修改的主要内容就是：AppendEntries RPC 每次携带从 `nextIndex` 直到最新的 log entries。

Reference
---
> [MIT 6.824 Week 3 notes](https://www.douban.com/note/549229678/)