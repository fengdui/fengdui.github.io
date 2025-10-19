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