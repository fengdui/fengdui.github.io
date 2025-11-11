---
title: "nginx time_wait过多"
date: "2023-10-08"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

模拟高并发的场景
短时间后，所有的 TIME_WAIT 全都消失，被回收，端口包括服务，均正常。

即，在高并发的场景下，TIME_WAIT 连接存在，属于正常现象。

线上场景中，持续的高并发场景

一部分 TIME_WAIT 连接被回收，但新的 TIME_WAIT 连接产生；

一些极端情况下，会出现大量的 TIME_WAIT

大量的 TIME_WAIT 状态 TCP 连接存在，其本质原因是什么？

大量的短连接存在

特别是 HTTP 请求中，如果 connection 头部取值被设置为 close 时，基本都由「服务端」发起主动关闭连接

而，TCP 四次挥手关闭连接机制中，为了保证 ACK 重发和丢弃延迟数据，设置 time_wait 为 2 倍的 MSL（报文最大存活时间）

TIME_WAIT 状态：

TCP 连接中，主动关闭连接的一方出现的状态；（收到 FIN 命令，进入 TIME_WAIT 状态，并返回 ACK 命令）

保持 2 个 MSL 时间，即，4 分钟；（MSL 为 2 分钟）

解决办法
解决上述 time_wait 状态大量存在，导致新连接创建失败的问题，一般解决办法：
1、客户端，HTTP 请求的头部，connection 设置为 keep-alive，保持存活一段时间：现在的浏览器，一般都这么进行了
2、服务器端， 允许 time_wait 状态的 socket 被重用 缩减 time_wait 时间，设置为 1 MSL（即，2 mins）