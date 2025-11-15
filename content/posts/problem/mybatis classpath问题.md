---
title: "mybatis classpath问题"
date: "2025-05-15"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

![img_34.png](/pic/img_34.png)
把mybatis的druid配置从common模块移到dao模块
dao模块下面 classpath:mapper/*.xml 的xml就加载不到了
mapper文件本来就在dao模块里面

改成classpath:mapper/*.xml 就可以了

理解是不带星号只找第一个
这边引入了一个二方包 他就只扫描新的二方包里面的

```
at org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:177) ~[spring-beans-5.0.8.RELEASE.jar!/:5.0.8.RELEASE]
        at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:91) ~[spring-beans-5.0.8.RELEASE.jar!/:5.0.8.RELEASE]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessPropertyValues(CommonAnnotationBeanPostProcessor.java:318) ~[spring-context-5.0.8.RELEASE.jar!/:5.0.8.RELEASE]
        ... 25 more
Caused by: java.lang.NoSuchMethodError: org.apache.ibatis.annotations.Insert.databaseId()Ljava/lang/String;
        at com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder$AnnotationWrapper.<init>(MybatisMapperAnnotationBuilder.java:692) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder.lambda$getAnnotationWrapper$4(MybatisMapperAnnotationBuilder.java:654) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:193) ~[?:1.8.0_411]
        at java.util.Spliterators$ArraySpliterator.forEachRemaining(Spliterators.java:948) ~[?:1.8.0_411]
        at java.util.stream.ReferencePipeline$Head.forEach(ReferencePipeline.java:580) ~[?:1.8.0_411]
        at java.util.stream.ReferencePipeline$7$1.accept(ReferencePipeline.java:270) ~[?:1.8.0_411]
        at java.util.HashMap$KeySpliterator.forEachRemaining(HashMap.java:1580) ~[?:1.8.0_411]
        at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:482) ~[?:1.8.0_411]
        at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:472) ~[?:1.8.0_411]
        at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708) ~[?:1.8.0_411]
        at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234) ~[?:1.8.0_411]
        at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499) ~[?:1.8.0_411]
        at com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder.getAnnotationWrapper(MybatisMapperAnnotationBuilder.java:655) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder.parseStatement(MybatisMapperAnnotationBuilder.java:295) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder.parse(MybatisMapperAnnotationBuilder.java:113) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.core.MybatisMapperRegistry.addMapper(MybatisMapperRegistry.java:83) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.core.MybatisConfiguration.addMapper(MybatisConfiguration.java:152) ~[mybatis-plus-core-3.4.2.jar!/:3.4.2]
        at org.apache.ibatis.builder.xml.XMLMapperBuilder.bindMapperForNamespace(XMLMapperBuilder.java:436) ~[mybatis-3.5.2.jar!/:3.5.2]
        at org.apache.ibatis.builder.xml.XMLMapperBuilder.parse(XMLMapperBuilder.java:96) ~[mybatis-3.5.2.jar!/:3.5.2]
        at com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean.buildSqlSessionFactory(MybatisSqlSessionFactoryBean.java:593) ~[mybatis-plus-extension-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean.afterPropertiesSet(MybatisSqlSessionFactoryBean.java:431) ~[mybatis-plus-extension-3.4.2.jar!/:3.4.2]
        at com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean.getObject(MybatisSqlSessionFactoryBean.java:628) ~[mybatis-plus-extension-3.4.2.jar!/:3.4.2]
        at com.hzmc.capaa.mgrs.common.config.DruidDataSourceConfig.masterSqlSessionFactory(DruidDataSourceConfig.java:163) ~[mgrs-common-1.0.0-SANPSHOT.jar!/:1.0.0-SANPSHOT]
        at com.hzmc.capaa.mgrs.common.config.DruidDataSourceConfig$$EnhancerBySpringCGLIB$$931ac066.CGLIB$masterSqlSessionFactory$0(<generated>) ~[mgrs-common-1.0.0-SANPSHOT.jar!/:1.0.0-SANPSHOT]
        at com.hzmc.capaa.mgrs.common.config.DruidDataSourceConfig$$EnhancerBySpringCGLIB$$931ac066$$FastClassBySpringCGLIB$$190f310e.invoke(<generated>) ~[mgrs-common-1.0.0-SANPSHOT.jar!/:1.0.0-SANPSHOT]
        at org.springframework.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:228) ~[spring-core-5.0.8.RELEASE.jar!/:5.0.8.RELEASE]
```