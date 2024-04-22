
## 为什么是Dubbo？

 - **Dubbo vs  HTTP**
	1.  稳定性。http的支持是集中式的，rpc的支持是分布式的。经验上，rpc的稳定性更高一些。
	2.  稳定性。rpc在协议上实现了诸多功能如限流、熔断、降级等，这些功能在稳定性治理上是刚需。
	3.  成本。http的附加支持是有成本的，http的支持附加成本比rpc高 （rpc不需要附加成本，这些附加成本是指：LB的成本，申请域名、审批、创建http vip配置等的成本）



![[image-Dubbo-1 调用过程之调用者-20240420205050456.png|450]]


设计理念？抽象 --> 实现

## 1、创建Invoker

-  **注册配置**

```xml
<dubbo:application name="consumer-demo"/>
<dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>
<dubbo:reference interface="com.example.service.UserService" id="userService" check="false" url="dubbo://userService.provider-demo"/>
```


-  **核心流程**

```text
ReferenceConfig#createInvoker
	-> RegistryProtocol#refer(Class<T>, url)           -- 1. 生成集合类invoker
		-> RegistryProtocol#getRegistry     -- 1.1 获取注册类
			-> AbstractRegistryFactory#getRegistry  -- 管理注册类
				-> ZookeeperRegistryFactory#createRegistry 
				-> NacosRegistryFactory#createRegistry
		-> RegistryDirectory<T>(type, url)  -- 1.2 节点->目录
		-> FailbackRegistry#register(url)   -- 1.3 消费者节点注册
		-> RegistryDirectory#subscribe(URL) -- 1.4 生产者节点订阅（订阅->invoker）
		    -> FailbackRegistry.subscribe()    - 1.41 Failback正常失败
				-> ZookeeperRegistry.doSubscribe  - 1.411 以zk为例
				    -> zkClient.create(path, false)  - 消费者节点创建
				    -> FailbackRegistry.notify       - Failback通知（通知->invoker）
					    -> AbstractRegistry.notify()  
					        -> RegistryDirectory.notify(List<URL>)  - Listener监听url
						         -> refreshInvoker(invokerUrls)      - Listener刷新url    
						            -> RegistryDirectory.toInvokers   - 生成新的invokers!  
							            -> ProtocolListenerWrapper#refer - invoker的封装
							            -> ProtocolFilterWrapper#refer
							            -> DubboProtocol#refer
								            -> DubboProtocol#getClients
										        -> ...NettyClient#doOpen  - Netty连接
										-> Map<String, Invoker<T>>构造到RegistryDirectory
						            -> destroyUnusedInvokers             - 销毁老的invokers
		-> FailbackCluster#join(Directory)  -- 1.5 目录->集群
	-> AbstractProxyFactory#getProxy(Invoker, generic) -- 2. 生成invoker代理类
		-> 2.1 JavassistProxyFactory#getProxy
		-> 2.2 JdkProxyFactory#getProxy
```


### 1.1、Registry（注册） & subscribe（订阅）

	Dubbo是在消费者初始化refer引用后，到注册中心创建调用者节点并订阅提供者节点后，再根据监听的订阅url信息开始封装invoker

#### Zookeeper

	![[image-Dubbo-1 调用过程之调用者-20240420225304815.png|450]]

-  **流程**
	1、服务提供者启动时:  向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址。
	2、服务消费者启动时:  订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址，并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址（用于监控）
	3、 监控中心启动时: 订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者 URL 地址

#### Nacos

#### Redis


### 1.2、Invoker（服务端）

	调用者在监听到调用者url节点后，更新本地 invoker 生成 Map<url, invoker>，并注入到 Directory

#### 跟服务端预连接（非懒加载）

	Netty打开连接 DubboProtocol#getSharedClient

#### 监听器Listener & 拦截器Filter

	在生成 DubboInvoker 核心类后，会对其封装一层拦截

- **核心过滤器**

	1.  CacheFilter（缓存过滤器）
	2.  ActiveLimitFilter（并发调用限制过滤器）：在请求的URL中
	3.  GenericImplFilter（泛化过滤器）：用消费者提供的元数据组装 Dubbo请求体 RpcInvocation，同样在服务端 GenericFilter 过滤器反序列化


-  **自定义过滤器**
	用户可根据需要，扩展 Filter SPI 的自定义过滤器

### 1.3、Directory（目录）

	目录维护Invoker列表 Map<String, Invoker<T>> urlInvokerMap，是一个集合类

### 1.4、Cluster（集群）

	集群聚合了Directory，来提供各种集群降级服务

-  **1、FailoverClusterInvoker** 

	默认执行策略，快速失败集群


-  **2、FailfastClusterInvoker** 

-  **3、BroadcastClusterInvoker** 

-  **4、FailbackClusterInvoker** 

-  **5、FailfastClusterInvoker** 

-  **6、FailfastClusterInvoker** 

## 2、生成代理
常用的动态代理大概有 ASM、CGLIB、JAVAASSIST等

###  Javassist


### JdkProxy





