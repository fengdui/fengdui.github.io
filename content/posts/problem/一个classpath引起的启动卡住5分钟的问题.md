---
title: "一个classpath引起的启动卡住5分钟的问题"
date: "2024-12-12"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

线上出现了一个问题，项目启动会突然卡住，然后无日志，卡住的是main线程，那就是spring启动的时候做了一些时间很慢，于是加上debug日志，发现这行日志后面就卡住了很久才继续的打日志。

![classpath扫描卡住.png](../../pic/classpath扫描卡住.png)
这日志说明启动的时候扫描了整个classpath\*,去获取了某些东西，这个日志是PathMatchingResourcePatternResolver这个类里面打的，结合日志里面他是 resolved "classpath\*:com/\*\*/\*/class", 于是本地打断点调试，发现是有一个二方包里面他去扫描了获取某个注解的类。。

先是修改成了"classpath和:com/abc/\*\*/\*/class 但是这样扫描出来的是空的

再是修改成了"classpath\*:com/abc/\*\*/\*/class 有效果

classpath和classpath\*的区别在哪 跟踪了一下源码
org.springframework.core.io.support.PathMatchingResourcePatternResolver#findPathMatchingResources

发现classpath底层用了classLoader.getResource，只能获取匹配的第一个jar包，所以扫出来是空的，而classpath底层用了classLoader.getResources，扫描的是匹配的所有jar包，所以能得到匹配的所有类。
