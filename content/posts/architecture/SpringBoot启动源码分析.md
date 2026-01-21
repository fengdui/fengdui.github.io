---
title: "SpringBoot启动源码分析"
date: "2016-12-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
SpringApplication 构造方法
```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 1. 设置资源加载器
    this.resourceLoader = resourceLoader;
    
    // 2. 设置主配置类（可以有多个）
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    
    // 3. 推断应用类型 - 重要！！！
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    
    // 4. 加载 ApplicationContextInitializer
    // 从 META-INF/spring.factories 中加载
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    
    // 5. 加载 ApplicationListener
    // 从 META-INF/spring.factories 中加载
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    
    // 6. 推断主类（用于日志等）
    this.mainApplicationClass = deduceMainApplicationClass();
}
```
getSpringFactoriesInstances() 工厂加载机制
```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, 
                                                     Class<?>[] parameterTypes, 
                                                     Object... args) {
    ClassLoader classLoader = getClassLoader();
    
    // 1. 使用 SpringFactoriesLoader 加载全类名
    Set<String> names = new LinkedHashSet<>(
        SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    
    // 2. 创建实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, 
                                                       classLoader, args, names);
    
    // 3. 排序（按 @Order 注解）
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

// SpringFactoriesLoader.loadFactoryNames 的实现
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    
    // 加载所有 META-INF/spring.factories 文件
    return loadSpringFactories(classLoader)
        .getOrDefault(factoryTypeName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 缓存机制，避免重复加载
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }
    
    try {
        // 获取所有类路径下的 META-INF/spring.factories 文件
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            
            // 解析 properties 文件
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : 
                     StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    } catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```
run() 方法
```
public ConfigurableApplicationContext run(String... args) {
    // 1. 启动计时器
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    
    // 2. 上下文引用和异常报告器
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    
    // 3. 设置 headless 模式（无图形界面）
    configureHeadlessProperty();
    
    // 4. 获取并启动 SpringApplicationRunListeners
    SpringApplicationRunListeners listeners = getRunListeners(args);
    
    // 5. 发布 ApplicationStartingEvent 事件
    listeners.starting();
    
    try {
        // 6. 准备环境（最重要的一步）
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        
        // 7. 配置忽略的 Bean 信息
        configureIgnoreBeanInfo(environment);
        
        // 8. 打印 Banner
        Banner printedBanner = printBanner(environment);
        
        // 9. 创建 ApplicationContext（根据应用类型）
        context = createApplicationContext();
        
        // 10. 准备异常报告器
        exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
        
        // 11. 准备上下文（核心）
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        
        // 12. 刷新上下文（最核心）
        refreshContext(context);
        
        // 13. 刷新后处理（空实现，留给子类扩展）
        afterRefresh(context, applicationArguments);
        
        // 14. 停止计时器
        stopWatch.stop();
        
        // 15. 打印启动耗时
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                .logStarted(getApplicationLog(), stopWatch);
        }
        
        // 16. 发布 ApplicationStartedEvent 事件
        listeners.started(context);
        
        // 17. 调用 Runner（ApplicationRunner 和 CommandLineRunner）
        callRunners(context, applicationArguments);
        
    } catch (Throwable ex) {
        // 18. 处理启动失败
        handleRunFailure(context, listeners, exceptionReporters, ex);
        throw new IllegalStateException(ex);
    }
    
    try {
        // 19. 发布 ApplicationReadyEvent 事件
        listeners.running(context);
    } catch (Throwable ex) {
        handleRunFailure(context, listeners, exceptionReporters, ex);
        throw new IllegalStateException(ex);
    }
    
    // 20. 返回上下文
    return context;
}
```
prepareEnvironment() 环境准备
```
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    
    // 1. 根据应用类型创建环境
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    
    // 2. 配置环境（命令行参数、配置文件等）
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    
    // 3. 发布 ApplicationEnvironmentPreparedEvent 事件
    // 监听器可以修改环境配置
    listeners.environmentPrepared(environment);
    
    // 4. 绑定环境到 SpringApplication
    bindToSpringApplication(environment);
    
    // 5. 如果不是 web 环境，转换为 StandardEnvironment
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader())
            .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    
    // 6. 配置 PropertySources 的合理顺序
    ConfigurationPropertySources.attach(environment);
    return environment;
}

// 环境创建逻辑
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
        case SERVLET:
            // 创建 StandardServletEnvironment
            return new StandardServletEnvironment();
        case REACTIVE:
            // 创建 StandardReactiveWebEnvironment
            return new StandardReactiveWebEnvironment();
        default:
            // 创建 StandardEnvironment
            return new StandardEnvironment();
    }
}
```
createApplicationContext() 创建上下文
```
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
                case SERVLET:
                    // AnnotationConfigServletWebServerApplicationContext
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    // AnnotationConfigReactiveWebServerApplicationContext
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    // AnnotationConfigApplicationContext
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        } catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Unable create a default ApplicationContext, " +
                "please specify an ApplicationContextClass", ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```
