---
title: "liquibase管理数据库版本"
date: "2025-10-14"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

新项目使用了liquibase来管理数据库版本  
要注意下新的sql文件不要drop table 因为这是增量的场景会导致数据丢失  
sql要注意幂等，即多次执行sql文件的结果和执行一次的结果是一样的。  
或者整个文件幂等 中间失败了回归整个文件 也可以达到幂等  
多个java服务启动它会加锁  
有没有执行某个sql文件 可以查看change_log表 里面是否有记录  

当首次执行的时候前面一个文件create table 后面一个文件drop在create当然没问题 因为没数据 实际上这样写不对  
不用刻意区分全量还是增量 因为liquibase会记录每个sql文件是否执行 没执行就执行  
