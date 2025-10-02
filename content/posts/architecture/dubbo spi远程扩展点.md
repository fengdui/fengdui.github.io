---
title: "dubbo spi远程扩展点"
date: "2019-10-25"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


# 背景
同一个dubbo接口有多个不同的实现类, 多个实现类由不同的团队维护，属于不同的微服务部署在不同的节点，因此需要根据业务标识找到特定的实现，找不到走默认的实现。
第一版只支持spring+dubbo, 后续需要支持不同的容器guice,多注册中心etcd,zk,eureka, nacos, 多rpc框架feign, grpc等。

# 方案1  修改注册中心实现

* 1、根节点：dubbo
* 2、一级子节点：提供服务的服务名
* 3、二级子节点：固定的四个子节点：分别为：consumers、configurators、routers、providers
* dubbo的Root层是根目录，通过<dubbo:registry group="dubbo" />的“group”来设置zookeeper的根节点，缺省值是“dubbo”。
* Service层是服务接口的全名。
* Type层是分类，一共有四种分类，分别是providers（服务提供者列表）、consumers（服务消费者列表）、routes（路由规则列表）、configurations（配置规则列表）。
* URL层：根据不同的Type目录：可以有服务提供者 URL 、服务消费者 URL 、路由规则 URL 、配置规则 URL 。不同的Type关注的URL不同。
* 所以可以自定义注册中心 加一层节点 表示spi的标识，要在接口的下一层，一个接口有多个spi实现。
# 方案2
* group 不推荐 可能内部用来区分环境
# 方案3
* 路由 但是需要在admin控制台配置 比较麻烦 不推荐
# 方案4 agent 字节码技术 筛选符合的spi提供者 业务团队 不能这么搞
* 其实有中间件团队做了字节码增强，怕冲突。
# 方案5 筛选符合的spi提供者 不过不是通过字节码 而是通过dubbo的spi扩展点机制 扩展了一些源码
* 采用装饰者模式，对所有的clusterinvoker包装了一层
* 但是这样会有一个问题，实际上会维护了多个不需要的dubboinvoker
* spiFailover=com.xxx.spi.core.framework.call.dubbo.cluster.SpiFailoverCluster
* spiDubbo=com.xxx.spi.core.framework.call.dubbo.SpiDubboRouterFactory


```js
AbstractSpiClusterInvoker 基类
暂时没有用到
package com.souche.spi.core.framework.call.dubbo.cluster;

import com.alibaba.dubbo.common.Constants;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.Result;
import com.alibaba.dubbo.rpc.RpcException;
import com.alibaba.dubbo.rpc.cluster.Directory;
import com.alibaba.dubbo.rpc.cluster.LoadBalance;
import com.alibaba.dubbo.rpc.cluster.Router;
import com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker;
import com.alibaba.dubbo.rpc.support.RpcUtils;

import java.util.List;

/**
 *
 * 因为java的单继承 所以该类无法被继承
 * @param <T>
 */
@Deprecated
public abstract class AbstractSpiClusterInvoker<T> extends AbstractClusterInvoker<T> {

    private Router router = ExtensionLoader.getExtensionLoader(Router.class).getExtension("spiDubbo");

    public AbstractSpiClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result invoke(final Invocation invocation) throws RpcException {

        checkWheatherDestoried();

        LoadBalance loadbalance;

        List<Invoker<T>> invokers = list(invocation);
        if (invokers != null && invokers.size() > 0) {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(invocation.getMethodName(), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
        } else {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
        }
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        invokers = beforeDoInvoke(invokers);
        return doInvoke(invocation, invokers, loadbalance);
    }

    /**
     * 根据上下文过滤bizCode
     * @return
     */
    private List<Invoker<T>> beforeDoInvoke(List<Invoker<T>> invokers) {

        return router.route(invokers, null, null);
    }
}
```


