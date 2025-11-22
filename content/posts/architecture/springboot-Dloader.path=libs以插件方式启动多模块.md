---
title: "springboot-Dloader.path=libs以插件方式启动多模块"
date: "2017-05-26"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

在Spring Boot应用程序中-Dloader.path=libs用于指定额外的库加载路径，让 Spring Boot 的自定义类加载器能够加载这些目录中的 JAR 包和资源。
当使用 -jar 参数启动 Spring Boot 应用时，默认使用 Spring Boot 的 LaunchedURLClassLoader loader.path 参数告诉这个类加载器额外查找哪些目录中的 JAR 文件
公司将每一个业务模块 即每个菜单下面的功能作为一个model 最后打成一个jar包 每个jar向protal注册自己的菜单 权限 功能点 等等功能
当有新的业务模块时 只需要将新的业务模块的jar包放到libs目录下 然后启动应用即可 不用打整个fatjar
另外一种用法就是 公司将所有项目的依赖二方包三方包都放到libs目录下 每个启动包都瘦身了 都依赖同一个libs目录下的jar包 节约磁盘
