---
title: "springmvc源码阅读"
date: "2015-10-11"
tags: ["架构"]
ShowToc: false
TocOpen: false
---




handlerMapping的作用
- DispatcherServlet就是一个servlet，doService方法调用doDispatcher方法
- 调用DispatcherServlet的getHandler(HttpServletRequest request)方法得到一个HandlerExecutionChain
- DispatcherServlet的getHandler方法其实调用了handlerMapping的getHandler方法，迭代DispatcherServlet的handlerMappings，
- 然后如果有一个handlerMapping的getHandler返回的HandlerExecutionChain不为空，就使用该HandlerExecutionChain。
- 这样就在DispatcherServlet的doDispatcher方法得到了一个请求对应的HandlerExecutionChain。
- HandlerExecutionChain里面封装了具体的controller，还有一组拦截器。
- 这里可能不止有一个handlerMapping，所以是list类型。
- handlerMapping维护了url到controller映射的一个map，根据url得到controller之后在封装成HandlerExecutionChain返回。
- 已SimpleUrlHandlerMapping为例，从initApplicationContext -》registerHandlers，registerHandlers方法调用了父类AbstractUrlHandlerMapping的registerHandlers方法，
- 从容器中得到bean，然后将url-bean放到名为handlerMap的LinkedHashMap中。
- handlerMapping是接口，只有子类AbstractHandlerMapping实现了方法
- 又调用了子类AbstractUrlHandlerMapping的getHandlerInternal方法
- 然后调用了AbstractUrlHandlerMapping的lookupHandler方法，其实就是从之前的handlerMap中得到handler。
- 得到之后封装成HandlerExecutionChain
- 这就是handlerMapping的作用，就是根据url得到controller，url和controller的映射关系之前都被保存在handlerMap中，得到之后封装成HandlerExecutionChain。
  handleradapt的作用
- 使用handleradapt和handlerMapping一样，也是遍历handlerAdapters list，如果有一个adapter的supports方法返回true，就使用这个adapter。
- 在dispatcherservlet中调用了adapter的handle方法，把HandlerExecutionChain的handler传进去。
- err = err1.handle(processedRequest, response, mappedHandler.getHandler());
- 什么都不配置，使用springmvc的dispatcherservlet.properties文件中的配置。
- Default implementation classes for DispatcherServlet's strategy interfaces.
- Used as fallback when no matching beans are found in the DispatcherServlet context.
- Not meant to be customized by application developers.
- org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
- org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver
- org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
- org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping
- org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
- org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
- org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter
- org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
- org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
- org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
- org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
- org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver
- org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
- DefaultAnnotationHandlerMapping 和 AnnotationMethodHandlerAdapter 的使用已经过时！
- 请使用..
- spring 3.1 开始我们应该用
- RequestMappingHandlerMapping 来替换 DefaultAnnotationHandlerMapping，
- RequestMappingHandlerAdapter 来替换 AnnotationMethodHandlerAdapter。
- 也可以配置
- mvc:annotation-driven/
- 申明，会为你注册