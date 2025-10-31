---
title: "jdk动态代理原理"
date: "2016-06-03"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

* 本质还是反射，动态的生成一个代理类，这个代理类名一般是$Proxy109这种形式，
* 这个类实现你设置的一些接口，并且继承了proxy，proxy有一个InvocationHandler，
* 当调用$Proxy109的某个方法时，他会调用InvocationHandler，传入method，再由InvocationHandler反射调用对应的被代理类那个方法（传入的method），
* method的获取是在一个静态代码块。 生成的逻辑在ProxyClassFactory中，有缓存

* 在系统运行期间设置系统参数（即上面测试中的两个设置）：
* /* 设置此系统属性,让JVM生成的Proxy类写入文件 */
* System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
* //设置将cglib生成的代理类字节码生成到指定位置
* System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:/Temp/code/cglib");
* 还有一种形式，没有测试过：
* 在运行时加入jvm 参数 <code>-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true
* 即我们要在运行当前main方法的路径下创建comsunproxy目录,并创建一个$Proxy0.class文件,才能够正常运行并保存class文件内容。