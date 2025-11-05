---
title: "shiro源码分析"
date: "2016-08-26"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

# 认证流程详解
认证（Authentication）就是验证用户身份的过程。当你调用 subject.login(token) 时，Shiro 内部会经历以下关键步骤：

提交登录信息：创建 UsernamePasswordToken 对象（AuthenticationToken 接口的一个实现），封装用户名和密码。

委托认证：Subject 的实现类将调用 securityManager.login(this, token)，将认证任务委托给核心安全管理器。

协调认证：SecurityManager 使用 Authenticator（通常是 ModularRealmAuthenticator）来协调认证过程。ModularRealmAuthenticator 的 doAuthenticate 方法会根据配置的 Realm 数量决定如何执行认证。

获取凭据：Authenticator 会调用你配置的 Realm（特别是 AuthenticatingRealm 子类）的 getAuthenticationInfo(token) 方法。在这个方法中，会执行到你自定义的 doGetAuthenticationInfo 方法，在这里你根据用户名从数据源（如数据库）中获取正确的凭证信息（例如密码和盐），并封装成 SimpleAuthenticationInfo 对象返回。

凭据比对：获取到 AuthenticationInfo 后，会调用 CredentialsMatcher 的 doCredentialsMatch 方法（例如 HashedCredentialsMatcher），来比较 Token 中的凭证（用户输入的）和 AuthenticationInfo 中的凭证（系统存储的）是否匹配。我们通常会通过继承 HashedCredentialsMatcher 并重写此方法来定制自己的密码比较逻辑。

声明认证成功：如果凭证匹配成功，认证通过，用户身份得到确认。如果过程中任何一步失败（如未知账号、密码错误），Shiro 会抛出相应的 AuthenticationException。

# 授权流程解析
授权（Authorization），即访问控制，检查已认证的用户是否拥有执行某项操作或访问某些资源的权限。当你调用如 subject.hasRole("admin") 或 subject.isPermitted("user:delete") 时，流程如下：    

授权请求：Subject 将授权请求委托给 SecurityManager。

协调授权：SecurityManager 使用 Authorizer（通常也是 ModularRealmAuthorizer）来协调授权过程。

查询角色/权限：Authorizer 会调用你配置的 Realm（特别是 AuthorizingRealm 子类）的相应方法，例如 doGetAuthorizationInfo(principal) 或特定的权限检查方法。在这个方法中，会执行到你自定义的授权逻辑，你根据用户身份（从 principal 中获取，通常是用户名或用户ID）去查询该用户所拥有的角色和权限信息，并返回。

判断授权结果：Authorizer 根据 Realm 返回的信息，判断用户是否拥有所需的角色或权限。

# 一个demo


用户登录 → 生成访问令牌（32位MD5字符串）  
前端存储访问令牌  
后续请求携带: Authorization: Bearer <访问令牌>  
后端继承AuthenticatingFilter 实现一个过滤器 例如Oauth2Filter  
过滤器prehandler 会执行return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);  
在上面两个方法里面调用executeLogin基类接口 这个接口会去请求 createToken方法，需要实现createToken方法去获取header里面的token  
executeLogin还会做登录  
```
try {
    Subject subject = this.getSubject(request, response);
    subject.login(token);
    return this.onLoginSuccess(token, subject, request, response);
} catch (AuthenticationException e) {
    return this.onLoginFailure(token, e, request, response);
}
```
subject.login会委托SecurityManager 调用 Realm 认证  
所以需要实现一个 比如Oauth2Realm doGetAuthenticationInfo 方法校验token的有效性 token和它的失效时间存在数据库里面  
如果token有效拿到用户信息 返回AuthenticationInfo  
AuthenticationInfo你设置什么principal subject拿到的就是这个你自定义的对象 可以强转  
UserDetail user = (UserDetail) subject.getPrincipal();  

Oauth2Realm实现doGetAuthorizationInfo 方法从数据库里面查用户的权限  
流程:  
用户代码: subject.hasPermission("sys:user:list")  
↓
DelegatingSubject.hasPermission()  
↓
DefaultSecurityManager.isPermitted()  
↓
ModularRealmAuthorizer.isPermitted()  
↓
AuthorizingRealm.isPermitted()  
↓
AuthorizingRealm.getAuthorizationInfo()  
↓
检查缓存 -> 缓存命中则返回  
↓
缓存未命中 -> doGetAuthorizationInfo() // 你的实现  
↓
结果放入缓存  
↓
AuthorizingRealm.isPermitted(permission, info)  
↓
返回权限检查结果  

Shiro是提供了自己的Session管理机制 如果要实现分布式session需要自己实现sessionDao  