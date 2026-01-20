---
title: "参与一次jdk8升级jdk11"
date: "2021-08-09"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
参与一次jdk8升级jdk11  
业务方其实也不需要改什么 一些类库迁移了 需要改下路径 一些类废弃了需要替换 启动参数变更  
jvm底层加密算法支持改变了 升级上来http请求会报错 需要配置下启动参数  
主要是中间件 cicd改的比较多  
mvn process-test-classes org.eclipse.emt4j:emt4j-maven-plugin:0.9-SNAPSHOT:process -DfromVersion=8 -DtoVersion=11 -DoutputFile=report.html  
执行之后的会产出一份报告在工程根目录下report.html  
主要是出了一个g1的问题 频繁oom  
中间件同事没解决 回滚到了cms  
目前就遇到这些问题  