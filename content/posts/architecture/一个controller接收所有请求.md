---
title: "一个controller接收所有请求"
date: "2020-12-14"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

一个controller接收所有请求  所有的http请求都路由到这个controller  然后根据不同的path 调用不同的方法

```
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.reflect.MethodUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.MethodParameter;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;

import java.util.List;


/**
 * 一个controller接收所有请求
 */
@Slf4j
public abstract class DispatcherController {
    @Autowired
    private RequestMappingHandlerAdapter requestMappingHandlerAdapter;

    @SneakyThrows
    protected Object runMethod(String serviceName, String method, NativeWebRequest webRequest, ParamMethod queryMethod) {

        Object[] params = convertToParams(queryMethod, webRequest);
        Object result = queryMethod.invoke(params);
        return result;
    }

    @SneakyThrows
    private Object[] convertToParams(ParamMethod method, NativeWebRequest webRequest) {
        Object[] result = new Object[method.getNamedMethodParameters().length];
        for (int i = 0; i < method.getNamedMethodParameters().length; i++) {
            MethodParameter methodParameter = method.getNamedMethodParameters()[i];
            WebDataBinderFactory webDataBinderFactory = createWebDataBinderFactory(method.getHandlerMethod());
            result[i] = resolveArgument(webRequest, methodParameter, webDataBinderFactory);
        }
        return result;
    }

    private Object resolveArgument(NativeWebRequest webRequest, MethodParameter methodParameter, WebDataBinderFactory webDataBinderFactory) {
        List<HandlerMethodArgumentResolver> argumentResolvers = this.requestMappingHandlerAdapter.getArgumentResolvers();
        return argumentResolvers.stream()
                .filter(resolver -> resolver.supportsParameter(methodParameter))
                .map(resolver -> {
                    try {
                        return resolver.resolveArgument(methodParameter, new ModelAndViewContainer(), webRequest, webDataBinderFactory);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                })
                .findFirst()
                .orElse(null);
    }

    @SneakyThrows
    private WebDataBinderFactory createWebDataBinderFactory(HandlerMethod handlerMethod) {
        return (WebDataBinderFactory) MethodUtils.invokeMethod(this.requestMappingHandlerAdapter, true, "getDataBinderFactory", handlerMethod);
    }
}
```

```
import com.google.common.collect.Sets;
import lombok.Getter;
import lombok.Value;
import org.apache.commons.lang3.reflect.MethodUtils;
import org.springframework.beans.BeanUtils;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.core.MethodParameter;
import org.springframework.core.ParameterNameDiscoverer;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.method.HandlerMethod;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.Set;

@Value
public class ParamMethod {
    private static final ParameterNameDiscoverer PARAMETER_NAME_DISCOVERER = new DefaultParameterNameDiscoverer();
    private final Object bean;
    private final Method method;
    @Getter
    private final Class[] paramTypes;
    @Getter
    private final String[] paramNames;

    @Getter
    private final MethodParameter[] namedMethodParameters;

    @Getter
    private final HandlerMethod handlerMethod;

    public ParamMethod(Object bean, Method method) {
        this.bean = bean;
        this.method = method;
        this.paramTypes = method.getParameterTypes();
        this.paramNames = PARAMETER_NAME_DISCOVERER.getParameterNames(this.method);
        this.namedMethodParameters = buildParameters();
        this.handlerMethod = new HandlerMethod(bean, method);
    }

    private MethodParameter[] buildParameters() {
        MethodParameter[] result = new MethodParameter[getParamTypes().length];
        for (int i = 0; i < getParamNames().length; i++) {
            String paramName = getParamNames()[i];
            Class paramCls = getParamTypes()[i];
            Set<Class<? extends Annotation>> annotations = Sets.newHashSet();
            if (BeanUtils.isSimpleValueType(paramCls)) {
                annotations.add(RequestParam.class);
            } else {
                annotations.add(ModelAttribute.class);
            }
            MethodParameter methodParameter = new MethodParameter(getMethod(), i);
            result[i] = methodParameter;
        }
        return result;
    }

    public Object invoke(Object[] params) throws Exception {
        return MethodUtils.invokeMethod(this.bean, this.method.getName(), params);
    }
}
```