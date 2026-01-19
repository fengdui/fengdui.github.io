---
title: "skywalking 链路日志traceId无法打印问题"
date: "2021-02-17"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
SkyWalking 采用 “基于字节码增强” 的 自动埋点 技术（对于Java等语言），无需修改业务代码。

Java Agent： 在应用启动时，通过 Java Agent 机制，利用 Byte Buddy 或 Javassist 等字节码操作工具， 动态地在关键方法（如 HTTP 请求处理、JDBC 调用、RPC 调用等）的入口和出口处“插入”监控代码。

Trace（追踪）： 一个完整请求链路，由多个 Span 组成。每个 Span 代表一个独立的工作单元（如一次方法调用、一次数据库查询），包含：操作名、开始时间、耗时、标签、日志、状态码等。

Context Propagation（上下文传播）： 这是实现分布式追踪的关键。当请求从服务 A 发往服务 B 时，Agent 会 自动将 Trace ID、Span ID、Endpoint 等信息注入到请求的 HTTP Header 或 RPC 上下文 中。服务 B 的 Agent 接收到请求后，会提取出这些上下文信息，从而将不同服务中的 Span 串联成一条完整的 Trace

日志集成： 通过 将 Trace ID 注入到应用日志 中（如 MDC），SkyWalking 可以将日志与具体的请求链路关联起来。在排查问题时，可以通过 Trace ID 一键查询到该请求的所有相关日志。

现在出现了日志打印的时候获取不到traceId  因为我们开发的agent中 skywalking监听monitor.filter做织入，然后time.filter跟他同级，执行顺序都是0，所以导致随机的可能time.filter在monitor.filter之前，导致获取不到traceId