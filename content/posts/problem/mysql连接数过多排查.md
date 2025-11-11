---
title: "mysql连接数过多排查"
date: "2023-09-14"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
mysql内存飙升
使用下面的命令排查
netstat -nap | grep '13306' | grep -v mysqld | grep EST |awk '{print $7}' | sort |uniq -c  
netstat -nat |awk '{print $6}'|sort|uniq -c|sort -rn  

发现mysql连接数太多 找到对应进程  
然后对应服务看 连接mysql就那么几个地方 一般是线程池 看看线程池怎么用的  
是不是有flway这种工具  

最后发现其他同事写的主从切换组件错了  
同事写了个主从切换的组件 在实现的驱动里面发现当某个连接请求之后得到read-only的报错时 会在启动一个连接 这个连接是连接到另外一个从库
但是没有把之前的连接关闭 导致连接数增加

