---
title: "记录一个elastic-job分片未触发问题"
date: "2024-05-09"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

现象：3个分片 2个分片执行 1个分片任务未执行 导致有部分任务一直没触发，版本是3.1。

zk上面的分片信息

![img_7.png](/pic/img_7.png)

可以看到分片1 是不在instances里面的。

阅读源码发现 192.168.52.64@-@4784 这种是instanceId, 系统启动时对于每个任务的每个节点都会生成一个临时节点，参考源码，org.apache.shardingsphere.elasticjob.lite.internal.instance.InstanceService#persistOnline。
当我重启某个节点时，可以看到 zk isntances目录下面确实刷新了。

继续阅读重新分片的源码，重新分片在任务触发时，判断是主节点并且有标记（位于leader/sharding/necessary）, 则会拿instances目录重新分片写到sharding目录下。

那什么时候有重新分片的标记？查看源码发现有个定时任务会检测sharding目录下面的分片节点是不是都属于instances, 如果有一个节点不属于则会设置标记，源码位于org.apache.shardingsphere.elasticjob.lite.internal.reconcile.ReconcileService#runOneIteration。

有段时间刷下面的日志
```
0915:29:44.932 [ReconcileService RUNNING] [WARN ] {"traceId":""}org.apache.shardingsphere.elasticjob.lite.internal.reconcile.ReconcileService[runOneIteration:59]
- Elastic Job: job status node has inconsistent value,start reconciling...

```
结合日志能印证确实是触发了重新分片，但是看zk的数据分片结果是不对的，那就很诧异了，如果没重新分片完成，那一直会等到完成才会继续执行任务（可以看到shardingIfNecessary方法里面有一句blockUntilShardingCompleted），但是目前只有一个节点任务没执行，其他两个节点是能够执行的，说明是分片完成的，那为什么分片完成了还是不正确？

是不是读取了错误的节点信息所以导致写入失败了？阅读源码发现不可能，分片的结果是取instances目录下的，没有结果其他处理。

是不是写完分片结果有其他地方又去变更sharding目录下的数据了，所以和instances目录下的节点不一致，阅读源码发现没其他地方更改。

是不是写完分片结果有其他地方又去变更instances目录下的数据了，所以和sharding目录下的节点不一致？也不可能，变更instances目录下的数据会重新分片，又重新分片任务不会执行，上面已经分析过了。

或者说我查询的zk信息还是在分片中, 分片完成已经一致了，是有其他地方卡住了，比如上一次任务没执行完，或者阻塞了？但当时确实是很长一段时间出现sharding目录下的数据和instances目录下的节点不一致，这段时间重新分片应该早就完成了。

又或者说脑裂了？但是重启节点是刷新的，3个分片节点的数据都在，且3个zk节点的数据都一样。

由于环境清理了也没办法打断点和继续打开debug日志，先记录下，下次继续分析。。。
