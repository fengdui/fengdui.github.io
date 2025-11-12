---
title: "shiro并发创建多个session问题记录"
date: "2025-11-12"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

项目里面用到了shiro的自定义filter  
session的创建和spring security不一样 他不是针对登录接口的  
是在你访问所有被你指定filter拦截的接口里面 发现subject没有认证 isAccessAllowed方法里面我们一般会加这个  

```
Subject subject = getSubject(request, response);
// 1. 检查是否登录（认证或记住我）
if (!subject.isAuthenticated() && !subject.isRemembered()) {
    logger.warn("用户未认证，拒绝访问: {}", requestURI);
    return false;
}
```
如果isAccessAllowed方法返回false 会继续请求onAccessDenied方法  
在这个方法里面我们会请求executeLogin去认证 生成session并且持久化  
所以一大堆请求打过来的时候有好几个请求判断没认证 去登录 并且同时生成了好几个session  

解决方法是加分布式锁  
或者即使创建了多个session 你持久化的key改成userid而不是sessionid 需要自定义sessionFactory  
我直接把session持久化关闭了  
```
DefaultSubjectDAO subjectDAO = new DefaultSubjectDAO();
DefaultSessionStorageEvaluator evaluator = new DefaultSessionStorageEvaluator();
evaluator.setSessionStorageEnabled(false);
subjectDAO.setSessionStorageEvaluator(evaluator);
securityManager.setSubjectDAO(subjectDAO);
```
不过这样每次请求都要去认证了  

