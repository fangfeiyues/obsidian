
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
	-> RegistryProtocol#refer          -- 1. 生成消费者引用的集合类invoker
		-> RegistryProtocol#getRegistry     -- 1.1 获取注册类
			-> AbstractRegistryFactory#getRegistry  -- 管理注册类
				-> ZookeeperRegistryFactory#createRegistry 
				-> NacosRegistryFactory#createRegistry
		-> RegistryDirectory<T>(type, url)  -- 1.2 生成目录？
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
										        -> 网络?
										-> Map<String, Invoker<T>>构造到RegistryDirectory
						            -> destroyUnusedInvokers             - 销毁老的invokers
		-> FailbackCluster#join(Directory)  -- 1.5 订阅的多个消费者聚合FailbackClusterInvoker
	-> AbstractProxyFactory#getProxy(Invoker, generic) -- 2. 生成代理类
		-> 2.1 JavassistProxyFactory#getProxy
		-> 2.2 JdkProxyFactory#getProxy
```


Dubbo调用者在正式发起网络请求之前，会有一系列准备动作，其中核心步骤如下
1.  --> 注册中心获取 Provider 地址
2.  --> 地址列表封装成 Directory，供正式请求负载使用
3.  --> 返回代理 Proxy
4.  --> ...


### 服务发现&注册

Dubbo主干目前支持的主流注册中心包括 Zookeeper、Nacos、Redis，同时也支持 Kubernetes、Mesh体系的服务发现。

-  **Zookeeper**

![[image-Dubbo-1 调用过程之调用者-20240420225304815.png|450]]

-  **流程**

	-  服务提供者启动时: 向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址。
	-  服务消费者启动时: 订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址。并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址
	-  监控中心启动时: 订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者 URL 地址



### 集群


### 路由


### 泛化？


### 代理







![[分层设计.png|800]]





## 2、生成代理

