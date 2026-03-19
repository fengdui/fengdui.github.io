---
title: "热点key中间件"
date: "2019-08-16"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

数据采集：业务服务器集成的客户端SDK（如HotKey的JdHotKeyStore、TMC的Hermes-SDK）会异步地将每一次key访问事件上报给热点探测服务端集群。这种异步设计保证了上报逻辑不会阻塞业务主线程。  

滑动窗口计算：热点探测服务端（如HotKey的Worker、TMC的Hermes服务端）是核心。它会为每个key维护一个滑动窗口（例如有赞TMC使用包含10个时间片的时间轮，每3秒一个时间片，总计30秒窗口；京东HotKey也有类似实现），来精准计算key在最近一个时间窗口内的访问频次。这比简单地统计每秒访问量能更有效地应对突发流量。  

规则判断与推送：当某个key在滑动窗口内的访问次数超过预设的阈值（例如2秒内达到20次）时，服务端会将其判定为"热key"。随后，它会通过长连接（HotKey使用Netty）或配置中心（TMC使用etcd）将热key毫秒级地推送给集群内的所有业务服务器。  

本地缓存生效：业务服务器收到热key列表后，会将它们加载到本地缓存（如Caffeine或简单的LRU Map）中。后续再有请求访问这些热key时，业务应用会直接从本地内存返回结果，而不再请求后端的Redis或数据库，从而大幅降低对数据层的压力。  

https://gitee.com/jd-platform-opensource/hotkey