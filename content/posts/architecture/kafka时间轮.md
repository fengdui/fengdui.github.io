---
title: "kafka时间轮"
date: "2017-08-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

# 延迟队列 时间轮
- TimingWheel是kafka时间轮的实现，内部包含了一个TimerTaskList数组，每个数组包含了一些链表组成的TimerTaskEntry事件，每个TimerTaskList表示时间轮的某一格，这一格的时间跨度为tickMs，同一个TimerTaskList中的事件都是相差在一个tickMs跨度内的，整个时间轮的时间跨度为interval = tickMs * wheelSize，改时间轮能处理的时间范围在cuurentTime到currentTime + interval之间的事件。
- 当添加一个时间他的超时时间大于整个时间轮的跨度时， expiration >= currentTime + interval，则会将该事件向上级传递，上级的tickMs是下级的interval，传递直到某一个时间轮满足expiration < currentTime + interval，
- 然后计算对应位于哪一格，然后将事件放进去，重新设置超时时间，然后放进jdk延迟队列
```
    else if (expiration < currentTime + interval) {
    // Put in its own bucket
    val virtualId = expiration / tickMs
    val bucket = buckets((virtualId % wheelSize.toLong).toInt)
    bucket.add(timerTaskEntry)
      // Set the bucket expiration time
      if (bucket.setExpiration(virtualId * tickMs)) {
        // The bucket needs to be enqueued because it was an expired bucket
        // We only need to enqueue the bucket when its expiration time has changed, i.e. the wheel has advanced
        // and the previous buckets gets reused; further calls to set the expiration within the same wheel cycle
        // will pass in the same value and hence return false, thus the bucket with the same expiration will not
        // be enqueued multiple times.
        queue.offer(bucket)
      }
```
- SystemTimer会取出queue中的TimerTaskList，根据expiration将currentTime往前推进，然后把里面所有的事件重新放进时间轮中，因为ct推进了，所以有些事件会在第0格，表示到期了，直接返回。
- else if (expiration < currentTime + tickMs) {
- 然后将任务提交到java线程池中处理。
  生产者
- ProducerInterceptor.onSend()方法对消息进行拦截处理
- 调用waitOnMetadata方法获取集群元数据
- 将topic添加到metadata的topics集合中，获取集群中分区数cluster.partitionCountForTopic(topic);，
- 如果不满足则会调用metadata.requestUpdate();将needUpdate设置为true，唤醒sender线程，
- 阻塞直到集群版本号大于当前版本号metadata.awaitUpdate(version, remainingWaitMs);
- 然后返回集群信息
- 序列化key和value
- 调用partition方法选择合适的分区
- 消息没有key则随机选择，有key则使用murmur2hash算出分区
- 调用accumulator.append方法扔进消息累加器，返回RecordAppendResult
- 消息累加器维护了一个map, ConcurrentMap<TopicPartition, Deque<ProducerBatch>>
- 根据消息对应的分区，取出对应的双端队列，没有则创建
- 对队列加锁，取出最后的一块ProducerBatch，然后调用tryAppend追加record，返回不为null得到一个future（FutureRecordMetadata）表示成功，封装成RecordAppendResult返回，然后解锁
- 如果上诉失败，则先申请内存 buffer = free.allocate(size, maxTimeToBlock);
- 然后加锁，这时候也许其他线程已经创建了新的ProducerBatch，所以在做一遍上面的操作（加锁-解锁那一段），如果成功解锁，在finally块释放buffer，否则创建一个ProducerBatch，然后调用tryAppend，将新建的ProducerBatch放到队尾，这时候因为还没有释放锁，所以肯定能加成功。
- 最后返回RecordAppendResult。
- 当我们使用RecordAppendResult.get()的时候会阻塞，什么时候能返回呢？
- FutureRecordMetadata 和ProducerBatch 都引用了同一个ProduceRequestResult，这个ProduceRequestResult维护了一个
- CountDownLatch，RecordAppendResult.get()阻塞，等到当sender线程完成消息的发送，会调用ProducerBatch的done方法，然后调用ProduceRequestResult.done()唤醒（在这之前会先回调callback），这时候使用ProduceRequestResult中的信息封装成RecordMetadata返回。
- 唤醒sender线程，由sender线程发送累加器中缓存的消息
- 如果返回的结果if (result.batchIsFull || result.newBatchCreated) {，则唤醒sender线程
- 返回RecordAppendResult中的future