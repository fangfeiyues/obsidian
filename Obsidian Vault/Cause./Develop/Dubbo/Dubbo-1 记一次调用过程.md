
 - **Dubbo vs  HTTP**
	1.  稳定性。http的支持是集中式的，rpc的支持是分布式的。经验上，rpc的稳定性更高一些。
	2.  稳定性。rpc在协议上实现了诸多功能如限流、熔断、降级等，这些功能在稳定性治理上是刚需。
	3.  成本。http的附加支持是有成本的，http的支持附加成本比rpc高 （rpc不需要附加成本，这些附加成本是指：LB的成本，申请域名、审批、创建http vip配置等的成本）


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

## 生成Invoker

### 注册


### 集群


## 路由



## 2、控制

### 失败避让


### 负载均衡


### 过滤拦截


## 3、请求

### 基本设置

#### 超时

#### 异步



### 网络调用



## 4、接收



## 5、调用



## 6、返回





![[分层设计.png]]