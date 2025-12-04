---
title: "rabbitmq延迟队列"
date: "2019-02-22"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
基于死信队列 (TTL + DLX)  
将消息发送到带TTL和死信交换机的队列，过期后转发到处理队列  
```
// 配置延迟队列（前端队列）
@Bean
public Queue orderDelayQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-message-ttl", 30 * 60 * 1000); // TTL: 30分钟
    args.put("x-dead-letter-exchange", "orderDLX"); // 死信交换机
    args.put("x-dead-letter-routing-key", "order.cancel"); // 死信路由键
    return new Queue("order.delay.queue", true, false, false, args);
}
// 配置死信交换机和处理队列
@Bean
public DirectExchange orderDLXExchange() {
    return new DirectExchange("orderDLX");
}
@Bean
public Queue orderProcessQueue() {
    return new Queue("order.process.queue"); // 真正处理超时订单的队列
}
@Bean
public Binding bindingDLX() {
    return BindingBuilder.bind(orderProcessQueue())
                         .to(orderDLXExchange())
                         .with("order.cancel");
}
```
队头堵塞问题：如果第一条消息TTL很长，会阻塞后面TTL短的消息，导致延迟时间不准确