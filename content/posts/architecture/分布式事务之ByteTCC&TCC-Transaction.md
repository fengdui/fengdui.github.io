---
title: "分布式事务之ByteTCC&TCC-Transaction"
date: "2018-04-12"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


tcc-transaction
- 查看源码的分支是master-1.2.x 时间2018.04.12
- 也是通过aop拦截业务方法，业务方法上带有Compensable注解
- 1 刚开始是trying阶段，先创建事务日志，状态为trying，然后进入业务方法发起rpc操作，链式的调用提供者的带有Compensable注解的业务方法，当某一个发起者不是root类型，则会使用rpc操作传过来的TransactionContext中的事务id创建事务日志
- 2 当最低层的提供者业务调用完之后，返回到上一层调用者的时候，如果没有错误，这进入confirm阶段，将事务日志状态设置为confirm，然后调用所有使用到的提供者的confirm方法，这里对于每一个发起者调用的提供者方法都会记录在participant中，cancel类似。
- 问题
- 如果trying阶段 过程中发起rpc操作，然后挂了，这时候日志信息状态是trying，但是看事务日志只处理了confirm和cancel。
- 看到第二个分支如果是cancel或者root类型的会rollback。

ByteTCC
- ByteTCC是一个基于TCC（Try/Confirm/Cancel）机制的分布式事务管理器。兼容JTA。