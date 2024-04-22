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


## 2、请求

### 2.1 代理处理

	InvokerInvocationHandler 从进入代理类开始

### 2.2 集群处理

#### 2.21 集群Mock

	MockClusterInvoker 是 MockClusterWrapper 作为 Cluster 的自适应切面注入时候包的

#### 2.22 集群路由

	从 Directory 中获取到初始化时候保存的 invoker list，但在这个过程中要脚本路由？

#### 2.23 集群策略

默认 FailoverClusterInvoker 执行策略，但在这个过程中会先 select invoker 做负载处理



### 2.3 拦截器&监听器处理

	拦截器 Filter 开始对请求的 invoker 做一些逻辑上的处理

### 2.4 请求处理

	到 DubboInvoker 这算是选中一个服务端的Provider了，开始真正的请求之路

-  **同步请求**

-  **异步请求**

-  **一次请求**

### 2.5 网络请求

	Netty 发送信息