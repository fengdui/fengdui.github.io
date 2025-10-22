---
title: "sharding-jdbc最大努力通知事务"
date: "2018-04-17"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

Sharding-JDBC 最大努力型事务理解
- 看的的1.5.4版本 最新的2.x版本
- demo查看
- https://github.com/shardingjdbc/sharding-jdbc/blob/1.5.4/sharding-jdbc-example/sharding-jdbc-example-jdbc-transaction/src/main/java/com/dangdang/ddframe/rdb/sharding/example/transaction/Main.java
- 需要硬编码
- SoftTransactionManager transactionManager=new SoftTransactionManager(getSoftTransactionConfiguration(dataSource));
- transactionManager.init();
- BEDSoftTransaction transaction = (BEDSoftTransaction transactionManager.getTransaction(SoftTransactionType.BestEffortsDelivery);
- 逻辑大致是将执行的sql记录日志到数据库中，失败了在读出来重复执行，成功了删除，直到一定的重试次数
- 如果还是失败 使用elastic-job异步的执行，
- 还失败，保留事务日志，人工处理，