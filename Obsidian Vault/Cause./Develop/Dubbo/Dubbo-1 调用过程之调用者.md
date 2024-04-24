
## 为什么是Dubbo？

 - **Dubbo vs  HTTP**
	1.  稳定性。http的支持是集中式的，rpc的支持是分布式的。经验上，rpc的稳定性更高一些。
	2.  稳定性。rpc在协议上实现了诸多功能如限流、熔断、降级等，这些功能在稳定性治理上是刚需。
	3.  成本。http的附加支持是有成本的，http的支持附加成本比rpc高 （rpc不需要附加成本，这些附加成本是指：LB的成本，申请域名、审批、创建http vip配置等的成本）


![[image-Dubbo-1 调用过程之调用者-20240420205050456.png|450]]

https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/architecture/code-architecture/

设计理念？抽象 --> 实现

## 1、Invoker

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


### 1.1、Registry & subscribe

	消费者初始化refer引用中，会到注册中心创建调用者节点，并订阅提供者，再根据监听的生产者url开始封装invoker

#### Zookeeper

- **架构**
	![[image-Dubbo-1 调用过程之调用者-20240420225304815.png|450]]

-  **流程**
	1、服务提供者启动时:  向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址。
	2、服务消费者启动时:  订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址，并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址（用于监控）
	3、 监控中心启动时: 订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者 URL 地址


#### Redis

- **架构**

	![[image-Dubbo-1 调用过程之调用者-20240424214835325.png|600]]
	- 主 Key 为服务名和类型
	- Map 中的 Key 为 URL 地址
	- Map 中的 Value 为过期时间，用于判断脏数据，脏数据由监控中心删除


- **通知过程**
	使用 Redis 的 Publish/Subscribe 事件通知数据变更：
	- 通过事件的值区分事件类型：`register`, `unregister`, `subscribe`, `unsubscribe`
	- 普通消费者直接订阅指定服务提供者的 Key，只会收到指定服务的 `register`, `unregister` 事件
	- 监控中心通过 `psubscribe` 功能订阅 `/dubbo/*`，会收到所有服务的所有变更事件

#### Nacos


### 1.2、Invoker

	调用者在监听到调用者url节点后，更新本地 invoker 生成 Map<url, invoker>，并注入到 Directory

-  **invoker触发的各层关系**
	
	![[image-Dubbo-1 调用过程之调用者-20240424172050449.png|500]]
	
	1.  `Invoker` 是 `Provider` 的一个可调用 `Service` 的抽象，`Invoker` 封装了 `Provider` 地址及 `Service` 接口信息
	2. `Directory` 代表多个 `Invoker`，可以把它看成 `List<Invoker>` ，但与 `List` 不同的是，它的值可能是动态变化的，比如注册中心推送变更
	3. `Cluster` 将 `Directory` 中的多个 `Invoker` 伪装成一个 `Invoker`，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个
	4. `Router` 负责从多个 `Invoker` 中按路由规则选出子集，比如读写分离，应用隔离等
	5. `LoadBalance` 负责从多个 `Invoker` 中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选

#### 服务端预连接

	Netty打开连接 DubboProtocol#getSharedClient

#### Listener & Filter

	在生成 DubboInvoker 核心类后，会对其封装一层拦截

- **核心过滤器**

	1.  CacheFilter（缓存过滤器）
	2.  ActiveLimitFilter（并发调用限制过滤器）：在请求的URL中
	3.  GenericImplFilter（泛化过滤器）：用消费者提供的元数据组装 Dubbo请求体 RpcInvocation，同样在服务端 GenericFilter 过滤器反序列化


-  **自定义过滤器**
	用户可根据需要，扩展 Filter SPI 的自定义过滤器

### 1.3、Directory

	目录维护Invoker列表 Map<String, Invoker<T>> urlInvokerMap，是一个集合类

### 1.4、Cluster

	集群聚合了Directory，来提供各种集群降级服务

-  **fail over**
	默认执行策略，失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟

-  **fail fast**
	快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录

-  **fail back**
	失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作 

-  **broad cast**
	广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息

## 2、Proxy

	Dubbo 提供 javassist 和 jdk 两种代理模式可选择，其为 invoker 增强了如上集群、过滤、路由等能力

[[设计模式01-动态代理]]