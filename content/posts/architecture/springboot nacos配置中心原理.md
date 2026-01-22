---
title: "springboot nacos配置中心原理"
date: "2023-07-28"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
引导上下文（Bootstrap Context）   
这是一个先于主应用上下文（ApplicationContext） 创建的、独立的Spring上下文。它的唯一任务就是加载外部配置。  
BootstrapApplicationListener：SpringApplication.run() 启动早期会发布 ApplicationEnvironmentPreparedEvent 事件。BootstrapApplicationListener 监听到此事件后，创建引导上下文。  
引导环境准备 BootstrapServiceContext：在引导上下文刷新时，会调用所有实现了 PropertySourceLocator 接口的Bean。Nacos客户端通过自动配置，将 NacosPropertySourceLocator 注册到了这个上下文中。  
拉取与注入：NacosPropertySourceLocator.locate() 被调用，执行上述的远程拉取逻辑。获取的配置被包装后，立即添加到当前 Environment，供后续的Bean初始化使用。  
配置加载时机：正因为引导上下文创建得更早，所以能在主应用读取任何application.yml或创建任何业务Bean之前，就从配置中心拿到配置。  
配置加载与优先级  
主动拉取：在引导上下文中，配置中心客户端（如ConfigService）会根据bootstrap.yml中定义的data-id、group等信息，主动发起请求到远程配置中心拉取配置。  
高优先级：这些从配置中心获取的属性，会以高优先级（通常最高）被添加到Spring的 Environment 中。这意味着，它们会覆盖本地application.yml中的同名属性。  
属性源（PropertySource）机制  
Spring使用 PropertySource 来管理不同来源的配置。启动后，你会看到 Environment 中有一个类似 NacosPropertySource 的属性源，它里面就存放着从Nacos拉取的所有配置。你可以通过 @Value 或 @ConfigurationProperties 直接注入这些属性。  
