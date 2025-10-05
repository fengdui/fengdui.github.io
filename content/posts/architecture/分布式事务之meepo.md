---
title: "分布式事务之meepo"
date: "2018-04-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---



- meepo是阿里GTS的一个开源实现，基于byteJta,byteJta是一个典型的两阶段提交协议，类XA/2PC（1pc+1）机制的分布式事务管理器。实现了Spring JTA接口，集成dubbo，提供阿里GTS的核心功能。
