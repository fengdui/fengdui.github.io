---
title: "dubbo源码阅读"
date: "2016-10-17"
tags: ["架构"]
ShowToc: false
TocOpen: false
---



# dubbo服务暴露过程
- serviceconfig#doExportUrlsFor1Protocol方法会先做一个代理，ref就是业务实现类，
```
* Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
```
- 使用代理 javassist或者jdk代理转换成一个invoker
- 接下来会使用protocol.export(wrapperInvoker); 将invoker转换成exporter，
- 注意一下ServiceConfig里面的protocol是ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
- 返回的结果是protocol&Adaptive ->protocolListenerWraper->protocolFilterWraper->DubboProtocol
- protocolFilterWraper#export方法会调用buildInvokerChain，
- DubboProtocol的export方法将invoker转换成了exporter，然后返回
- DubboProtocol内部维护了一个Map<String, ExchangeServer> serverMap，调用server = Exchangers.bind(url, requestHandler); 得到一个ExchangeServer，它的子类有ExchangeClient和ExchangeServer2种，内部是通过HeaderExchanger得到的。
- HeaderExchangeServer内部有一个server，默认是nettyServer，它是调用Transport生成的，
- 总体来说，就是DubboProtocol维护了HeaderExchangeServer，ExchangeServer维护了一个nettyServer。
- nettyserver的父类AbstractEndpoint维护了一个codec2
- 还是以dubbo协议为例，默认使用的是DubboCountCodec，它内部又使用了DubboCodec，encode和decode方法后获取
- Serialization进行序列化和反序列化，CodecSupport.getSerialization(channel.getUrl());
# dubbo消费者流程
- ReferenceConfig#createProxy 创建代理
- refprotocol#refer 返回invoker
- refprotocol为Protocol$Adpative，是动态生成的
```
  com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);  
  return extension.refer(arg0, arg1);
```
- 因此实际上又获取了一个protocol，然后调用它的refer方法，这个protocol是ProtocolListenerWrapper
- 简单说一下这里ExtensionLoader#getExtension 参数是registry，首先从cachedInstances缓存获取，没有使用createExtension创建
- createExtension方法也会从EXTENSION_INSTANCES缓存中获取，没有的话会实例化一个（这里实力化了一个RegistryProtocol）
- 然后调用injectExtension方法进行依赖注入
- 然后使用cachedWrapperClasses包装，这里cachedWrapperClasses有两个
- ProtocolListenerWrapper#refer 方法
```
  public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
  if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
  return protocol.refer(type, url);
  }
  return new ListenerInvokerWrapper<T>(protocol.refer(type, url),
  Collections.unmodifiableList(
  ExtensionLoader.getExtensionLoader(InvokerListener.class)
  .getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)));
  }
```
- 实际上又调用了ProtocolFilterWrapper#refer
```
  public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
  if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
  return protocol.refer(type, url);
  }
  return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
  }
```

