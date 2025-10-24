---
title: "ChainedTransactionManager解决分布式事务"
date: "2021-01-20"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

ChainedTransactionManager（链式事务管理器）的核心作用是：在一个涉及多个异构数据源（或消息队列等资源）的微服务架构中，提供一个"尽力而为"的分布式事务协调机制。

它试图在非XA协议（不依赖两阶段提交）的情况下，解决多个系统之间的数据一致性问题。它是一种最终一致性的解决方案。

比如我一个方法里面既要在A库插入数据 又要在B库插入数据

能够解决当前微服务能够连接所有数据库的情况下的分布式事务问题

他内部包含了一组事务管理器

```xml
<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager" primary="true">
    <constructor-arg name="transactionManagers">
        <list>
            <ref bean="transactionManagerA"/>
            <ref bean="transactionManagerB"/>
            <ref bean="transactionManagerC"/>
        </list>
    </constructor-arg>
</bean>
```

实际用的时候它会按反向顺序打开事务 C B A

然后提交的时候按正向顺序commit A B C

回滚则是反向的 C B A

你需要把最危险的A或者说没办法回滚的 放第一 这样它会先提交 提交失败后面能回滚

由于事务是依次开启和提交的，整个事务的持续时间等于所有单个事务时间的总和，这会导致资源（如数据库连接、锁）被长时间占用，影响系统性能和可扩展性

而且可能会出现数据不一致的情况 比如A B 提交了 C失败了 A B以及提交了 不会回滚了

ChainedTransactionManager 在 Spring Boot 1.x 中存在，但在 Spring Boot 2.x 中已被标记为 @Deprecated（弃用），并最终被移除