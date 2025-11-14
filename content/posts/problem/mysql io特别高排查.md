---
title: "mysql io特别高排查"
date: "2023-05-10"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
dstat命令查看，会有20秒左右持续的每秒50m的写入  
通过iostat查看到是vda这个磁盘设备的写入很大（39-60M每秒）  
执行pidstat -d  1 20查看具体是哪个服务导致的io高  
用iotop可以看到对应的mysql线程id4340占用的IO特别高  
执行查询语句 select PROCESSLIST_ID, THREAD_OS_ID, PROCESSLIST_INFO  from performance_schema.threads where THREAD_OS_ID = 4340;\G  
可以看到占用IO高的sql  
复杂sql可能会导致mysql生成临时表  
sql导致cpu过高也可以通过show processlist查看
