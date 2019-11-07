---
title: Raft--易于理解的一致性算法
date: 2019-11-07T00:00:00+23:00
---

这篇文章讲解如何用go实现raft算法，代码框架来自于Mit6.824分布式课程

### 领导选举

------

1. **electForLeader**函数主干，候选人针对每一个peer发送**请求投票RPC**

   ```go
   for i:=0; i<len(rf.peers); i++ {
       // meet myself
       if i==rf.me {
           continue
       }
       go func(index int) {
           reply := &RequestVoteReply{}
           response := rf.sendRequestVote(index, &args, reply)
           if response {
               ...
           }
       }(i)
   }
   ```

2. 获得响应后，要将reply的term于candidate的currentTerm进行比较

   ```go
   if reply.Term > rf.currentTerm {
   	rf.currentTerm = reply.Term
   	rf.changeRole("follower")
   	return
   }
   ```

3. 获得响应后，候选人要检查自己的**state** 和 **term** 是否因为发送**RPC**而改变 (**复制日志中同样需要考虑**)

   ```go
   if rf.state != "candidate" || rf.currentTerm!= args.Term { return }
   ```

4. 若候选人获得的投票超过**半数**，则变成领导人

5. **请求投票PRC** ⭐（接收者指接收 *请求投票PRC* 的peer）

   - 如果**candidate的term小于接收者的currentTerm**， 则不投票，并且返回接收者的currentTerm

     ```go
     reply.VoteGranted = false
     reply.Term = rf.currentTerm
     if rf.currentTerm > args.Term { return }
     ```

   - 如果**接收者的votedFor为空或者为candidateId，并且candidate的日志至少和接收者一样新**，那么就投票给候选人。candidate的日志至少和接收者一样新的含义：**candidate的最后一个日志条目的term大于接收者的最后一个日志条目的term或者当二者相等时，candidate的最后一个日志条目的index要大于等于接收者的**

     ```go
     if (rf.votedFor==-1 || rf.votedFor==args.CandidateId) &&
     		(args.LastLogTerm > rf.getLastLogTerm() ||
     			((args.LastLogTerm==rf.getLastLogTerm())&& (args.LastLogIndex>=rf.getLastLogIndex()))) {
                 reply.VoteGranted = true
                 rf.votedFor = args.CandidateId
                 rf.state = "follower"   // rf.state can be follower or candidate
                 ...
     }
     ```

------



### 日志复制

1. **appendLogEntries**函数的主干，leader针对每一个peer发送**附加日志RPC**

   ```go
   for i:=0; i<len(rf.peers); i++ {
       if i == rf.me {
           continue
       }
       go func(index int) {
           reply := &AppendEntriesReply{}
           respond := rf.sendAppendEntries(index, &args, reply)
           if reply.Success {
               ...
               return
           } else {
               ...
           }
       }(i)
   }
   ```

2. 同**领导选举**

3. 同**领导选举**

4. **回复成功**

   - 更新nextIndex, matchIndex
   - 如果存在一个N满足**N>commitIndex**，并且大多数**matchIndex[i] > N** 成立，并且**log[N].term == currentTerm**，则更新**commitIndex=N**

5. **回复不成功**

   - 更新**nextIndex**，然后重试

6. **附加日志RPC** ⭐

   - 几个再次明确的地方：

     - **preLogIndex**的含义：新的日志条目(s)紧随之前的索引值，是针对每一个follower而言的==nextIndex[i]-1，每一轮重试都会改变。

     - **entries[]**的含义：准备存储的日志条目；表示心跳时为空

       ```go
       append(make([]LogEntry, 0), rf.log[rf.nextIndex[index]-rf.LastIncludedIndex:]...)
       ```

     - **领导人获得权力**后，初始化所有的nextIndex值为自己的最后一条日志的index+1；如果一个follower的日志跟领导人的不一样，那么在附加日志PRC时的一致性检查就会失败。领导人选举成功后跟随着可能的情况

       ![](/static/images/appendLog.jpg)

   - reply增加 **ConflictIndex** 和 **ConflictTerm** 用于记录日志冲突index和term

   - 如果**leader的term小于接收者的currentTerm**， 则不投票

   - 接下来就三种情况
  
     1. **follower的日志长度比leader的短**
     2. **follower的日志长度比leader的长，且在prevLogIndex处的term相等**
     3. **follower的日志长度比leader的长，且在prevLogIndex处的term不相等**
   ```go
   if args.PrevLogIndex >=rf.LastIncludedIndex && args.PrevLogIndex < rf.logLen() {
     
  	  if args.PrevLogTerm != rf.log[args.PrevLogIndex-rf.LastIncludedIndex].Term {
     		  reply.ConflictTerm = rf.log[args.PrevLogIndex-rf.LastIncludedIndex].Term
  		     //  then search its log for the first index
     		  //  whose entry has term equal to conflictTerm.
     		  for i:=rf.LastIncludedIndex; i<rf.logLen(); i++ {
     			  if rf.log[i-rf.LastIncludedIndex].Term==reply.ConflictTerm {
     				  reply.ConflictIndex = i
     				  break
     			  }
     		  }
     		  return
      }
   }else {
      reply.ConflictIndex = rf.logLen()
      return 
   }
     
   index := args.PrevLogIndex
   for i:=0; i<len(args.Entries); i++ {
      index++
      if index >= rf.logLen() {
          rf.log = append(rf.log, args.Entries[i:]...)
          rf.persist()
          break
      }
      if rf.log[index-rf.LastIncludedIndex].Term != args.Entries[i].Term {
          rf.log = rf.log[:index-rf.LastIncludedIndex]
          rf.log = append(rf.log, args.Entries[i:]...)
          rf.persist()
          break
      }
   }
   ```
     
     
     
   - 如果 **leaderCommit > commitIndex**，令 commitIndex等于leaderCommit 和新日志条目索引值中较小的一个
------



### 日志压缩

1. 增量压缩的方法，这个方法每次只对一小部分数据进行操作，这样就分散了压缩的负载压力

2. ![](/static/images/log-compact.jpg)

3. **安装快照RPC**

   - 尽管服务器通常都是独立的创建快照，但是领导人必须偶尔的发送快照给一些落后的跟随者

   - 三种情况

     - leader的**LastIncludedIndex**小于等于follower的**LastIncludeIndex**
     - leader的**LastIncludedIndex**大于follower的**LastIncludeIndex**，leader的**LastIncludedIndex**小于follower日志的最大索引值
     - leader的**LastIncludedIndex**大于等于follower日志的最大索引值

     **对应的处理方式**

     - 如果接收到的快照是自己日志的前面部分，那么快照包含的条目将全部被删除，但是快照后面的条目仍然有效，要保留
     - 如果快照中包含没有在接收者日志中存在的信息，那么跟随者丢弃其整个日志，全部被快照取代。



------

不涉及集群成员变化