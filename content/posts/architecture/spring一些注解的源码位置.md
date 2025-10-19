---
title: "spring一些注解的源码位置"
date: "2022-07-15"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


* springboot默认使用的容器是AnnotationConfigEmbeddedWebApplicationContext
* 改容器内部使用了AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner
* AnnotatedBeanDefinitionReader在构造函数中注册了很多BeanFactoryPostProcessor
* AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
* 其中就有一个ConfigurationClassPostProcessor
* 由这个ConfigurationClassPostProcessor处理springboot的启动类的@Configuration注解
* 大致流程postProcessBeanDefinitionRegistry=》processConfigBeanDefinitions=》parse=》processConfigurationClass
* @PropertySources @ComponentScans@Import @ImportResource@Bean 等注解都在
* doProcessConfigurationClass方法中处理
* 再回到之前的AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);发现里面还注册了其他的处理器，
* AutowiredAnnotationBeanPostProcessor就是其中一个 这个注解处理@Autowired和@Value注解
* postProcessBeanDefinitionRegistry 方法中解析完bean的信息之后， 会使用ConfigurationClassBeanDefinitionReader注册bean。
* @controller方法调用的源码是在ServletInvocableHandlerMethod。
* @EnableAutoConfiguration自动配置是从classpath中搜寻所有的META-INF/spring.factories配置文件，
* 并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射
* 实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后加载到IoC容器。
* AnnotationConfigUtils.registerAnnotationConfigProcessors 注册了@Autowired、@Value、@Inject等注解的processor。
* @Configuration注解的处理逻辑是在ConfigurationClassPostProcessor。
* @Autowired、@Value、@Inject注解的处理逻辑是在AutowiredAnnotationBeanPostProcessor。
* @Required注解的处理逻辑是在RequiredAnnotationBeanPostProcessor。
* @PostConstruct、@PreDestroy、@Resource注解的处理逻辑是在CommonAnnotationBeanPostProcessor。
* @jpa相关注解PersistenceAnnotationBeanPostProcessor。
* @ComponentScan注解在ComponentScanAnnotationParser.parse。
* @ResponseBody
    * 入口是MvcNamespaceHandler，注册了这个AnnotationDrivenBeanDefinitionParser解析器，
    * 在getMessageConverters方法最后如果我们加入了jackson的jar包，就使用jackson，否则如果有Gson，使用Gson，在RequestMappingHandlerAdapter的messageConverters域就存在这个json转换器。所以如果是使用默认的 标签，messageConverters就会有这些
    * RequestMappingHandlerAdapter#getDefaultArgumentResolvers方法
    * resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    * RequestResponseBodyMethodProcessor是处理@requestBody和@ReponseBody注解的
    * RequestMappingHandlerAdapter#afterPropertiesSet方法创建了HandlerMethodArgumentResolverComposite然后把上一步的Resolvers添加进去
    * RequestMappingHandlerAdapter#invokeHandlerMethod方法创建了ServletInvocableHandlerMethod
    * 将RequestMappingHandlerAdapter的HandlerMethodArgumentResolverComposite设置到ServletInvocableHandlerMethod里面
    * 委托ServletInvocableHandlerMethod#invokeForRequest
    * 然后ServletInvocableHandlerMethod #getMethodArgumentValues创建请求的参数（就是给你自动生成controller里面的参数），
    * 遍历HandlerMethodArgumentResolverComposite里面的Resolvers ，
    * 如果有一个HandlerMethodArgumentResolver#supportsParameter支持，使用该resolves#resolveArgument得到结果
    * controller的流程大致是
    * dispatcherServlet => RequestMappingHandlerAdapter => invokeHandlerMethod=>ServletInvocableHandlerMethod
* @RequestParam 位于RequestParamMethodArgumentResolver和RequestParamMapMethodArgumentResolver
* @PathVariable 位于PathVariableMethodArgumentResolver和PathVariableMapMethodArgumentResolver
* @MatrixVariable 位于MatrixVariableMethodArgumentResolver和MatrixVariableMapMethodArgumentResolver
* @ModelAttribute ServletModelAttributeMethodProcessor
* @RequestBody @ResponseBody RequestResponseBodyMethodProcessor
* @RequestPart RequestPartMethodArgumentResolver
* @RequestHeader RequestHeaderMethodArgumentResolver和RequestHeaderMapMethodArgumentResolver
* @CookieValue ServletCookieValueMethodArgumentResolver
* @Value ExpressionValueMethodArgumentResolver
* @SessionAttribute SessionAttributeMethodArgumentResolver
* @RequestAttribute RequestAttributeMethodArgumentResolver