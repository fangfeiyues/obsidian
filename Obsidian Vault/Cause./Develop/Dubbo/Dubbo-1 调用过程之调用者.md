
## 为什么是Dubbo？

 - **Dubbo vs  HTTP**
	1.  稳定性。http的支持是集中式的，rpc的支持是分布式的。经验上，rpc的稳定性更高一些。
	2.  稳定性。rpc在协议上实现了诸多功能如限流、熔断、降级等，这些功能在稳定性治理上是刚需。
	3.  成本。http的附加支持是有成本的，http的支持附加成本比rpc高 （rpc不需要附加成本，这些附加成本是指：LB的成本，申请域名、审批、创建http vip配置等的成本）

## 1、初始化

Dubbo调用者在正式发起网络请求之前，会有一系列准备动作，其中核心步骤如下
1.  --> 注册中心获取 Provider 地址
2.  --> 地址列表封装成 Directory，供正式请求负载使用
3.  --> 返回代理 Proxy
4.  --> ...


代码流程
invoker = refprotocol.refer(interfaceClass, url);      生成invoker，并开启netty-client  
  1.1 RegistryProtocol 注册zookeeper  
      RegistryProtocol#refer(Class<T> type, URL url)  
  1.2 ProtocolFilterWrapper 拿到DubboInvoker之后组装一链串的new Invoker(){ invoke(){ filter.invoke(next, invocation) }}  
  1.3 ProtocolListenerWrapper  
  1.4 DubboProtocol  
     1.4.0 DubboProtocol / RedisProtocol / InjvmProtocol / ThriftProtocol  
     1.4.1 DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);     即new了一个DubboInvoker  
     1.4.2 ExchangeClient = Exchanges.connect(URL, ExchangeHandler) exchanger层开始连接 + ChannelHandler的心路历程     每个interface一个client?  
           ExchangeHandler(ChannelHandler)== received到Server返回之后最后的处理步骤??  HeaderExchangeHandler - AbstractChannelHandlerDelegate - DecodeHandler  
           Transports  - ChannelHandlerDispatcher NettyClient.connect(URL url, ChannelHandler listener), - HeartbeatHandler AbstractChannelHandlerDelegate - MultiMessageHandler  
     1.4.3 开启心跳机制，打开netty客户端，数据序列化配置...  
b. (T) proxyFactory.getProxy(invoker)         把对interface接口的方法调用代理成对invoke的调用  
   2.1 JavassistProxyFactory  字节码生成代理  
   2.2 JdkProxyFactory  jdk.proxy生成代理   InvokerInvocationHandler类  
   2.3 每次都会生成一个invoker然后通过invoker对接口生成代理会不会导致初始化启动太慢？



### 服务发现


![[image-Dubbo-1 调用过程之调用者-20240420205050456.png|450]]



### 集群


### 路由


### 泛化？


### 代理







![[分层设计.png|800]]


