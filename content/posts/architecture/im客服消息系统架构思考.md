---
title: "im客服消息系统架构思考"
date: "2019-07-31"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

需求背景: 买家能够和商家客服进行沟通咨询  

客户通过微信公众号、小程序、网页等等多个渠道发送消息 这里要对接各种开发平台 做一个消息接入层  
后端接到消息之后进行客服分配 如果没有会话 建立会话 已存在就加入 刷新会话状态时间等 一个客户在一个店铺里面对应一个会话  
如果没有可用客服 进入等待队列 持久化到数据库 下次客服登录获取队列的待分配会话 加入会话 会话维护在kv数据库里面 no schema比较好维护  
会话包括了客户id 客服id等等信息  
然后推送消息 出发自动回复 机器人 快速问答 敏感词过滤等等  
客服收到消息进行回复 出去也可能是微信公众号、小程序、网页等等多个渠道  
消息是存储的架构是hbase+es  
提供反向索引搜索  
hbase借助rowkey进行顺序查询 聊天记录是不会变更的 hbase很适合 rowkey的设计要注意 避免热点问题  
conversationId: "shop123#fuser456" 会话id是 店铺id#用户id  
msgId: 100  消息id  
userid: "456" 用户id  
hbase rowkey 用户id反转#会话id#Integer.MAX_VALUE - msgId  
RowKey: "654#shop123#user456#2147483547"  
用户id反转后相同前缀的用户分散到不同Region id自增一般都需要这样做一下  
Integer.MAX_VALUE - msgId  实现时间倒序排列 msgId递增，减法后变为递减，最新消息排在前面  

es使用消息id作为唯一id 去重  
注意设计路由策略 比如es是给商家查询的 那这个索引同商家的要路由到一个分片上面 减少跨分片查询  

接入层 即netty 实现websocket  
当业务复杂的时候 需要引入接入层  接入层和后端业务分离 这样扩容和缩容 各自上线可以单独做 将技术和业务分类  
3个connectionId  
connectionIdA 接入层维护 表示客户端到接入层的连接  
connectionIdB 接入层维护 表示接入层到后端业务的连接  
connectionIdC 后端业务维护 接入层到后端业务的连接  
一个客户端请求过来 先打到接入层 创建connectionIdA 建立connectionIdA ->channel的关系  
接入层选择后端业务节点 生成connectionIdB 建立connectionIdB->channel的关系 请求打到后端的包上面带上connectionIdA  
后端生成和接入层的连接 生成connectionIdC 建立connectionIdC->channel的关系  
然后生成回复消息进行回复 服务端通过connectionIdC拿到channel 消息被发送到了接入层  
接入层怎么知道发往哪个客户端呢 通过包里面的connectionIdA拿到channel发往客户端  

连接id是技术上的通道 当发送消息的时候怎么知道该发送给谁 这个人的通道是什么  
所以在用户登录的时候 需要维护userId和connectionId的关系 如果根据userId找不到connectionId 说明用户不在线  


```
会话创建完成
    ↓
消息分流处理 handleAllocation()
    ↓
检查会话中在线客服
    ├── 有在线客服 → 直接推送消息
    └── 无在线客服 → 执行分配策略
        ├── 获取可分配客服列表
        ├── 执行负载均衡算法
        └── 分配结果处理
            ├── 分配成功
            │   ├── 更新会话成员
            │   ├── 推送消息给客服
            │   ├── 创建咨询会话
            │   ├── 保存灰字消息
            │   ├── 发送通知消息
            │   └── 记录统计事件
            └── 分配失败
                ├── 加入排队列表
                ├── 保存排队消息
                ├── 发送排队通知
                └── 记录排队事件
    ↓
消息推送处理
    ├── 更新活跃会话
    ├── 处理离线策略
    └── 写入消息队列
    ↓
客服端收到消息/通知

```

