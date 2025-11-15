---
title: "cache aside模式"
date: "2019-06-09"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

cache aside模式  
1.先更新数据库，再更新缓存 X 多线程并发读的时候就能导致写入的是旧值，并且可能没有用到也去缓存了, 写多读少的情况频繁写完全。  
2.先删缓存，再更新数据库 X 多线程并发读写的时候能导致写入的是旧值, 可以延时双删。mysql读写分离的情况下，导致可能查到的是旧值，也可以双删。  
3.先更新数据库，再删缓存 Cache-Aside pattern。删缓存失败可以监听binlog异步重试。  

写key清缓存 同事设置一个key带上超时时间 读服务发现key hit 说明还没有同步 强制写主  
缓存淘汰 先淘汰在写数据库 因为如果先写数据库然后淘汰缓存失败可能导致缓存数据不一致 写淘汰顶多会导致cache miss  
可以专门做异构服务层 去数据从服务层取 对缓存的失效 淘汰等透明  
异步监听缓存变更  
缓存二次淘汰法 以为淘汰缓存只会可能业务还没有做完 比较耗时 数据库的值可以更新 所以读到的还是旧值 这时候在淘汰一次  
cache as sor  
read through  
write throuth  
write behind caching  