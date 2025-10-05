---
title: "xxljob原理研究"
date: "2018-11-03"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

- xxljob基于quartz的，job的信息还是持久化到mysql中的，调度的时候也是用数据库锁的方式避免重复获取统一任务，然后分发调执行器端，分发使用rpc 方式，本质使用了jetty 执行器作为客户端，执行器启动的时候回向调度中心注册。
- 任务可以在执行器端，使用@JobHandler注解，需要在admin控制台配置一个对应的name，rpc只需传name，
- 然后客户端从spring容器中获取对应的bean只需。任务也可以在调度中心，rpc的时候会将源码传过去编译运行。（这时候依赖的jar包咋办）