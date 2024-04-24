```text
proxy0#sayHello(String)
  —> InvokerInvocationHandler#invoke(Object, Method, Object[])
    —> MockClusterInvoker#invoke(Invocation)
      —> AbstractClusterInvoker#invoke(Invocation)
        —> FailoverClusterInvoker#doInvoke(Invocation, List<Invoker<T>>, LoadBalance)
          —> Filter#invoke(Invoker, Invocation)  // 包含多个 Filter 调用
            —> ListenerInvokerWrapper#invoke(Invocation)
              —> AbstractInvoker#invoke(Invocation)
                —> DubboInvoker#doInvoke(Invocation)
                  —> ReferenceCountExchangeClient#request(Object, int)
                    —> HeaderExchangeClient#request(Object, int)
                      —> HeaderExchangeChannel#request(Object, int)
                        —> AbstractPeer#send(Object)
                          —> AbstractClient#send(Object, boolean)
                            —> NettyChannel#send(Object, boolean)
                              —> NioClientSocketChannel#write(Object)
```


## 2、Request

### 2.1 Proxy

	InvokerInvocationHandler 从进入代理类开始

### 2.2 Cluster

#### 2.21 Mock

	MockClusterInvoker 是 MockClusterWrapper 作为 Cluster 的自适应切面注入时候包的

#### 2.22 Route

	从 Directory 中获取到初始化时候保存的 invoker list，但在这个过程中要脚本路由？

#### 2.23 LoadBalance

	 FailoverClusterInvoker 默认执行策略，但在这个过程中会先 select invoker 做负载均衡处理

- **随机负载（默认）**
	根据服务提供者的权重随机选择一个进行调用。如果权重相同，则等概率随机选择；如果权重不同，则按权重比例随机选择

- **一致性哈希负载**
	相同参数的请求总是分发到同一个服务提供者，当某个提供者崩溃时，原本发往该提供者的请求会基于虚拟节点平摊到其他提供者，不会引起剧烈变动

- **最近活跃负载**


-  **轮询负载均衡**
	按照请求的顺序轮流从服务提供者列表中选择一个进行调用，类似于轮询的方式。它也考虑了权重，如果权重相同，则请求顺序轮流；如果权重不同，则权重越大的提供者越容易被选中


### 2.3 Filter & Listener

	拦截器 Filter 开始对请求的 invoker 做一些逻辑上的处理

### 2.4 Request

	到 DubboInvoker 这算是选中一个服务端的Provider了，开始真正的请求之路

-  **同步请求**

-  **异步请求**

	![[image-Dubbo-2 调用过程之请求-20240424174326417.png|500]]


-  **一次请求**

### 2.5 Netty

	Netty 发送信息

