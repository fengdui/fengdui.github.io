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