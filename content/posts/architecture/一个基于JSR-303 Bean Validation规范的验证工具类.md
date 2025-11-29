---
title: "一个基于JSR-303 Bean Validation规范的验证工具类"
date: "2020-01-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

ValidationUtils基于Java官方的Bean Validation API实现，核心依赖包括：  

javax.validation.Validation：创建Validator实例的工厂类  
javax.validation.Validator：执行实际验证的核心接口  
javax.validation.ConstraintViolation：表示验证约束违反的接口  

```
import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;

import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.lang.StringUtils;

public class ValidationUtils {
    /**
     * validator是线程安全的,使用同一个实例即可
     */
    public final static Validator VALIDATOR = Validation.buildDefaultValidatorFactory().getValidator();

    /**
     * 是否违反约束
     *
     * @param obj
     * @return
     */
    public static <T> boolean isViolated(T obj) {
        if (Objects.isNull(obj)) {
            return false;
        }
        return VALIDATOR.validate(obj).size() > 0;
    }

    /**
     * 返回违反约束的信息 没有则返回空字符串
     *
     * @param obj
     * @return
     */
    public static <T> String getViolationMessage(T obj) {
        if (Objects.isNull(obj)) {
            return StringUtils.EMPTY;
        }
        Set<ConstraintViolation<T>> violations = VALIDATOR.validate(obj);
        if (CollectionUtils.isEmpty(violations)) {
            return StringUtils.EMPTY;
        }
        //使用了distinct去重,防止出现大量相同错误信息(验证内嵌集合时)
        return violations.stream().map(ConstraintViolation::getMessage).distinct().collect(Collectors.joining(","));
    }

    public static <T> void assertNoViolation(T obj){
        Set<ConstraintViolation<T>> violations = VALIDATOR.validate(obj);
        if(CollectionUtils.isNotEmpty(violations)){
            throw ExceptionUtil.fail(violations.stream().map(ConstraintViolation::getMessage).distinct().collect(Collectors.joining(",")));
        }
    }
}
```
需要校验的bean需要添加校验注解，如@NotNull、@Size等 和使用springmvc时需要在Controller方法参数前添加@Valid注解效果一样  
只不过是手动触发校验，而不是在Controller方法参数前添加@Valid注解 适合没有Controller层的场景  