- 又调用了RegistryProtocol#refer
- 这里创建了一个RegistryDirectory subscribe开始订阅服务，实际上还是使用了ZooKeeperRegistry注册。
- 请求过程
- InvokerInvocationHandler#invoke方法
- MockClusterInvoker#invoke方法
- AbstractClusterInvoker#invoke方法 这里会获取负载均衡类loadbalance
- 根据配置例如FailoverClusterInvoker#doInvoke方法 这里会做负载均衡
- RegistryDirectory的静态内部类InvokerDelegete#invoke 这里是个拦截器链 是在ProtocolFilterWrapper类的buildInvokerChain方法创建的
- 我这里经过了几个filter 最后到达了DubboInvoker(dubbo协议)
- client一个线程调用远程接口，会生成一个唯一的id，并且封装调用信息和处理结果的回调对象callback成一个object，存在concurrenthashmap，使用iosession.write(connrequest)异步发送出去，当前线程再使用callback的get()方法获取远程返回的结果，在get方法内部，使用synchronize获取回调对象callback的锁，再检车是否已经获取结果，如果没有，则调用callback的wait()方法，释放callback的锁，线程等待。
- 服务端将结果发送client后，clientsocket连接上监听线程，获取id，再从concurrenthashmap找到object，并且将结果存在callback里面，监听线程获取callback的锁，会notifyall(),继续执行
# dubbo extensionLoader
- ExtensionLoader用法都是这样的形式
- ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()
- 直接看ServiceConfig的申明变量，
```
  public class ServiceConfig<T> extends AbstractServiceConfig {
  private static final long   serialVersionUID = 3033787999037024738L;
  private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
  private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
  private static final Map<String, Integer> RANDOM_PORT_MAP = new HashMap<String, Integer>();
```
- ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()，返回的Protocol是什么？
- 以及Exporter<?> exporter = protocol.export(invoker);发生了什么。
- getAdaptiveExtension
- 从缓存cachedAdaptiveInstance中获取，如果没有调用createAdaptiveExtension获取。
- createAdaptiveExtension
- 调用getAdaptiveExtensionClass得到class，然后newinstance生成对象。
- getAdaptiveExtensionClass
- 先调用一下getExtensionClasses方法，然后如果cachedAdaptiveClass不为空就用这个class，否则使用createAdaptiveExtensionClass方法
- cachedAdaptiveClass的设置看下面的loadExtensionClasses方法，这个方法的调用顺序是getExtensionClasses-》getExtensionClasses-》loadExtensionClasses
- getExtensionClasses
- 从缓存cachedClasses获取classes，如果为空则调用loadExtensionClasses方法加载
- loadExtensionClasses
- 获取Protocol接口的SPI注解，然后将cachedDefaultName设为SPI注解的value
- @SPI("dubbo")
- public interface Protocol
- 然后在读取本地文件配置的class信息
```
  private static final String SERVICES_DIRECTORY = "META-INF/services/";
  private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
  private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";
```
- 具体根据什么条件选择，暂时没有看，我这里返回了7个，如下，但是getAdaptiveExtensionClass方法并没有使用这个返回值
- 如果类如果带有Adaptive注解，会把cachedAdaptiveClass设置为这个class，默认是都没有带这个注解的
- createAdaptiveExtensionClass
- 默认情况下，第一次都会使用这个方法返回的class，这个方法使用了createAdaptiveExtensionClassCode方法生成一个string描述的对象，
- code中包含了需要生成的对象的信息
- createAdaptiveExtensionClassCode
- 完全就是用stringbuilder拼接一个对象，66666
```
  codeBuidler.append("package " + type.getPackage().getName() + ";");
  codeBuidler.append("\nimport " + ExtensionLoader.class.getName() + ";");
  codeBuidler.append("\npublic class " + type.getSimpleName() + "$Adpative" + " implements " + type.getCanonicalName() + " {")
```
- 这三行等于
- package com.alibaba.dubbo.rpc;
- import com.alibaba.dubbo.common.extension.ExtensionLoader;
- public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
- 然后下面一大堆，不说了，结果就是
- 对于protocol
```
  package com.alibaba.dubbo.rpc;
  import com.alibaba.dubbo.common.extension.ExtensionLoader;
  public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
  public void destroy() {throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
  }
  public int getDefaultPort() {throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
  }
  public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
  if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
  if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
  String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
  com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
  return extension.export(arg0);
  }
  public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
  if (arg1 == null) throw new IllegalArgumentException("url == null");
  com.alibaba.dubbo.common.URL url = arg1;
  String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
  com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
  return extension.refer(arg0, arg1);
  }
  }
```
- 类似的对于ProxyFactory
```
  package com.alibaba.dubbo.rpc;
  import com.alibaba.dubbo.common.extension.ExtensionLoader;
  public class ProxyFactory$Adpative implements com.alibaba.dubbo.rpc.ProxyFactory {
  public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
  if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
  if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
  String extName = url.getParameter("proxy", "javassist");
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
  com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
  return extension.getProxy(arg0);
  }
  public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws java.lang.Object {
  if (arg2 == null) throw new IllegalArgumentException("url == null");
  com.alibaba.dubbo.common.URL url = arg2;
  String extName = url.getParameter("proxy", "javassist");
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
  com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
  return extension.getInvoker(arg0, arg1, arg2);
  }
  }
```
- 对于registryfactory
```
  package com.alibaba.dubbo.registry;
  import com.alibaba.dubbo.common.extension.ExtensionLoader;
  public class RegistryFactory$Adpative implements com.alibaba.dubbo.registry.RegistryFactory {
  public com.alibaba.dubbo.registry.Registry getRegistry(com.alibaba.dubbo.common.URL arg0) {
  if (arg0 == null) throw new IllegalArgumentException("url == null");
  com.alibaba.dubbo.common.URL url = arg0;
  String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" + url.toString() + ") use keys([protocol])");
  com.alibaba.dubbo.registry.RegistryFactory extension = (com.alibaba.dubbo.registry.RegistryFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.registry.RegistryFactory.class).getExtension(extName);
  return extension.getRegistry(arg0);
  }
  }
```
- 对于cluster
```
  package com.alibaba.dubbo.rpc.cluster;
  import com.alibaba.dubbo.common.extension.ExtensionLoader;
  public class Cluster$Adpative implements com.alibaba.dubbo.rpc.cluster.Cluster {
  public com.alibaba.dubbo.rpc.Invoker join(com.alibaba.dubbo.rpc.cluster.Directory arg0) throws com.alibaba.dubbo.rpc.cluster.Directory {
  if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument == null");
  if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
  String extName = url.getParameter("cluster", "failover");
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.cluster.Cluster) name from url(" + url.toString() + ") use keys([cluster])");
  com.alibaba.dubbo.rpc.cluster.Cluster extension = (com.alibaba.dubbo.rpc.cluster.Cluster)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
  return extension.join(arg0);
  }
  }
```
- 对于transport
```
  package com.alibaba.dubbo.remoting;
  import com.alibaba.dubbo.common.extension.ExtensionLoader;
  public class Transporter$Adaptive implements com.alibaba.dubbo.remoting.Transporter {
  public com.alibaba.dubbo.remoting.Client connect(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
  if (arg0 == null) throw new IllegalArgumentException("url == null");
  com.alibaba.dubbo.common.URL url = arg0;
  String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([client, transporter])");
  com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
  return extension.connect(arg0, arg1);
  }
  public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
  if (arg0 == null) throw new IllegalArgumentException("url == null");
  com.alibaba.dubbo.common.URL url = arg0;
  String extName = url.getParameter("server", url.getParameter("transporter", "netty"));
  if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([server, transporter])");
  com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
  return extension.bind(arg0, arg1);
  }
  }
```
- 这里返回的Protocol是什么已经知道了，就是自己生成字节码然后反射生成的，
- 生成之后查看export方法发现获取了一个Protocol，然后在调用这个Protocol的export方法，
- ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
- String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
- url是从arg0获取的 arg0是Invoker类型，
- 从Invoker的url中获取protocol类型，如果没有默认是dubbo，我这里返回的是registry。
- 然后调用getExtension -》createExtension，返回的的是RegistryProtocol。
- RegistryProtocol的export注册服务。
- 可以发现所有的XXX$Adpative都是动态生成的，然后调用他的方法还会根据不同的参数生成一个XXX对象，真正的调用时这个XXX对象。