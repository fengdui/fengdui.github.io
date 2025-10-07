---
title: "spring 相关的日志无法打印"
date: "2024-06-15"
tags: ["问题"]
ShowToc: false
TocOpen: false
---


1.spring 相关的日志无法打印。
即图中spring相关的日志没打印

![img_8.png](img_8.png)

![img_9.png](img_9.png)


升级解决，spring日志模块低版本适配各开源实现的优先级不一样，这个版本才是优先选择slf4j。

```
<dependency> 
    <groupId>org.springframework</groupId> 
    <artifactId>spring-jcl</artifactId> 
    <version>5.3.25</version> 
    <scope>compile</scope> 
</dependency>

```