prepareContext() 准备上下文
```
private void prepareContext(ConfigurableApplicationContext context,
                           ConfigurableEnvironment environment,
                           SpringApplicationRunListeners listeners,
                           ApplicationArguments applicationArguments,
                           Banner printedBanner) {
    
    // 1. 设置环境
    context.setEnvironment(environment);
    
    // 2. 后处理 ApplicationContext
    postProcessApplicationContext(context);
    
    // 3. 应用 ApplicationContextInitializer
    applyInitializers(context);
    
    // 4. 发布 ApplicationContextInitializedEvent 事件
    listeners.contextPrepared(context);
    
    // 5. 打印启动日志
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    
    // 6. 添加单例 Bean
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    
    // 6.1 注册 applicationArguments 单例
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    
    // 6.2 注册 printedBanner 单例
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    
    // 7. 设置 Bean 生成器的作用域
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    
    // 8. 设置懒加载
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    
    // 9. 加载 Bean 定义
    // 这一步会加载主配置类（@SpringBootApplication 标注的类）
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    
    // 10. 创建 BeanDefinitionLoader
    load(context, sources.toArray(new Object[0]));
    
    // 11. 发布 ApplicationPreparedEvent 事件
    listeners.contextLoaded(context);
}
```
refreshContext() 刷新上下文
```
private void refreshContext(ConfigurableApplicationContext context) {
    // 注册关闭钩子
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        } catch (AccessControlException ex) {
            // 忽略
        }
    }
    
    // 调用核心的 refresh 方法
    refresh(context);
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    
    // 调用 Spring 的核心刷新方法
    ((AbstractApplicationContext) applicationContext).refresh();
}
```
@EnableAutoConfiguration 的工作机制 AutoConfigurationImportSelector 源码
```
public class AutoConfigurationImportSelector implements ... {
    
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        
        // 1. 获取自动配置的元数据
        AutoConfigurationMetadata autoConfigurationMetadata = 
            AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
        
        // 2. 获取所有候选配置
        AutoConfigurationEntry autoConfigurationEntry = 
            getAutoConfigurationEntry(autoConfigurationMetadata, annotationMetadata);
        
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }
    
    protected AutoConfigurationEntry getAutoConfigurationEntry(
            AutoConfigurationMetadata autoConfigurationMetadata,
            AnnotationMetadata annotationMetadata) {
        
        // 1. 检查是否启用自动配置
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        
        // 2. 获取注解属性
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        
        // 3. 获取候选配置
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        
        // 4. 去重
        configurations = removeDuplicates(configurations);
        
        // 5. 根据 exclude/excludeName 排除
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        
        // 6. 应用过滤器
        configurations = filter(configurations, autoConfigurationMetadata);
        
        // 7. 触发自动配置导入事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        
        return new AutoConfigurationEntry(configurations, exclusions);
    }
    
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, 
                                                     AnnotationAttributes attributes) {
        // 从 spring.factories 加载 EnableAutoConfiguration 的所有配置类
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        
        Assert.notEmpty(configurations, 
            "No auto configuration classes found in META-INF/spring.factories. " +
            "If you are using a custom packaging, make sure that file is correct.");
        
        return configurations;
    }
}
```
@Configuration 类的处理
```
// ConfigurationClassPostProcessor 处理 @Configuration 类
public class ConfigurationClassPostProcessor implements ... {
    
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // 1. 生成 ConfigurationClass 的 BeanDefinition 的 hash
        int registryId = System.identityHashCode(registry);
        if (this.registriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(...);
        }
        
        // 2. 处理配置类
        processConfigBeanDefinitions(registry);
    }
    
    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
        
        // 1. 找出所有候选的配置类（标注了 @Configuration 的 Bean）
        String[] candidateNames = registry.getBeanDefinitionNames();
        for (String beanName : candidateNames) {
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
                // 已经处理过
            } else if (ConfigurationClassUtils.checkConfigurationClassCandidate(
                beanDef, this.metadataReaderFactory)) {
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }
        
        // 2. 排序配置类（按 @Order）
        configCandidates.sort((bd1, bd2) -> {
            int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
            int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
            return Integer.compare(i1, i2);
        });
        
        // 3. 创建 ConfigurationClassParser 解析配置类
        ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);
        
        Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
        Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
        
        do {
            // 4. 解析配置类
            parser.parse(candidates);
            parser.validate();
            
            // 5. 获取解析出的配置类
            Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);
            
            // 6. 读取配置类（注册 Import、@Bean 等）
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
            }
            this.reader.loadBeanDefinitions(configClasses);
            alreadyParsed.addAll(configClasses);
            
            candidates.clear();
            
            // 7. 检查是否有新注册的 BeanDefinition
            if (registry.getBeanDefinitionCount() > candidateNames.length) {
                String[] newCandidateNames = registry.getBeanDefinitionNames();
                Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
                Set<String> alreadyParsedClasses = new HashSet<>();
                
                for (ConfigurationClass configurationClass : alreadyParsed) {
                    alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
                }
                
                for (String candidateName : newCandidateNames) {
                    if (!oldCandidateNames.contains(candidateName)) {
                        BeanDefinition bd = registry.getBeanDefinition(candidateName);
                        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, 
                            this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                            candidates.add(new BeanDefinitionHolder(bd, candidateName));
                        }
                    }
                }
                candidateNames = newCandidateNames;
            }
        } while (!candidates.isEmpty());
        
        // 8. 注册 ImportRegistry
        if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
            sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
        }
    }
}
```
@Conditional 的处理
```
// ConditionEvaluator 条件评估器
class ConditionEvaluator {
    
    public boolean shouldSkip(AnnotatedTypeMetadata metadata) {
        return shouldSkip(metadata, null);
    }
    
    public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, 
                             @Nullable ConfigurationPhase phase) {
        // 1. 如果没有 @Conditional 注解，不跳过
        if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
            return false;
        }
        
        // 2. 如果没有指定阶段，推断阶段
        if (phase == null) {
            if (metadata instanceof AnnotationMetadata &&
                ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
                return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
            }
            return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
        }
        
        // 3. 获取所有条件
        List<Condition> conditions = new ArrayList<>();
        for (String[] conditionClasses : getConditionClasses(metadata)) {
            for (String conditionClass : conditionClasses) {
                Condition condition = getCondition(conditionClass, this.context.getClassLoader());
                conditions.add(condition);
            }
        }
        
        // 4. 排序条件
        AnnotationAwareOrderComparator.sort(conditions);
        
        // 5. 评估条件
        for (Condition condition : conditions) {
            ConfigurationPhase requiredPhase = null;
            if (condition instanceof ConfigurationCondition) {
                requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
            }
            
            if ((requiredPhase == null || requiredPhase == phase) &&
                !condition.matches(this.context, metadata)) {
                return true;
            }
        }
        
        return false;
    }
}
```
