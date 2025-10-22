---
title: "raft"
date: "2020-11-12"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

- 在一些一致性算法中，例如：Viewstamped Replication，
- 即使一开始没有包含全部已提交的条目也可以被选为领导人。
- 这些算法都有一些另外的机制来保证找到丢失的条目并将它们传输给新的领导人，
- 这个过程要么在选举过程中完成，要么在选举之后立即开始。
- 不幸的是，这种方式大大增加了复杂性。
- Raft 使用了一种更简单的方式来保证：也就是日志落后的候选人，
- 无法被选中为Leader，这其实大大减小了选举和日志复制的相关性，
- 简化了raft的算法理解难度。
- termId：任期号，时间被划分成一个个任期，每次选举后都会产生一个新的 termId，一个任期内只有一个 leader。
- termId 相当于 paxos 的 proposalId。
- 2个超时
- 选举超时 超过一定时间则变为candidate开始发起选举。
- AppendEntries 作为心跳超时的依据 超过时间则变为candidate开始发起选举 选举阶段和日志复制阶段都会用到这个类型的rpc请求
- 复制
- 在 Leader 复制给 Follower 时，要传递当前最新日志 currenTermId 和currentLogIndex，以及上一条日志 preCurrentTermId 和 preCurrentLogIndex
- Leader可能在如下几种情形挂掉：
- 1.数据到达Leader前：无影响，Client重试即可
- 2.数据到达Leader，但未复制到Follower：Follower没有数据，重选Leader后Client重试即可，原Leader恢复重新加入集群从当前任期新Leader处同步数据
- 3.数据到达Leader，成功复制到所有Follower，但还未向Leader响应接收：Follower上有Uncommitted的数据但保持一致，重选Leader后可完成数据提交。
- Client由于没有收到反馈，可重试提交，但这种情况Raft要求RPC请求实现幂等性
- 4.数据到达Leader，成功复制到部分Follower，但还未向Leader响应接收：Follower上有Uncommitted的数据且不一致，Raft要求投票只能投给拥有最新数据的节点，因此情况同上
- 5.数据到达Leader，成功复制到所有或超过半数Follower，数据已Commit，但还未响应Client：集群内部数据已一致，Client 重复重试基于幂等策略对一致性无影响