```js
SpiFailoverClusterInvoker 针对Failover的实现类 继承FailoverClusterInvoker
主要是加了一句invokers = beforeDoInvoke(invokers);
package com.xxx.spi.core.framework.call.dubbo.cluster;

import com.alibaba.dubbo.common.Constants;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.Result;
import com.alibaba.dubbo.rpc.RpcException;
import com.alibaba.dubbo.rpc.cluster.Directory;
import com.alibaba.dubbo.rpc.cluster.LoadBalance;
import com.alibaba.dubbo.rpc.cluster.Router;
import com.alibaba.dubbo.rpc.cluster.RouterFactory;
import com.alibaba.dubbo.rpc.cluster.support.FailoverClusterInvoker;
import com.alibaba.dubbo.rpc.support.RpcUtils;

import java.util.List;

public class SpiFailoverClusterInvoker<T> extends FailoverClusterInvoker<T> {

    protected Router router = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension("spiDubbo").getRouter(null);

    public SpiFailoverClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result invoke(final Invocation invocation) throws RpcException {

        checkWheatherDestoried();

        LoadBalance loadbalance;

        List<Invoker<T>> invokers = list(invocation);
        if (invokers != null && invokers.size() > 0) {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(invocation.getMethodName(), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
        } else {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
        }
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        invokers = beforeDoInvoke(invokers);
        return doInvoke(invocation, invokers, loadbalance);
    }

    /**
     * 根据上下文过滤bizCode
     * @return
     */
    private List<Invoker<T>> beforeDoInvoke(List<Invoker<T>> invokers) {

        return router.route(invokers, null, null);
    }
}
```


```js
SpiFailoverCluster 工厂 获取实现类SpiFailoverClusterInvoker
package com.xxx.spi.core.framework.call.dubbo.cluster;

import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.RpcException;
import com.alibaba.dubbo.rpc.cluster.Cluster;
import com.alibaba.dubbo.rpc.cluster.Directory;

public class SpiFailoverCluster implements Cluster {

    public final static String NAME = "spiFailover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new SpiFailoverClusterInvoker<>(directory);
    }
}


SpiDubboRouterFactory 用来获取具体的router逻辑实现类。
package com.xxx.spi.core.framework.call.dubbo;

import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.rpc.cluster.Router;
import com.alibaba.dubbo.rpc.cluster.RouterFactory;

public class SpiDubboRouterFactory implements RouterFactory {
    @Override
    public Router getRouter(URL url) {
        return new SpiDubboRouter();
    }
}


SpiDubboRouter 某个router路由的实现类
package com.xxx.spi.core.framework.call.dubbo;

import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.common.utils.CollectionUtils;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.RpcException;
import com.alibaba.dubbo.rpc.cluster.Router;
import com.xxx.spi.core.framework.BusinessContext;

import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

public class SpiDubboRouter implements Router {

    @Override
    public URL getUrl() {
        throw new IllegalArgumentException();
    }

    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        if (CollectionUtils.isEmpty(invokers)) {
            return invokers;
        }
        //如果为空说明是正常的dubbo调用
        if (Objects.isNull(BusinessContext.getBizCode())) {
            return invokers;
        } else {
            String currentBizCode = BusinessContext.getBizCode();
            List<Invoker<T>> result = invokers.stream().filter((invoker) -> currentBizCode.equals(invoker.getUrl().getParameter(Constants.TAG_KEY))).collect(Collectors.toList());
            return result;
        }
    }

    @Override
    public int compareTo(Router o) {
        return 0;
    }
}
```
# 方案6 打算扩展url的机制，加上spi的标识 类似于dubbo新版的路由功能
* 但是发现提供者虽然把标识附加到url上 注册到zk上 但是消费的时候没有用标识去找到对应的提供者
* dubbo新版的路由功能的标识是写死去获取这个标识再去发现提供者的
* 所以采用方案5 使用的时候才去筛选


<https://github.com/dsc-cmt/Extension>     
<https://github.com/shengchaojie/dubbo_remote_spi>         
<https://mp.weixin.qq.com/s/iS7QjI-LSOET4cmRGo_Jow>              
https://zhuanlan.zhihu.com/p/55087695
