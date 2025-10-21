---
title: "spring源码阅读"
date: "2016-07-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


spring aop原理
- <aop:aspectj-autoproxy proxy-target-class="true" expose-proxy="true"/>
- AopNamespaceHandler 注册AspectJAutoProxyBeanDefinitionParser，
- AspectJAutoProxyBeanDefinitionParser又注册了AnnotationAwareAspectJAutoProxyCreator。
- AnnotationAwareAspectJAutoProxyCreator实现了BeanPostProcessor，实现此接口的类会在获取bean的时候调用postProcessAfterInitialization方法，
- 继而又会调用wrapIfNecessary方法。
- wrapIfNecessary方法：

- 1 getAdvicesAndAdvisorsForBean 获取增强器, 获取增强器分两步，第一步获得所有的增强器，第二部获取适合当前类的增强器，
- 获得所有的增强器
- 调用过程是委托了aspectJAdvisorsBuilder找出所有的增强器，然后aspectJAdvisorsBuilder又使用了advisorFactory获取增强器。
- 在advisorFactory的getAdvisor方法中，根据不同的注解创建了不同的增强器，before，after等等。
- 然后使用InstantiationModelAwarePointcutAdvisorImpl封装起来。
- 获得适合的增强器
- 获取到所有的增强器之后在使用aoputils的canApply找出适合的增强器。

- 如果advisor是PointcutAdvisor子类，就会使用这个advisor的pointcut来匹配，
- 例如spring的事务BeanFactoryTransactionAttributeSourceAdvisor，它会使用TransactionAttributeSourcePointcut来查看是否使用事
- 务，如果可以就使用这个advisor。
- 2 createProxy 创建代理
- 创建代理类是在ProxyFactory的getProxy方法中，其中又调用了AopProxyFactory来创建aopProxy（例如JdkDynamicAopProxy或者
- ObjenesisCglibAopProxy），AopProxyFactory 默认的实现类是DefaultAopProxyFactory，得到aopProxy之后在使用aopProxy创建代理。
- 熟悉动态代理的都知道，代理类会调用invoke方法，invoke方法主要做了以下2个工作。
- 2.1.获得拦截器链
- List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
- advised是之前创建的proxyfactory，它又委托了AdvisorChainFactory，AdvisorChainFactory只有一个默认的实现
- 类，DefaultAdvisorChainFactory，它将advice封装成MethodInterceptor，之后又封装成InterceptorAndDynamicMethodMatcher。
- 2.2 使用ReflectiveMethodInvocation封装拦截器链，然后调用。
- invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
- // Proceed to the joinpoint through the interceptor chain.
- retVal = invocation.proceed();
  spring生命周期源码
- populateBean填充属性
- BeanNameAware和BeanFactoryAware
- 源码位于doCreateBean => AbstractAutowireCapableBeanFactory#initializeBean#invokeAwareMethods方法
- invokeAwareMethods方法
```
private void invokeAwareMethods(final String beanName, final Object bean) {
  if (bean instanceof Aware) {
    if (bean instanceof BeanNameAware) {
      ((BeanNameAware) bean).setBeanName(beanName);
    }
    if (bean instanceof BeanClassLoaderAware) {
      ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
    }
    if (bean instanceof BeanFactoryAware) {
      ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
    }
  }
}

```
- 调用beanPostProcessor前置处理
- beanProcessor.postProcessBeforeInitialization
- 位于AbstractAutowireCapableBeanFactory#initializeBean#applyBeanPostProcessorsBeforeInitialization
- afterPropertiesSet
- initMethod
- 位于AbstractAutowireCapableBeanFactory#initializeBean#invokeInitMethods
- 调用beanPostProcessor后置处理
- applyBeanPostProcessorsAfterInitialization
- 位于AbstractAutowireCapableBeanFactory#initializeBean#applyBeanPostProcessorsAfterInitialization
  spring的refresh方法
- spring 的ApplicationContext 包含了beanFactory的所有功能，还有其他额外功能。
- ClassPathXmlApplicationContext的父类AbstractApplicationContext的refresh()方法。
```
  public void refresh() throws BeansException, IllegalStateException {
      synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
  }
}

```
- 1.prepareBeanFactory方法，注册一个BeanPostProcessor。
- beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
- 这个类的功能就是添加各种aware功能，比如ApplicationContextAware。
- 2.invokeBeanFactoryPostProcessors方法，调用BeanFactoryPostProcessors。
- 这个方法就把所有的BeanFactoryPostProcessor调用了一遍。
- PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
- 2.1如果beanFactory是BeanDefinitionRegistry类型
- 遍历beanFactoryPostProcessors列表，如果是BeanDefinitionRegistryPostProcessor类型，强转在调用自定义的方法。
- 将预编码注册的BeanDefinitionRegistryPostProcessor（硬编码就是调用this.beanFactoryPostProcessors.add(postProcessor)方法放到beanFactoryPostProcessors中的）
- 放到registryPostProcessors
- 将硬编码注册的BeanFactoryPostProcessor放到regularPostProcessors中
- 就是BeanDefinitionRegistryPostProcessor还是BeanFactoryPostProcessor，如果是前者 先调用一下子类的方法，在全部调用父类的postProcessBeanFactory。
- 2.2如果beanFactory是BeanDefinitionRegistry类型
- 就直接调用postProcessBeanFactory方法。
- 3 registerBeanPostProcessors
- 这个方法就注册BeanPostProcessors，当你getbean的时候才会调用。
- registerListeners()
- 将硬编码的事件监听器(在applicationListeners中)，以及xml配置的事件监听器，以及earlyApplicationEvents中的事件放到事件广播器applicationEventMulticaster。
- earlyApplicationEvents作用是当你调用publishEvent方法的时候，事件广播器还没有初始化的时候，会先放到publishEvent。
  自定义BeanFactoryPostProcessor以javaconfig方式配置要加上static。
