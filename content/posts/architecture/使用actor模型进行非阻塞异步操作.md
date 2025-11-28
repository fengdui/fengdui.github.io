---
title: "使用actor模型进行非阻塞异步操作"
date: "2015-12-18"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


Akka的Actor模型天然支持非阻塞异步处理消息处理  
// 异步方式 - 立即返回，不阻塞  
ioActor.tell(new SendEmailRequest(message), getSelf());  
// 后续代码立即执行，用户体验好  

项目里面主要处理：  
活动预约和审核  
直播流量统计  
邮件通知发送  
余额报警等业务  
这些业务中，比如邮件发送是一个典型的I/O密集型操作，需要异步处理  
如果使用同步方式处理邮件发送，会阻塞后续代码的执行，导致用户体验下降 可能有大量用户同时操作 Akka可以高效处理大量并发消息  

为什么不使用线程池 即使使用线程池 也需要创建线程 某个线程在做业务处理时还需要等待I/O完成  
I/O密集型：主要时间花在网络I/O上 actor模型可以在等待I/O完成时切换到其他消息处理 充分利用CPU资源  
// 将I/O操作委托给专门的I/O Actor或线程池  
ioActor.tell(new SendEmailRequest(message), getSelf());  
有点像python的异步io  