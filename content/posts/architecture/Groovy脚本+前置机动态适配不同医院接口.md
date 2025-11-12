---
title: "Groovy脚本+前置机动态适配不同医院接口"
date: "2022-10-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
公司做了统一的小程序app入口来做医疗的业务 类似于浙里办  
可以选择各种医院去做挂号问诊等医疗业务  
首先接口先打到自己的平台 做一些自己的业务 然后同步到his 每家医院都有自己的his系统 需要适配  
所以出去的时候先使用出口网关  
出口网关使用了groovy脚本去屏蔽不同医院的API差异  
新医院只需编写对应的Groovy脚本即可接入  
脚本可在线编辑、实时生效，无需发版  

```
GroovyClassLoader groovyClassLoader = new GroovyClassLoader(this.getClass().getClassLoader());
Class<?> clazz = groovyClassLoader.parseClass(scriptContent);
```
使用当前类的ClassLoader作为父加载器  
类路径共享 Groovy脚本可以访问所有已加载的Java类  
最后在groovy脚本里面接口打到了前置机  
前置机是部署在医院前端 做了一些接口包装的事情 可以在这里屏蔽医院的不同技术栈 或者做一些接口聚合  
如果在前置机里面做到屏蔽各种医院接口的差异性 对外暴露接口统一 那其实就不用在groovy脚本里面再次做差异化编程了  
