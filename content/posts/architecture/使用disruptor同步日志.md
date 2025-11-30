---
title: "使用disruptor同步日志"
date: "2025-09-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
项目里面需要对数数据库库表列分析 需要加上动态日志进度给客户展示 目前处理到哪一步了
由于是到表的列字段的 如果有几万个库 每个库有上千张表 每个表有上百个字段 那么需要处理的量是非常大的
这些全在内存处理 每处理一步都要进行日志写入
所以使用了disruptor 来同步日志

相比传统的BlockingQueue，Disruptor提供了：
避免锁竞争 无锁设计：避免线程竞争，提高并发性能 能到达百万级每秒
内存预分配：RingBuffer预先分配事件对象，减少GC压力
缓存行友好：优化CPU缓存使用效率
减少上下文切换：更高效的线程间通信机制
背压控制：当RingBuffer满时，会丢弃日志并发出警告，避免系统过载
固定大小的RingBuffer(1024)防止内存无限增长

```
public void afterPropertiesSet() {
    ThreadFactory threadFactory = ThreadUtil.newNamedThreadFactory("job-log-", (ThreadGroup)null, false, (t, e) -> log.warn("【作业日志】发送失败:{}", e.getMessage()));
    Disruptor<LogItemEvent> disruptor = new Disruptor(LogItemEvent::new, 1024, threadFactory, ProducerType.SINGLE, new PhasedBackoffWaitStrategy(0L, 100L, TimeUnit.MILLISECONDS, new BlockingWaitStrategy()));
    disruptor.handleEventsWith(new EventHandler[]{new JobLogSendHandler(this.callbackService), new JobLogPersistentHandler(this.discoveryLogService)});
    disruptor.setDefaultExceptionHandler(new IgnoreExceptionHandler());
    this.ringBuffer = disruptor.start();
}

@EventListener({JobLogEvent.class})
public void jobLog(JobLogEvent event) {
    JobLogItem item = event.getLog();

    try {
        long next = this.ringBuffer.tryNext();
        LogItemEvent data = (LogItemEvent)this.ringBuffer.get(next);
        data.setProcessingTables(event.getProcessingTables());
        data.setLogItem(item);
        this.ringBuffer.publish(next);
    } catch (InsufficientCapacityException var6) {
        log.warn("【作业日志】队列阻塞，丢弃日志: jobId({}) {} {} {} {}", new Object[]{item.getJobId(), item.getTime(), item.getLevel(), item.getMessage(), item.getRemark()});
    }

}
```
handler实现 一个是通过websocket 发送给前端 另外一个是持久化到数据库
```
public class JobLogPersistentHandler implements EventHandler<LogItemEvent> {
    private final DiscoveryLogService discoveryLogService;
    private List<JobLogItem> infoList = new ArrayList();

    public void onEvent(LogItemEvent event, long sequence, boolean endOfBatch) {
        this.infoList.add(event.getLogItem());
        event.clear1();
        if (endOfBatch) {
            this.discoveryLogService.persistentLog(this.infoList);
            this.infoList = new ArrayList();
        }

    }
}
```
