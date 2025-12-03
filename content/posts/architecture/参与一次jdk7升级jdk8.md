---
title: "参与一次jdk7升级jdk8"
date: "2018-08-20"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

参与一次jdk7升级jdk8  
jvm参数调整 JDK8 用元空间（Metaspace） 取代了永久代（PermGen）。与 -XX:PermSize 和 -XX:MaxPermSize 相关的参数在 JDK8 中被移除。 -XX:MaxMetaspaceSize=256m  
避免使用任何sun.*包  
对于打出的二方包，需要仍旧打成1.7的jar 以免其他服务依赖方还是用1.7的jdk 后续逐步升级到1.8  
其他cicd工具也需要升级到1.8  
docker镜像 也需要升级到1.8  
