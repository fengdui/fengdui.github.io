---
title: "dubbo隐式传参结合threadlocal透传信息"
date: "2018-11-03"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


saas要做连锁店版本 部分表数据是全局多个门店共享的 整个系统改动过大    
直接使用dubbo隐式传参传用户和门店的信息 然后设置在threadlocal里面 需要使用的时候从threadlocal里面取 判断是否是连锁店  

```
@Activate(group = { Constants.SERVER_KEY, Constants.CONSUMER})
public class DubboShopInfoFilter implements Filter {

    @Override
    @SuppressWarnings("unchecked")
    public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
        CommonUserInfo commonUserInfo = UserInfoContext.getUserInfoCache();
        RpcContext.getContext().setAttachment("SHOP_INFO", JSON.toJSONString(commonUserInfo));

        return invoker.invoke(invocation);
    }
}
```
```
@Slf4j
@Component("dubboServiceInterceptor")
public class DubboServiceInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        try {
            //打印接口请求参数
            //printRequestLog(invocation);
            String context = RpcContext.getContext().getAttachment("SHOP_INFO");
            CommonUserInfo commonUserInfo = JSON.parseObject(context, CommonUserInfo.class);
            if (commonUserInfo != null) {
                UserInfo userInfo = new UserInfo();
                userInfo.setChainShop(commonUserInfo.isChainShop());
                userInfo.setShopId(commonUserInfo.getShopId());
                userInfo.setUserId(commonUserInfo.getUserId());
                userInfo.setChainShopId(commonUserInfo.getChainShopId());
                userInfo.setHeadOffice(commonUserInfo.isHeadOffice());
                UserUtils.setUserInfoCache(userInfo);
            }
            return invocation.proceed();
        } catch (Exception e) {
            log.error("Exception:", e);
            Object r = exceptionProcessor(invocation, e);
            return r;
        } finally {
            UserUtils.cleanCurrentUserInfoCache();
        }
    }
}
```
首先在每个请求的filter里面 拿到用户和门店信息 然后设置到threadlocal里面 记得请求结束的时候要清理掉threadlocal
通过上面扩展点 可以在服务调用的时候 把门店信息设置到attachment
rpc拦截器
根据业务自行处理 从attachment里面取门店信息 然后设置到threadlocal里面
其他项目用了spring security 也需要从threadlocal里面取用户信息 所以要在security的filter里面 也设置一下用户信息
