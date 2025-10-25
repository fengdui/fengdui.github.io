---
title: "lettuce 客户端使用报gc overhead limit exceeded"
date: "2024-11-21"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
测试环境出现使用redis lettuce客户端gc overhead limit exceeded问题，其他接口都正常访问。
使用jprofile分析.hprof文件, 发现有2个大对象，

![img_26.png](/pic/img_26.png)
总共内存就1g, 这边就占了将近700m了。。

但是奇怪的是为什么就redis lettuce客户端报错，其他接口都访问正常，猜想是lettuce需要的内存更多，而且是一整晚都报这个异常，按理说内存回收了不会在报错，这个由于重启了没法在jmap获取dump文件了，猜想是内存都用完了，接下来只能看看具体是做什么操作报的oom异常。

![img_27.png](/pic/img_27.png)
日志没有很详细，只能跟踪到源码这里

```js
public static <T> T awaitOrCancel(RedisFuture<T> cmd, long timeout, TimeUnit unit) {

    try {
        if (!cmd.await(timeout, unit)) {
            cmd.cancel(true);
            throw new RedisCommandTimeoutException();
        }
        return cmd.get();
    } catch (RuntimeException e) {
        throw e;
    } catch (ExecutionException e) {

        if (e.getCause() instanceof RedisCommandExecutionException) {
            throw new RedisCommandExecutionException(e.getCause().getMessage(), e.getCause());
        }

        throw new RedisException(e.getCause());
    } catch (InterruptedException e) {

        Thread.currentThread().interrupt();
        throw new RedisCommandInterruptedException(e);
    } catch (Exception e) {
        throw new RedisCommandExecutionException(e);
    }
}
```

catch了一个ExecutionException异常，这个因为是同步的逻辑，所以通过future.get得到结果，然后只拿到了一个gc overhead limit exceeded，具体哪一行需要看后面异步的AsyncCommand有没有打印日志。

大致链路是io.lettuce.core.api.sync.RedisStringCommands#set(K, V, io.lettuce.core.SetArgs) ->
io.lettuce.core.api.StatefulRedisConnection#sync 得到一个代理类，代理类实现了RedisCommands接口。

也就是说同步的逻辑是通过一个jdk动态代理FutureSyncInvocationHandler代理了一个异步的RedisAsyncCommandsImpl，也就是最终是通过它的父类去做具体的各自redis操作，
我这边是set操作，逻辑就是在io.lettuce.core.AbstractRedisAsyncCommands#set(K, V, io.lettuce.core.SetArgs)。

看完源码发现得到future返回之后几乎没打日志，也就是在遇到异常，对future进行io.lettuce.core.protocol.RedisCommand#completeExceptionally之前，没打什么日志，都是debug。。。

于是将lettuce对应的日志级别调整为debug，后续复现在排查，因为服务已经被重启了，通过arthas的方式也不行了。

总结：
1.对于该打的日志要打，便于排查问题。
2.遇到问题不要重启服务器，保留现场。
