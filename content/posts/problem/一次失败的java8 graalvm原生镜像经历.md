---
title: "一次失败的java8 graalvm原生镜像经历"
date: "2025-11-26"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
spring boot 2.4.13 
jdk使用graalvm-ce-java8-linux-amd64-21.3.1.tar 支持java8
插件配置如下
```
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <version>0.9.18</version>
    <configuration>
      <mainClass>com.utopia.factory.web.Application</mainClass>
      <imageName>brobot-factory</imageName>
      <useArgFile>false</useArgFile>  <!-- 关键：启用参数文件 -->
      <buildArgs>
        <buildArg>--no-fallback</buildArg>
        <buildArg>-H:+ReportExceptionStackTraces</buildArg>
        <buildArg>--verbose</buildArg>
        <buildArg>-cp</buildArg>
        <buildArg>${project.build.outputDirectory}:${project.build.directory}/lib/*</buildArg>
        <buildArg>
          --initialize-at-run-time=org.slf4j,org.apache.logging,org.apache.commons.logging,net.sf.json,net.sf.ezmorph,io.netty,com.alibaba.fastjson,redis.clients.jedis,org.apache.commons.pool2,org.codehaus.groovy,groovy
        </buildArg>
      </buildArgs>
    </configuration>
    <executions>
      <execution>
        <id>build-native</id>
        <goals>
          <goal>build</goal>
        </goals>
        <phase>package</phase>
      </execution>
    </executions>
  </plugin>
```
下面是踩坑记录
linux识别不到@符号 改成这个<useArgFile>false</useArgFile> 不生成文件
linux用native插件 springboot2.7出现com.oracle.svm.core.util.UserError$UserException: Main entry point class 'com.factory.web.Application' not found.
graalvm需要找target/classes下面的启动类
配置加上<buildArg>${project.build.outputDirectory}:${project.build.directory}/lib/*</buildArg>
```
Error: com.oracle.graal.pointsto.constraints.UnresolvedElementException: Discovered unresolved type during parsing: org.springframework.boot.SpringApplication. To diagnose the issue you can use the --allow-incomplete-classpath option. The missing type is then reported at run time when it is accessed the first time.
Detailed message:
Trace: 
        at parsing com.utopia.factory.web.Application.main(Application.java:18)
Call path from entry point to com.oracle.svm.core.JavaMainWrapper.runCore(): 
        at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:126)
        at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:183)

com.oracle.svm.core.util.UserError$UserException: com.oracle.graal.pointsto.constraints.UnresolvedElementException: Discovered unresolved type during parsing: org.springframework.boot.SpringApplication. To diagnose the issue you can use the --allow-incomplete-classpath option. The missing type is then reported at run time when it is accessed the first time.
Detailed message:
Trace: 
        at parsing com.utopia.factory.web.Application.main(Application.java:18)
Call path from entry point to com.oracle.svm.core.JavaMainWrapper.runCore(): 
        at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:126)
        at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:183)

        at com.oracle.svm.core.util.UserError.abort(UserError.java:87)
        at com.oracle.svm.hosted.FallbackFeature.reportAsFallback(FallbackFeature.java:233)
        at com.oracle.svm.hosted.NativeImageGenerator.runPointsToAnalysis(NativeImageGenerator.java:759)
        at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:529)
        at com.oracle.svm.hosted.NativeImageGenerator.run(NativeImageGenerator.java:488)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.buildImage(NativeImageGeneratorRunner.java:403)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.build(NativeImageGeneratorRunner.java:569)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.main(NativeImageGeneratorRunner.java:122)
Caused by: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: com.oracle.graal.pointsto.constraints.UnresolvedElementException: Discovered unresolved type during parsing: org.springframework.boot.SpringApplication. To diagnose the issue you can use the --allow-incomplete-classpath option. The missing type is then reported at run time when it is accessed the first time.
Detailed message:
Trace: 
        at parsing com.utopia.factory.web.Application.main(Application.java:18)
Call path from entry point to com.oracle.svm.core.JavaMainWrapper.runCore(): 
        at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:126)
        at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:183)

        at com.oracle.graal.pointsto.constraints.UnsupportedFeatures.report(UnsupportedFeatures.java:126)
        at com.oracle.svm.hosted.NativeImageGenerator.runPointsToAnalysis(NativeImageGenerator.java:756)
        ... 5 more
Caused by: com.oracle.graal.pointsto.constraints.UnresolvedElementException: Discovered unresolved type during parsing: org.springframework.boot.SpringApplication. To diagnose the issue you can use the --allow-incomplete-classpath option. The missing type is then reported at run time when it is accessed the first time.
        at com.oracle.svm.hosted.phases.SharedGraphBuilderPhase$SharedBytecodeParser.reportUnresolvedElement(SharedGraphBuilderPhase.java:307)
        at com.oracle.svm.hosted.phases.SharedGraphBuilderPhase$SharedBytecodeParser.handleUnresolvedType(SharedGraphBuilderPhase.java:263)
        at com.oracle.svm.hosted.phases.SharedGraphBuilderPhase$SharedBytecodeParser.handleUnresolvedMethod(SharedGraphBuilderPhase.java:289)
        at com.oracle.svm.hosted.phases.SharedGraphBuilderPhase$SharedBytecodeParser.handleUnresolvedInvoke(SharedGraphBuilderPhase.java:252)
        at org.graalvm.compiler.java.BytecodeParser.genInvokeStatic(BytecodeParser.java:1677)
        at org.graalvm.compiler.java.BytecodeParser.genInvokeStatic(BytecodeParser.java:1652)
        at org.graalvm.compiler.java.BytecodeParser.processBytecode(BytecodeParser.java:5419)
        at org.graalvm.compiler.java.BytecodeParser.iterateBytecodesForBlock(BytecodeParser.java:3477)
        at org.graalvm.compiler.java.BytecodeParser.handleBytecodeBlock(BytecodeParser.java:3437)
        at org.graalvm.compiler.java.BytecodeParser.processBlock(BytecodeParser.java:3282)
        at org.graalvm.compiler.java.BytecodeParser.build(BytecodeParser.java:1145)
        at org.graalvm.compiler.java.BytecodeParser.buildRootMethod(BytecodeParser.java:1030)
        at org.graalvm.compiler.java.GraphBuilderPhase$Instance.run(GraphBuilderPhase.java:84)
        at com.oracle.svm.hosted.phases.SharedGraphBuilderPhase.run(SharedGraphBuilderPhase.java:81)
        at org.graalvm.compiler.phases.Phase.run(Phase.java:49)
        at org.graalvm.compiler.phases.BasePhase.apply(BasePhase.java:212)
        at org.graalvm.compiler.phases.Phase.apply(Phase.java:42)
        at org.graalvm.compiler.phases.Phase.apply(Phase.java:38)
        at com.oracle.graal.pointsto.flow.AnalysisParsedGraph.parseBytecode(AnalysisParsedGraph.java:132)
        at com.oracle.graal.pointsto.meta.AnalysisMethod.ensureGraphParsed(AnalysisMethod.java:616)
        at com.oracle.graal.pointsto.phases.InlineBeforeAnalysisGraphDecoder.lookupEncodedGraph(InlineBeforeAnalysis.java:182)
        at org.graalvm.compiler.replacements.PEGraphDecoder.doInline(PEGraphDecoder.java:1120)
        at org.graalvm.compiler.replacements.PEGraphDecoder.tryInline(PEGraphDecoder.java:1103)
        at org.graalvm.compiler.replacements.PEGraphDecoder.trySimplifyInvoke(PEGraphDecoder.java:961)
        at org.graalvm.compiler.replacements.PEGraphDecoder.handleInvoke(PEGraphDecoder.java:915)
        at org.graalvm.compiler.nodes.GraphDecoder.processNextNode(GraphDecoder.java:791)
        at com.oracle.graal.pointsto.phases.InlineBeforeAnalysisGraphDecoder.processNextNode(InlineBeforeAnalysis.java:242)
        at org.graalvm.compiler.nodes.GraphDecoder.decode(GraphDecoder.java:532)
        at org.graalvm.compiler.replacements.PEGraphDecoder.decode(PEGraphDecoder.java:787)
        at com.oracle.graal.pointsto.phases.InlineBeforeAnalysis.decodeGraph(InlineBeforeAnalysis.java:99)
        at com.oracle.graal.pointsto.flow.MethodTypeFlowBuilder.parse(MethodTypeFlowBuilder.java:171)
        at com.oracle.graal.pointsto.flow.MethodTypeFlowBuilder.apply(MethodTypeFlowBuilder.java:321)
        at com.oracle.graal.pointsto.flow.MethodTypeFlow.createTypeFlow(MethodTypeFlow.java:293)
        at com.oracle.graal.pointsto.flow.MethodTypeFlow.ensureTypeFlowCreated(MethodTypeFlow.java:282)
        at com.oracle.graal.pointsto.flow.MethodTypeFlow.addContext(MethodTypeFlow.java:103)
        at com.oracle.graal.pointsto.flow.StaticInvokeTypeFlow.update(InvokeTypeFlow.java:420)
        at com.oracle.graal.pointsto.PointsToAnalysis$2.run(PointsToAnalysis.java:595)
        at com.oracle.graal.pointsto.util.CompletionExecutor.executeCommand(CompletionExecutor.java:188)
        at com.oracle.graal.pointsto.util.CompletionExecutor.lambda$executeService$0(CompletionExecutor.java:172)
        at java.util.concurrent.ForkJoinTask$RunnableExecuteAction.exec(ForkJoinTask.java:1402)
        at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
        at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1056)
        at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1692)
        at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:175)
[brobot-factory:65898]      [total]:   8,305.41 ms,  1.54 GB
# Printing build artifacts to: /root/test/brobot-factory/brobot-factory-web/target/brobot-factory.build_artifacts.txt
Error: Image build request failed with exit status 1
com.oracle.svm.driver.NativeImage$NativeImageError: Image build request failed with exit status 1
        at com.oracle.svm.driver.NativeImage.showError(NativeImage.java:1762)
        at com.oracle.svm.driver.NativeImage.build(NativeImage.java:1473)
        at com.oracle.svm.driver.NativeImage.performBuild(NativeImage.java:1434)
        at com.oracle.svm.driver.NativeImage.main(NativeImage.java:1421)
```
使用spring native和 aot插件解决 版本是0.9.2 下载不到使用阿里云的镜像仓库grails-core
原来spring boot fatjar的方式不行 需要把将fatjar里面的依赖包放到lib目录下 再去依赖<buildArg>${project.build.outputDirectory}:${project.build.directory}/lib/*</buildArg>
```
Error: Could not find target method: private java.util.List org.springframework.nativex.substitutions.boot.Target_FailureAnalyzers.loadFailureAnalyzers(org.springframework.context.ConfigurableApplicationContext,java.lang.ClassLoader)
com.oracle.svm.core.util.UserError$UserException: Could not find target method: private java.util.List org.springframework.nativex.substitutions.boot.Target_FailureAnalyzers.loadFailureAnalyzers(org.springframework.context.ConfigurableApplicationContext,java.lang.ClassLoader)
        at com.oracle.svm.core.util.UserError.abort(UserError.java:73)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.findOriginalMethod(AnnotationSubstitutionProcessor.java:742)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.handleMethodInAliasClass(AnnotationSubstitutionProcessor.java:368)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.handleAliasClass(AnnotationSubstitutionProcessor.java:340)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.handleClass(AnnotationSubstitutionProcessor.java:310)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.init(AnnotationSubstitutionProcessor.java:266)
        at com.oracle.svm.hosted.NativeImageGenerator.createDeclarativeSubstitutionProcessor(NativeImageGenerator.java:912)
        at com.oracle.svm.hosted.NativeImageGenerator.setupNativeImage(NativeImageGenerator.java:839)
        at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:527)
        at com.oracle.svm.hosted.NativeImageGenerator.run(NativeImageGenerator.java:488)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.buildImage(NativeImageGeneratorRunner.java:403)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.build(NativeImageGeneratorRunner.java:569)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.main(NativeImageGeneratorRunner.java:122)
```
这是Spring Native版本兼容性问题。Spring Native 0.9.2与你使用的Spring Boot 2.7.18和GraalVM版本不兼容。
Spring 版本（比如 Spring 5+）在 BeanUtils 里硬加了 Kotlin 兼容代码”，GraalVM 编译原生镜像时会扫描到这些代码，但你的项目没引 Kotlin 依赖，所以报 “找不到 KotlinDelegate 方法”。
引入kotlin依赖
出现问题
```
Error: Could not find target method: private java.util.List org.springframework.nativex.substitutions.boot.Target_FailureAnalyzers.loadFailureAnalyzers(org.springframework.context.ConfigurableApplicationContext,java.lang.ClassLoader)
com.oracle.svm.core.util.UserError$UserException: Could not find target method: private java.util.List org.springframework.nativex.substitutions.boot.Target_FailureAnalyzers.loadFailureAnalyzers(org.springframework.context.ConfigurableApplicationContext,java.lang.ClassLoader)
        at com.oracle.svm.core.util.UserError.abort(UserError.java:73)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.findOriginalMethod(AnnotationSubstitutionProcessor.java:742)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.handleMethodInAliasClass(AnnotationSubstitutionProcessor.java:368)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.handleAliasClass(AnnotationSubstitutionProcessor.java:340)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.handleClass(AnnotationSubstitutionProcessor.java:310)
        at com.oracle.svm.hosted.substitute.AnnotationSubstitutionProcessor.init(AnnotationSubstitutionProcessor.java:266)
        at com.oracle.svm.hosted.NativeImageGenerator.createDeclarativeSubstitutionProcessor(NativeImageGenerator.java:912)
        at com.oracle.svm.hosted.NativeImageGenerator.setupNativeImage(NativeImageGenerator.java:839)
        at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:527)
        at com.oracle.svm.hosted.NativeImageGenerator.run(NativeImageGenerator.java:488)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.buildImage(NativeImageGeneratorRunner.java:403)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.build(NativeImageGeneratorRunner.java:569)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.main(NativeImageGeneratorRunner.java:122)
```
Spring Native 的替换机制与 Spring Boot 版本不匹配。 替换机制（Substitution）：GraalVM原生镜像不支持所有的Java特性，因此Spring Native会使用一些以Target_开头的替换类，在构建时替换掉那些无法直接编译为原生镜像的Spring内部类。 方法缺失：替换类Target_FailureAnalyzers试图替换原始Spring Boot类中一个名为loadFailureAnalyzers的私有方法。但您项目中的Spring Boot版本，这个方法可能已经不存在、方法签名发生了变化，或者已经被移动到了其他类中，导致GraalVM在构建时找不到目标方法
Spring Native是实验性质的项目 也不知道是哪个版本能够兼容 spring 2.7.18 但是0.9.2spring native支持springboot 2.4.13 于是先把springboot版本降低一下
```
Error: Classes that should be initialized at run time got initialized during image building:
 io.netty.util.AbstractReferenceCounted the class was requested to be initialized at run time (from jar:file:///home/aaa/m2/io/netty/netty-common/4.1.70.Final/netty-common-4.1.70.Final.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted' and from jar:file:///home/aaa/m2/com/alibaba/nacos/nacos-client/2.1.2/nacos-client-2.1.2.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted' and from jar:file:///home/aaa/test/brobot-factory/brobot-factory-web/target/lib/nacos-client-2.1.2.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted' and from jar:file:///home/aaa/test/brobot-factory/brobot-factory-web/target/lib/netty-common-4.1.70.Final.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted'). To see why io.netty.util.AbstractReferenceCounted got initialized use --trace-class-initialization=io.netty.util.AbstractReferenceCounted

com.oracle.svm.core.util.UserError$UserException: Classes that should be initialized at run time got initialized during image building:
 io.netty.util.AbstractReferenceCounted the class was requested to be initialized at run time (from jar:file:///home/aaa/m2/io/netty/netty-common/4.1.70.Final/netty-common-4.1.70.Final.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted' and from jar:file:///home/aaa/m2/com/alibaba/nacos/nacos-client/2.1.2/nacos-client-2.1.2.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted' and from jar:file:///home/aaa/test/brobot-factory/brobot-factory-web/target/lib/nacos-client-2.1.2.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted' and from jar:file:///home/aaa/test/brobot-factory/brobot-factory-web/target/lib/netty-common-4.1.70.Final.jar!/META-INF/native-image/io.netty/common/native-image.properties with 'io.netty.util.AbstractReferenceCounted'). To see why io.netty.util.AbstractReferenceCounted got initialized use --trace-class-initialization=io.netty.util.AbstractReferenceCounted

        at com.oracle.svm.core.util.UserError.abort(UserError.java:73)
        at com.oracle.svm.hosted.classinitialization.ConfigurableClassInitialization.checkDelayedInitialization(ConfigurableClassInitialization.java:555)
        at com.oracle.svm.hosted.classinitialization.ClassInitializationFeature.duringAnalysis(ClassInitializationFeature.java:168)
        at com.oracle.svm.hosted.NativeImageGenerator.lambda$runPointsToAnalysis$12(NativeImageGenerator.java:727)
        at com.oracle.svm.hosted.FeatureHandler.forEachFeature(FeatureHandler.java:73)
        at com.oracle.svm.hosted.NativeImageGenerator.runPointsToAnalysis(NativeImageGenerator.java:727)
        at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:529)
        at com.oracle.svm.hosted.NativeImageGenerator.run(NativeImageGenerator.java:488)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.buildImage(NativeImageGeneratorRunner.java:403)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.build(NativeImageGeneratorRunner.java:569)
        at com.oracle.svm.hosted.NativeImageGeneratorRunner.main(NativeImageGeneratorRunner.java:122)
[brobot-factory:3193]      [total]: 139,610.35 ms,  7.84 GB
# Printing build artifacts to: /home/aaa/test/brobot-factory/brobot-factory-web/target/brobot-factory.build_artifacts.txt
Error: Image build request failed with exit status 1
com.oracle.svm.driver.NativeImage$NativeImageError: Image build request failed with exit status 1
        at com.oracle.svm.driver.NativeImage.showError(NativeImage.java:1762)
        at com.oracle.svm.driver.NativeImage.build(NativeImage.java:1473)
        at com.oracle.svm.driver.NativeImage.performBuild(NativeImage.java:1434)
        at com.oracle.svm.driver.NativeImage.main(NativeImage.java:1421)
```
这个问题是说有些配置说netty需要运行时初始化 有些是编译的时候就要初始化
加上配置进行链路分析
<buildArg>--trace-class-initialization=io.netty.util.AbstractReferenceCounted</buildArg>
调试信息显示 io.netty.channel.kqueue.KQueue 类在构建时被初始化，它触发了 AbstractReferenceCounted 的初始化 加上配置--initialize-at-run-time=io.netty.channel.kqueue
```
Logging system failed to initialize using configuration from 'null'
java.lang.IllegalStateException: Could not initialize Logback logging from classpath:logback-spring.xml
        at org.springframework.boot.logging.logback.LogbackLoggingSystem.loadConfiguration(LogbackLoggingSystem.java:168)
        at org.springframework.boot.logging.AbstractLoggingSystem.initializeWithConventions(AbstractLoggingSystem.java:80)
        at org.springframework.boot.logging.AbstractLoggingSystem.initialize(AbstractLoggingSystem.java:60)
        at org.springframework.boot.logging.logback.LogbackLoggingSystem.initialize(LogbackLoggingSystem.java:132)
        at org.springframework.boot.context.logging.LoggingApplicationListener.initializeSystem(LoggingApplicationListener.java:312)
        at org.springframework.boot.context.logging.LoggingApplicationListener.initialize(LoggingApplicationListener.java:281)
        at org.springframework.boot.context.logging.LoggingApplicationListener.onApplicationEnvironmentPreparedEvent(LoggingApplicationListener.java:239)
        at org.springframework.boot.context.logging.LoggingApplicationListener.onApplicationEvent(LoggingApplicationListener.java:216)
        at org.springframework.context.event.SimpleApplicationEventMulticaster.doInvokeListener(SimpleApplicationEventMulticaster.java:176)
        at org.springframework.context.event.SimpleApplicationEventMulticaster.invokeListener(SimpleApplicationEventMulticaster.java:169)
        at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:143)
        at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:131)
        at org.springframework.boot.context.event.EventPublishingRunListener.environmentPrepared(EventPublishingRunListener.java:82)
        at org.springframework.boot.SpringApplicationRunListeners.lambda$environmentPrepared$2(SpringApplicationRunListeners.java:63)
        at java.util.ArrayList.forEach(ArrayList.java:1259)
        at org.springframework.boot.SpringApplicationRunListeners.doWithListeners(SpringApplicationRunListeners.java:117)
        at org.springframework.boot.SpringApplicationRunListeners.doWithListeners(SpringApplicationRunListeners.java:111)
        at org.springframework.boot.SpringApplicationRunListeners.environmentPrepared(SpringApplicationRunListeners.java:62)
        at org.springframework.boot.SpringApplication.prepareEnvironment(SpringApplication.java:375)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:333)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318)
        at com.utopia.factory.web.Application.main(Application.java:23)
Caused by: ch.qos.logback.core.LogbackException: Unexpected filename extension of file [resource:logback-spring.xml]. Should be either .groovy or .xml
        at ch.qos.logback.classic.util.ContextInitializer.configureByResource(ContextInitializer.java:58)
        at org.springframework.boot.logging.logback.LogbackLoggingSystem.configureByResourceUrl(LogbackLoggingSystem.java:191)
        at org.springframework.boot.logging.logback.LogbackLoggingSystem.loadConfiguration(LogbackLoggingSystem.java:165)
        ... 22 more
```
不知道为什么明明文件名是xml结尾的他识别不出来 看源码也看不出来 猜测是文件已经是二进制的了
Unexpected filename extension of file [resource:logback-spring.xml]. Should be either .groovy or .xml
于是把日志配置文件拿出来
又出现了dubbo日志的空指针问题 
```
java.lang.NullPointerException: null
        at org.apache.dubbo.common.logger.LoggerFactory.lambda$getLogger$0(LoggerFactory.java:119) ~[na:na]
        at java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1660) ~[na:na]
        at org.apache.dubbo.common.logger.LoggerFactory.getLogger(LoggerFactory.java:119) ~[na:na]
        at org.apache.dubbo.spring.boot.context.event.WelcomeLogoApplicationListener.onApplicationEvent(WelcomeLogoApplicationListener.java:59) ~[na:na]
        at org.apache.dubbo.spring.boot.context.event.WelcomeLogoApplicationListener.onApplicationEvent(WelcomeLogoApplicationListener.java:42) ~[na:na]
        at org.springframework.context.event.SimpleApplicationEventMulticaster.doInvokeListener(SimpleApplicationEventMulticaster.java:176) ~[na:na]
        at org.springframework.context.event.SimpleApplicationEventMulticaster.invokeListener(SimpleApplicationEventMulticaster.java:169) ~[na:na]
        at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:143) ~[na:na]
        at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:131) ~[na:na]
        at org.springframework.boot.context.event.EventPublishingRunListener.environmentPrepared(EventPublishingRunListener.java:82) ~[brobot-factory:na]
        at org.springframework.boot.SpringApplicationRunListeners.lambda$environmentPrepared$2(SpringApplicationRunListeners.java:63) ~[na:na]
        at java.util.ArrayList.forEach(ArrayList.java:1259) ~[brobot-factory:na]
        at org.springframework.boot.SpringApplicationRunListeners.doWithListeners(SpringApplicationRunListeners.java:117) ~[na:na]
        at org.springframework.boot.SpringApplicationRunListeners.doWithListeners(SpringApplicationRunListeners.java:111) ~[na:na]
        at org.springframework.boot.SpringApplicationRunListeners.environmentPrepared(SpringApplicationRunListeners.java:62) ~[na:na]
        at org.springframework.boot.SpringApplication.prepareEnvironment(SpringApplication.java:375) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:333) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:24) [brobot-factory:na]
```
NPE 发生在 loggerAdapter.getLogger(key)： loggerAdapter 这个静态变量为 null 当 lambda 执行时，调用 null.getLogger(key) 导致 NPE
org.apache.dubbo.common.logger.LoggerFactory的static方法自动检测可用的日志框架 - 在 Native 下失败 没用被正确初始化
加上配置解决System.setProperty("dubbo.application.logger", "slf4j");
```
java.lang.IllegalStateException: Failed to create adaptive instance: java.lang.IllegalStateException: Can't create adaptive extension interface org.apache.dubbo.common.extension.ExtensionInjector, cause: No adaptive method exist on extension org.apache.dubbo.common.extension.ExtensionInjector, refuse to create the adaptive class!
        at org.apache.dubbo.common.extension.ExtensionLoader.getAdaptiveExtension(ExtensionLoader.java:730) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionLoader.<init>(ExtensionLoader.java:209) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionDirector.createExtensionLoader0(ExtensionDirector.java:123) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionDirector.createExtensionLoader(ExtensionDirector.java:114) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionDirector.getExtensionLoader(ExtensionDirector.java:104) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionAccessor.getExtensionLoader(ExtensionAccessor.java:27) ~[na:na]
        at org.apache.dubbo.metadata.definition.TypeDefinitionBuilder.initBuilders(TypeDefinitionBuilder.java:43) ~[na:na]
        at org.apache.dubbo.rpc.model.FrameworkModel.<init>(FrameworkModel.java:90) ~[brobot-factory:na]
        at org.apache.dubbo.rpc.model.FrameworkModel.defaultModel(FrameworkModel.java:181) ~[brobot-factory:na]
        at org.apache.dubbo.config.spring.context.DubboSpringInitializer.customize(DubboSpringInitializer.java:186) ~[na:na]
        at org.apache.dubbo.config.spring.context.DubboSpringInitializer.initContext(DubboSpringInitializer.java:106) ~[na:na]
        at org.apache.dubbo.config.spring.context.DubboSpringInitializer.initialize(DubboSpringInitializer.java:63) ~[na:na]
        at org.apache.dubbo.config.spring.context.annotation.DubboConfigConfigurationRegistrar.registerBeanDefinitions(DubboConfigConfigurationRegistrar.java:40) ~[brobot-factory:na]
        at org.springframework.context.annotation.ImportBeanDefinitionRegistrar.registerBeanDefinitions(ImportBeanDefinitionRegistrar.java:86) ~[brobot-factory:5.3.13]
        at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.lambda$loadBeanDefinitionsFromRegistrars$1(ConfigurationClassBeanDefinitionReader.java:396) ~[na:na]
        at java.util.LinkedHashMap.forEach(LinkedHashMap.java:684) ~[na:na]
        at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsFromRegistrars(ConfigurationClassBeanDefinitionReader.java:395) ~[na:na]
        at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass(ConfigurationClassBeanDefinitionReader.java:157) ~[na:na]
        at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitions(ConfigurationClassBeanDefinitionReader.java:129) ~[na:na]
        at org.springframework.context.annotation.ConfigurationClassPostProcessor.processConfigBeanDefinitions(ConfigurationClassPostProcessor.java:343) ~[brobot-factory:5.3.13]
        at org.springframework.context.annotation.ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(ConfigurationClassPostProcessor.java:247) ~[brobot-factory:5.3.13]
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:311) ~[na:na]
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:112) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:746) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:564) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:144) ~[na:na]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:771) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:763) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:438) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:339) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:19) [brobot-factory:na]
Caused by: java.lang.IllegalStateException: Can't create adaptive extension interface org.apache.dubbo.common.extension.ExtensionInjector, cause: No adaptive method exist on extension org.apache.dubbo.common.extension.ExtensionInjector, refuse to create the adaptive class!
        at org.apache.dubbo.common.extension.ExtensionLoader.createAdaptiveExtension(ExtensionLoader.java:1316) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionLoader.getAdaptiveExtension(ExtensionLoader.java:726) ~[brobot-factory:na]
        ... 32 common frames omitted
Caused by: java.lang.IllegalStateException: No adaptive method exist on extension org.apache.dubbo.common.extension.ExtensionInjector, refuse to create the adaptive class!
        at org.apache.dubbo.common.extension.AdaptiveClassCodeGenerator.generate(AdaptiveClassCodeGenerator.java:104) ~[na:na]
        at org.apache.dubbo.common.extension.AdaptiveClassCodeGenerator.generate(AdaptiveClassCodeGenerator.java:94) ~[na:na]
        at org.apache.dubbo.common.extension.ExtensionLoader.createAdaptiveExtensionClass(ExtensionLoader.java:1338) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionLoader.getAdaptiveExtensionClass(ExtensionLoader.java:1325) ~[brobot-factory:na]
        at org.apache.dubbo.common.extension.ExtensionLoader.createAdaptiveExtension(ExtensionLoader.java:1309) ~[brobot-factory:na]
        ... 33 common frames omitted
```
但是我实际上dubbo是关闭的 配置文件里dubbo.enabled=false 因为dubbo是关闭的 本地启动压根就不会进入报错的代码路径
@ConditionalOnProperty(prefix = "dubbo", value = "enabled", matchIfMissing = true)在Native环境中： 构建时无法读取dubbo.enabled=false配置 matchIfMissing = true导致条件被解析为true Dubbo自动配置被错误启用
GraalVM Native Image确实对@ConditionalOnProperty有特殊处理： 构建时解析：Native Image在构建阶段就尝试解析条件注解 属性不可用：构建时应用程序属性文件还没有被加载 默认值问题：matchIfMissing = true可能在构建时被错误解析
上面是ai解释的 但是应该spring native会build的时候进行分析啊 不知道啥原理
@SpringBootApplication(exclude = { DubboAutoConfiguration.class}) 先绕过dubbo
```
java.lang.ExceptionInInitializerError: null
        at org.mybatis.spring.mapper.MapperScannerConfigurer.postProcessBeanDefinitionRegistry(MapperScannerConfigurer.java:363) ~[brobot-factory:na]
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:311) ~[na:na]
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:142) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:746) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:564) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:144) ~[na:na]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:771) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:763) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:438) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:339) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:21) [brobot-factory:na]
Caused by: org.apache.ibatis.logging.LogException: Error creating logger for logger org.mybatis.spring.mapper.ClassPathMapperScanner.  Cause: java.lang.NullPointerException
        at org.apache.ibatis.logging.LogFactory.getLog(LogFactory.java:54) ~[na:na]
        at org.apache.ibatis.logging.LogFactory.getLog(LogFactory.java:47) ~[na:na]
        at org.mybatis.logging.LoggerFactory.getLogger(LoggerFactory.java:32) ~[na:na]
        at org.mybatis.spring.mapper.ClassPathMapperScanner.<clinit>(ClassPathMapperScanner.java:60) ~[na:na]
        ... 13 common frames omitted
Caused by: java.lang.NullPointerException: null
        at org.apache.ibatis.logging.LogFactory.getLog(LogFactory.java:52) ~[na:na]
        ... 16 common frames omitted
```
增加反射配置解决
```
2025-11-25 13:44:23.568  WARN 11710 --- [           main] ConfigServletWebServerApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.context.ApplicationContextException: Unable to start web server; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'undertowServletWebServerFactory' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/ServletWebServerFactoryConfiguration$EmbeddedUndertow.class]: Initialization of bean failed; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'errorPageCustomizer' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/error/ErrorMvcAutoConfiguration.class]: Unsatisfied dependency expressed through method 'errorPageCustomizer' parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'dispatcherServletRegistration' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletRegistrationConfiguration.class]: Unsatisfied dependency expressed through method 'dispatcherServletRegistration' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dispatcherServlet' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
2025-11-25 13:44:23.579  INFO 11710 --- [           main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2025-11-25 13:44:23.581 ERROR 11710 --- [           main] o.s.boot.SpringApplication               : Application run failed

org.springframework.context.ApplicationContextException: Unable to start web server; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'undertowServletWebServerFactory' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/ServletWebServerFactoryConfiguration$EmbeddedUndertow.class]: Initialization of bean failed; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'errorPageCustomizer' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/error/ErrorMvcAutoConfiguration.class]: Unsatisfied dependency expressed through method 'errorPageCustomizer' parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'dispatcherServletRegistration' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletRegistrationConfiguration.class]: Unsatisfied dependency expressed through method 'dispatcherServletRegistration' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dispatcherServlet' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:162) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:577) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:144) ~[na:na]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:771) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:763) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:438) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:339) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:24) [brobot-factory:na]
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'undertowServletWebServerFactory' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/ServletWebServerFactoryConfiguration$EmbeddedUndertow.class]: Initialization of bean failed; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'errorPageCustomizer' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/error/ErrorMvcAutoConfiguration.class]: Unsatisfied dependency expressed through method 'errorPageCustomizer' parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'dispatcherServletRegistration' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletRegistrationConfiguration.class]: Unsatisfied dependency expressed through method 'dispatcherServletRegistration' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dispatcherServlet' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:628) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:213) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.getWebServerFactory(ServletWebServerApplicationContext.java:216) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.createWebServer(ServletWebServerApplicationContext.java:179) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:159) ~[na:na]
        ... 9 common frames omitted
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'errorPageCustomizer' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/error/ErrorMvcAutoConfiguration.class]: Unsatisfied dependency expressed through method 'errorPageCustomizer' parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'dispatcherServletRegistration' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletRegistrationConfiguration.class]: Unsatisfied dependency expressed through method 'dispatcherServletRegistration' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dispatcherServlet' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:800) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:541) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1352) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1195) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBeansOfType(DefaultListableBeanFactory.java:671) ~[na:na]
        at org.springframework.boot.web.server.ErrorPageRegistrarBeanPostProcessor.getRegistrars(ErrorPageRegistrarBeanPostProcessor.java:76) ~[na:na]
        at org.springframework.boot.web.server.ErrorPageRegistrarBeanPostProcessor.postProcessBeforeInitialization(ErrorPageRegistrarBeanPostProcessor.java:67) ~[na:na]
        at org.springframework.boot.web.server.ErrorPageRegistrarBeanPostProcessor.postProcessBeforeInitialization(ErrorPageRegistrarBeanPostProcessor.java:56) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization(AbstractAutowireCapableBeanFactory.java:440) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1796) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:620) ~[na:na]
        ... 17 common frames omitted
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'dispatcherServletRegistration' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletRegistrationConfiguration.class]: Unsatisfied dependency expressed through method 'dispatcherServletRegistration' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dispatcherServlet' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:800) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:541) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1352) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1195) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1380) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1300) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:887) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:791) ~[na:na]
        ... 33 common frames omitted
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dispatcherServlet' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/DispatcherServletAutoConfiguration$DispatcherServletConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
        at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:658) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:638) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1352) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1195) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1380) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1300) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:887) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:791) ~[na:na]
        ... 47 common frames omitted
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.DispatcherServlet]: Factory method 'dispatcherServlet' threw exception; nested exception is java.lang.ExceptionInInitializerError
        at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:185) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:653) ~[na:na]
        ... 61 common frames omitted
Caused by: java.lang.ExceptionInInitializerError: null
        at java.lang.Class.ensureInitialized(DynamicHub.java:546) ~[na:na]
        at java.lang.Class.ensureInitialized(DynamicHub.java:546) ~[na:na]
        at java.lang.Class.ensureInitialized(DynamicHub.java:546) ~[na:na]
        at java.lang.Class.ensureInitialized(DynamicHub.java:546) ~[na:na]
        at org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration$DispatcherServletConfiguration.dispatcherServlet(DispatcherServletAutoConfiguration.java:90) ~[brobot-factory:na]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[na:na]
        at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154) ~[na:na]
        ... 62 common frames omitted
Caused by: java.util.MissingResourceException: Resource bundle not found javax.servlet.LocalStrings, locale en_US. Register the resource bundle using the option -H:IncludeResourceBundles=javax.servlet.LocalStrings.
        at com.oracle.svm.core.jdk.localization.OptimizedLocalizationSupport.getCached(OptimizedLocalizationSupport.java:72) ~[na:na]
        at java.util.ResourceBundle.getBundle(ResourceBundle.java:51) ~[na:na]
        at javax.servlet.GenericServlet.<clinit>(GenericServlet.java:51) ~[na:na]
        ... 69 common frames omitted
```
国际化资源包找不到 先看一下源码为什么找不到
GenericServlet private static ResourceBundle lStrings = ResourceBundle.getBundle(LSTRING_FILE);
GraalVM的封闭世界假设（Closed World Assumption） 静态分析限制： GraalVM只能分析代码中明确引用的资源 无法动态发现ResourceBundle.getBundle("xxx")调用中的字符串参数
-H:IncludeResourceBundles=javax.servlet.LocalStrings resource-config.json加上bundles  并且自定义了一个LocalStrings_en_US.properties解决
加了很多配置具体加的哪一个生效的不清楚 后面有空回来细看
加了这些 native-image.properties
-H:IncludeResourceBundles=javax.servlet.LocalStrings
-H:IncludeResourceBundles=javax.servlet.LocalStrings_en_US
resource-config.json
```
"bundles": [
    {
      "name": "com.mysql.cj.LocalizedErrorMessages"
    },
    {
      "name": "javax.servlet.LocalStrings"
    },
    {
      "name": "javax.servlet.http.LocalStrings"
    },
    {
      "name": "javax.servlet.LocalStrings_en_US"
    }
  ]
```
新的问题
```
org.springframework.context.ApplicationContextException: Unable to start web server; nested exception is java.lang.NoSuchMethodError: io.undertow.util.FastConcurrentDirectDeque.<init>()
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:162) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:577) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:144) ~[na:na]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:771) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:763) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:438) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:339) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:24) [brobot-factory:na]
Caused by: java.lang.NoSuchMethodError: io.undertow.util.FastConcurrentDirectDeque.<init>()
        at io.undertow.util.ConcurrentDirectDeque.<clinit>(ConcurrentDirectDeque.java:45) ~[na:na]
        at io.undertow.server.handlers.cache.LRUCache.<init>(LRUCache.java:67) ~[na:na]
        at io.undertow.servlet.handlers.ServletPathMatches.<init>(ServletPathMatches.java:82) ~[na:na]
        at io.undertow.servlet.core.DeploymentImpl.<init>(DeploymentImpl.java:95) ~[na:na]
        at io.undertow.servlet.core.DeploymentManagerImpl.deploy(DeploymentManagerImpl.java:151) ~[na:na]
        at org.springframework.boot.web.embedded.undertow.UndertowServletWebServerFactory.createManager(UndertowServletWebServerFactory.java:347) ~[brobot-factory:na]
        at org.springframework.boot.web.embedded.undertow.UndertowServletWebServerFactory.getWebServer(UndertowServletWebServerFactory.java:316) ~[brobot-factory:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.createWebServer(ServletWebServerApplicationContext.java:181) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:159) ~[na:na]
        ... 9 common frames omitted
```
反射问题 于是使用agent生成的反射配置 但是不行 自动生成的配置太多了 而且有问题 编译都不行了 退回之前的反射配置 去掉undertow改用tomcat试试
```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'deviceAuthService': Injection of resource dependencies failed; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'com.utopia.factory.dao.code.mapper.ProductSNMapper' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {@javax.annotation.Resource}
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessProperties(CommonAnnotationBeanPostProcessor.java:332) ~[brobot-factory:5.3.13]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1431) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:619) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:944) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:918) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:583) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:144) ~[na:na]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:771) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:763) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:438) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:339) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:24) [brobot-factory:na]
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'com.utopia.factory.dao.code.mapper.ProductSNMapper' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {@javax.annotation.Resource}
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.raiseNoMatchingBeanFound(DefaultListableBeanFactory.java:1790) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1346) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1300) ~[na:na]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.autowireResource(CommonAnnotationBeanPostProcessor.java:544) ~[brobot-factory:5.3.13]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.getResource(CommonAnnotationBeanPostProcessor.java:520) ~[brobot-factory:5.3.13]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.getResourceToInject(CommonAnnotationBeanPostProcessor.java:673) ~[na:na]
        at org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:228) ~[na:na]
        at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:119) ~[na:na]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessProperties(CommonAnnotationBeanPostProcessor.java:329) ~[brobot-factory:5.3.13]
        ... 18 common frames omitted 
```
反射问题 ProductSNMapper接口没有被注册为Spring Bean，这是因为： Spring Native限制：Native Image构建时无法进行动态类路径扫描 MyBatis Mapper扫描：@MapperScan注解在运行时有效，但在构建时无效 接口代理：MyBatis Mapper接口需要运行时动态代理
加上proxy-config.json也没用 说是不支持mapperscan 于是手动注册mapper
```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'deviceAuthService': Injection of resource dependencies failed; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'databaseConfig' defined in class path resource [com/utopia/factory/web/archetype/config/database/DatabaseConfig.class]: Unsatisfied dependency expressed through constructor parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'sqlSessionFactory' defined in class path resource [com/baomidou/mybatisplus/autoconfigure/MybatisPlusAutoConfiguration.class]: Unsatisfied dependency expressed through method 'sqlSessionFactory' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessProperties(CommonAnnotationBeanPostProcessor.java:332) ~[brobot-factory:5.3.13]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1431) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:619) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:944) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:918) ~[na:na]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:583) ~[na:na]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:144) ~[na:na]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:771) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:763) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:438) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:339) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1329) [brobot-factory:2.4.13]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1318) [brobot-factory:2.4.13]
        at com.utopia.factory.web.Application.main(Application.java:24) [brobot-factory:na]
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'databaseConfig' defined in class path resource [com/utopia/factory/web/archetype/config/database/DatabaseConfig.class]: Unsatisfied dependency expressed through constructor parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'sqlSessionFactory' defined in class path resource [com/baomidou/mybatisplus/autoconfigure/MybatisPlusAutoConfiguration.class]: Unsatisfied dependency expressed through method 'sqlSessionFactory' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:800) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.autowireConstructor(ConstructorResolver.java:229) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireConstructor(AbstractAutowireCapableBeanFactory.java:1372) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1222) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:410) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1352) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1195) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:213) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.resolveBeanByName(AbstractAutowireCapableBeanFactory.java:479) ~[na:na]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.autowireResource(CommonAnnotationBeanPostProcessor.java:550) ~[brobot-factory:5.3.13]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.getResource(CommonAnnotationBeanPostProcessor.java:520) ~[brobot-factory:5.3.13]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.getResourceToInject(CommonAnnotationBeanPostProcessor.java:673) ~[na:na]
        at org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:228) ~[na:na]
        at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:119) ~[na:na]
        at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessProperties(CommonAnnotationBeanPostProcessor.java:329) ~[brobot-factory:5.3.13]
        ... 18 common frames omitted
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'sqlSessionFactory' defined in class path resource [com/baomidou/mybatisplus/autoconfigure/MybatisPlusAutoConfiguration.class]: Unsatisfied dependency expressed through method 'sqlSessionFactory' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:800) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:541) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1352) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1195) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1380) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1300) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:887) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:791) ~[na:na]
        ... 43 common frames omitted
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
        at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:658) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:638) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1352) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1195) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:582) ~[na:na]
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335) ~[na:na]
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333) ~[na:na]
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208) ~[na:na]
        at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1380) ~[na:na]
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1300) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:887) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:791) ~[na:na]
        ... 57 common frames omitted
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
        at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:185) ~[na:na]
        at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:653) ~[na:na]
        ... 71 common frames omitted
Caused by: org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
        at org.springframework.boot.autoconfigure.jdbc.DataSourceProperties.determineDriverClassName(DataSourceProperties.java:236) ~[brobot-factory:na]
        at org.springframework.boot.autoconfigure.jdbc.DataSourceProperties.initializeDataSourceBuilder(DataSourceProperties.java:177) ~[brobot-factory:na]
        at org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration.createDataSource(DataSourceConfiguration.java:48) ~[na:na]
        at org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration$Hikari.dataSource(DataSourceConfiguration.java:90) ~[brobot-factory:na]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[na:na]
        at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154) ~[na:na]
        ... 72 common frames omitted
```
数据库找不到驱动 应该是资源没配置对 不管了 放弃了 换思路