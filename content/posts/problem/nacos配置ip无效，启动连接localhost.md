---
title: "nacos配置ip无效 启动连接localhost"
date: "2024-08-12"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

接手的代码没有bootstrap.yml，nacos的地址也写在application.yml，nacos和服务器在一台机子上可以启动，远程则不行。

```js

public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    try {
       ApplicationArguments applicationArguments = new DefaultApplicationArguments(
             args);
       ConfigurableEnvironment environment = prepareEnvironment(listeners,
             applicationArguments);
       configureIgnoreBeanInfo(environment);
       Banner printedBanner = printBanner(environment);
       context = createApplicationContext();
       exceptionReporters = getSpringFactoriesInstances(
             SpringBootExceptionReporter.class,
             new Class[] { ConfigurableApplicationContext.class }, context);
       prepareContext(context, environment, listeners, applicationArguments,
             printedBanner);
       refreshContext(context);
       afterRefresh(context, applicationArguments);
       stopWatch.stop();
       if (this.logStartupInfo) {
          new StartupInfoLogger(this.mainApplicationClass)
                .logStarted(getApplicationLog(), stopWatch);
       }
       listeners.started(context);
       callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
       handleRunFailure(context, ex, exceptionReporters, listeners);
       throw new IllegalStateException(ex);
    }

    try {
       listeners.running(context);
    }
    catch (Throwable ex) {
       handleRunFailure(context, ex, exceptionReporters, null);
       throw new IllegalStateException(ex);
    }
    return context;
}

```

阅读源码发现 prepareEnvironment方法里面就去实例化nacosconfigservice,com.alibaba.cloud.nacos.NacosConfigProperties的postConstruct执行，这时候暂未读取application.yml的配置，只有bootstrap.yml的配置，没有naocs地址的配置，读出来默认是localhost:8848。

prepareContext方法里面会去获取nacos配置，只是连的都是localhost，所以本机可以启动。


```js
public void addLast(PropertySource<?> propertySource) {
    if (logger.isDebugEnabled()) {
       logger.debug("Adding PropertySource '" + propertySource.getName() + "' with lowest search precedence");
    }
    removeIfPresent(propertySource);
    this.propertySourceList.add(propertySource);
}
```


至于application.yml的加载org.springframework.boot.context.config.ConfigFileApplicationListener#postProcessEnvironment 确实是在后面。


最后把nacos的配置提取出来放到bootstrap.yml解决。