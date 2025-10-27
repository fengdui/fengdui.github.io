---
title: "redis zset延迟队列"
date: "2022-10-09"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


```
import org.apache.commons.collections4.CollectionUtils;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;

import java.util.Set;

public class RedisDelayQueue {

    private RedisTemplate redisTemplate;

    public void addToDelayQueue(String bizId, Long deadTime) {
        try {
            if (bizId == null || deadTime == null) {
                return;
            }
            //倒计时提前1秒钟
            deadTime = deadTime - 1000L;
            ZSetOperations<String, String> zset = redisTemplate.opsForZSet();
            zset.add("waitPayQueueName", bizId, Double.valueOf(deadTime + ""));
        } catch (Exception e) {

        }
    }

    /**
     * 定时执行
     */
    public void execute() {
        ZSetOperations<String, String> zset = redisTemplate.opsForZSet();
        Long count = zset.zCard("waitPayQueueName");
        if (count == null || count <= 0) {
            return;
        }
        long currentMillis = System.currentTimeMillis();
        Set<String> bizIds = zset.rangeByScore("waitPayQueueName", 0, currentMillis);
        if (CollectionUtils.isNotEmpty(bizIds)) {
            //将元素移除队列
            zset.remove("waitPayQueueName", bizIds.toArray(new String[bizIds.size()]));
        } else {

        }
    }
}
```