---
title: "rocketmq事务消息"
date: "2016-09-03"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

rocketmq事务消息的理解
- RocketMQ第一阶段发送Prepared消息时，会拿到消息的地址，第二阶段执行本地事物，第三阶段通过第一阶段拿到的地址去访问消息，并修改状态。细心的你可能又发现问题了，如果确认消息发送失败了怎么办？RocketMQ会定期扫描消息集群中的事物消息，这时候发现了Prepared消息，它会向消息发送者确认，Bob的钱到底是减了还是没减呢？如果减了是回滚还是继续发送确认消息呢？RocketMQ会根据发送端设置的策略来决定是回滚还是继续发送确认消息。这样就保证了消息发送与本地事务同时成功或同时失败。
- 如果endTransaction方法执行失败，导致数据没有发送到broker，broker会有回查线程定时（默认1分钟）扫描每个存储事务状态的表格文件，如果是已经提交或者回滚的消息直接跳过，如果是prepared状态则会向Producer发起CheckTransaction请求，Producer会调用DefaultMQProducerImpl.checkTransactionState()方法来处理broker的定时回调请求，而checkTransactionState会调用我们的事务设置的决断方法，最后调用endTransactionOneway让broker来更新消息的最终状态。
- 再回到转账的例子，如果Bob的账户的余额已经减少，且消息已经发送成功，Smith端开始消费这条消息，这个时候就会出现消费失败和消费超时两个问题？解决超时问题的思路就是一直重试，直到消费端消费消息成功，整个过程中有可能会出现消息重复的问题，按照前面的思路解决即可。
- 本质上还是个二阶段提交
- 重复消费幂等性要自己做
- 问题
- 要是本地事务提交，然后挂了，然后确认消息没发送，然后又没法回查确认，导致分布式事务的失败？