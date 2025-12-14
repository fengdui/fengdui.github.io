---
title: "kafka消息可靠性 数据一致性"
date: "2020-09-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
# 消息可靠性 数据一致性
- AR Assigned Replicas 所有副本
- ISR是AR中的一个子集，由leader维护ISR列表，follower从leader同步数据有一些延迟（包括延迟时间replica.lag.time.max.ms和延迟条数replica.lag.max.messages两个维度, 当前最新的版本0.10.x中只支持replica.lag.time.max.ms这个维度），任意一个超过阈值都会把follower剔除出ISR,存入OSR（Outof-Sync Replicas）列表，新加入的follower也会先存放在OSR中。
- AR=ISR+OSR。
- HW HighWatermark consumer能够看到的此partition的位置 取一个partition对应的ISR中最小的LEO作为HW，consumer最多只能消费到HW所在的位置
- LEO LogEndOffset的缩写，表示每个partition的log最后一条Message的位置
![img_39.png](/pic/img_39.png)

- request.required.acks
- 1（默认）：这意味着producer在ISR中的leader已成功收到数据并得到确认。如果leader宕机了，则会丢失数据。
- 0：这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
- -1：producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当ISR中只有leader时（前面ISR那一节讲到，ISR中的成员由于某些情况会增加也会减少，最少就只剩一个leader），这样就变成了acks=1的情况。所以需要将min.insync.replicas至少设置为2。
- min.insync.replicas 设定ISR中的最小副本数是多少, 默认值为1，当且仅当request.required.acks参数设置为-1时，此参数才生效。
- unclean.leader.election.enable=false时，leader只能从ISR中选举。
# 1.什么时候HW追上LEO, 使得消费者能够消费到对应的数据，如7 8 9 10。
- 当所有ISR集合中的分区都同步到了7, leader才会将HW设置到7的offset, 当ISR中只有leader这个主分区时, 意味着不需要副本的同步就能直接将HW往后移动, 此时如果主分区异常， 选择
- Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。当request.required.acks=1就是有多个从的异步复制, request.required.acks=-1 并且min.insync.replicas>=2相当于同步复制, 起码有一主一从。
- 本质上也是cap中, ap和cp的权衡。
# 2.request.required.acks=-1  min.insync.replicas = 2 ISR集合中少于2会怎么样？
- 如果ISR中的副本数少于min.insync.replicas配置的数量时，客户端进行写操作会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。生产者需要使用同步的api, 对该异常进行处理, 如反馈系统异常给上游。但此时假如ISR集合中只存在一个分区的话还是可以read的。
# 3.leader crash
- leader crash之后，假如unclean.leader.election.enable=false 必须从ISR集合中选择作为新的leader, 如果配合request.required.acks=-1 min.insync.replicas>=2, 则不会数据丢失。
# 4.request.required.acks=-1  min.insync.replicas = 2 假如主分区写入数据之后 副本分区没来得及同步或者同步一半主分区就crash
![img_40.png](/pic/img_40.png)
- 生产者使用同步的api, 主分区写入4 5消息, 此时还没有响应给客户端结果，需要等待所有ISR集合都同步完成才能返回给客户端结果。
- 然后副本1同步了4 5消息, LEO为5, 副本2同步4消息, LEO为4, 三个分区的HW都为3,当重新选举leader之后会进行截断到HW。