---
title: "too many open files"
date: "2024-01-25"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

ulimit -a 查看当前限制  
ulimit -n 开启上限  
使用下面这三个定位进程  
lsof -n | awk '{print$2}' | sort | uniq -c | sort -nr | more  
lsof -n |grep 17092 |sort -k 10,10 |uniq -c | sort -nr  
lsof -n |grep 17092 | awk '{print$10}'| sort |uniq -c | sort -nr  
上面的数量不一定就是fd的数量 可能有很多线程都共用了打开的文件  
可以用 /proc/${PID}/fd 确认一下  
定位到进程了 用这个确认一下  
lsof -p <pid>| awk '{print $4}' |grep "^[0-9]" |wc -l  