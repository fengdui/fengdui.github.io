---
title: "apollo配置中心"
date: "2018-05-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


* 创建一个app(项目)的时候，portal模块默认创建一个appnamespace，config模块创建一个默认集群，一个appnamespace和一个namespace（是根据appnamespace创建的，即取出所有的appnamespace，然后创建对应的namespace）
* 创建集群的时候，会根据app下所有的appnamespace创建对应的namespace。
* 创建私有或公有的非关联的namespace，也会根据所有的集群创建对应的namespace。
* 创建关联的namespace，只会在namespace表中记录。
* 配置项记录在item表中。
* 配置发布adminservice会记录消息在表中，然后configservice会轮询表，发现消息，通知客户端（长轮询）
* 依赖了mysql，用到了spring boot， spring cloud eureka，cat等。





