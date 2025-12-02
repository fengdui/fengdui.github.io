---
title: "类加载器解决jar包冲突"
date: "2019-08-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
Jar包多版本冲突，如log4j、guava等  
Jar包升级问题  
升级周期长，当二方包新版本发布时，应用无法及时升级，有很多项目过了几个月甚至一年都没更新，存在隐患；  
升级成本高，新版本发布后，相关项目都得一个个去升级，项目负责人和测试都得跟进去升级、去测试，重复工作，人员成本高。  
Jar包无统一管理  
多个jar包版本之间的配合怎么样最合理，这个目前未知，让每个开发同学都去探索研究不合适，如log4j和slf4j的搭配；  
目前每个项目开发大家都是根据自己的需要去依赖第三方jar包，很多jar可能存在风险，无法验证，也无法统一。如json相关的jar包有很多种，如果统一的话，出问题就很好定位。  

OSGI——开发模式将切底改变，成本很高  
Java9——要求所有java版本升级，并且开发模式也将切底改变  
Maven方式，如发布版本为snapshot或version为RELEASE等——容易出错，不成体系  
自定义classloader——相对比较符合要求  

Java的类加载遵循双亲委派的设计模式，从App ClassLoader开始自底向上寻找，并自顶向下加载，所以在没有自定义classloader时，应用的启动是通过App ClassLoader去加载Main启动类去运行。  
自定义ClassLoader后，系统ClassLoader将被设置成容器自定义的classLoader，自定义ClassLoader重新去加载Main启动类运行，此时后续所有的类加载都会先去自定义的classLoader里查找。   
应用默认系统类加载器是AppClassLoader，在new对象时不会经过自定义的classLoader。  
巧妙之处：Main函数启动时，AppClassLoader加载main和容器，容器获取到Main class，用自定义classLoader重复加载main，设置系统类加载器为自定义类加载器，此时new对象都会经过自定义的classLoader。  

暴露出每个模块的类到文件里面 然后判断类应该在哪个模块里面在去使用对应的类加载器

蚂蚁金服的业务系统模块化之模块化隔离方案 https://www.sofastack.tech/blog/sofastack-modular-isolation/  
SOFAArk介绍 https://www.sofastack.tech/projects/sofa-boot/sofa-ark-readme/  
Java隔离容器之sofa-ark使用说明及源码解析 https://juejin.cn/post/6844903653828984845
OSGi原理与最佳实践 http://www.osgi.com.cn/article/7289456