---
title: "jrebel热部署"
date: "2018-12-07"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
项目启动太慢 同事建议使用jrebel热部署

Run with JRebel启动  
修改一个简单的Java方法（如返回的字符串），保存文件（Ctrl+S）。如果控制台立刻打印出JRebel重载类的日志，且改动在应用页面即时生效，说明配置成功  
JRebel 通过在JVM层面拦截类的加载过程，并配合其生成的 rebel.xml 配置文件，  
将IDE中编译好的新类字节码直接提供给运行中的应用，从而绕过了传统的容器重启。Spring容器会感知到这些类的变化，并重新注入依赖  