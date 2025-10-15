---
title: "netty源码分析"
date: "2016-11-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


- ServerBootstrap.bind()方法会实例化一个channel，这个channel的类型是通过channel方法指定的，例如ServerBootstrap.channel(NioServerSocketChannel.class)
- 指定的同时会创建一个ChannelFactory工厂，默认是ReflectiveChannelFactory，然后bind()方法通过这个ChannelFactory实例化NioServerSocketChannel，在channel的构造函数（见AbstractChannel）会实例化DefaultChannelPipeline。
- 实例化之后会初始化 init方法，获取channel的pipeline，添加handler处理器，
- ChannelInitializer是在channel注册channelRegistered时调用的
- p.addLast(new ChannelInitializer<Channel>() {
```java
    @Override
    public void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }
        pipeline.addLast(new ServerBootstrapAcceptor(
                currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
    }

```

- 然后使用bossGroup注册，bossGroup是我们创建的NioEventLoopGroup，它维护了一组NioEventLoop，每个维护了一个selector选择器，NioEventLoopGroup。
- 选择一个NioEventLoop来把channel注册到这个NioEventLoop的选择器上。