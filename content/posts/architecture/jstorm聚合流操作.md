---
title: "jstorm聚合流操作"
date: "2018-12-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

数据无限：数据持续流入，没有终点
数据无序：两个流中的数据到达时间可能不一致（例如，订单流先到，用户信息流后到）

两个Spout分别读取两个数据源（如Kafka的topic_a和topic_b）
在声明流时，必须对两个流都使用 fieldsGrouping（域分组），并指定相同的Join键字段（如user_id）。这保证相同键的数据被发送到同一个Bolt任务实例
核心处理单元。内部维护一个缓存结构（如TimeCacheMap）来临时存放来自两个流的Tuple，等待配对
使用TimeCacheMap或其变种，为每个键的数据设置一个超时时间（如30秒）。超时后仍未匹配到的数据将被清理，防止内存泄漏
当来自流A的Tuple到达时，检查缓存中是否有相同键的流B的Tuple：
• 若有，则合并两者数据，触发计算，并向下游输出聚合结果。
• 若没有，则将流A的Tuple存入缓存，等待流B的数据

```
public class JoinBolt extends BaseBasicBolt {
    // 用于缓存流A和流B数据的Map，可设置超时
    private TimeCacheMap<Object, List<Tuple>> streamACache;
    private TimeCacheMap<Object, List<Tuple>> streamBCache;

    @Override
    public void execute(Tuple input, BasicOutputCollector collector) {
        String streamId = input.getSourceStreamId();
        Object joinKey = input.getValueByField("join_key");
        // 根据数据来自哪个流，放入对应的缓存
        if ("stream_a".equals(streamId)) {
            // 1. 先检查B流的缓存里是否有匹配项
            if (streamBCache.containsKey(joinKey)) {
                for (Tuple tupleB : streamBCache.get(joinKey)) {
                    // 2. 匹配成功：合并tuple和tupleB的数据
                    Values joinedResult = joinFunction(input, tupleB);
                    collector.emit(joinedResult);
                }
                streamBCache.remove(joinKey); // 清理已匹配的
            } else {
                // 3. 无匹配：存入A流缓存，等待B流数据
                streamACache.put(joinKey, input);
            }
        }
        // 处理来自stream_b的逻辑与之对称...
    }
}
```
没有深入学习 后续迁入flink