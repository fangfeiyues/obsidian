
## 为什么是Dubbo？

 - **Dubbo vs  HTTP**
	1.  稳定性。http的支持是集中式的，rpc的支持是分布式的。经验上，rpc的稳定性更高一些。
	2.  稳定性。rpc在协议上实现了诸多功能如限流、熔断、降级等，这些功能在稳定性治理上是刚需。
	3.  成本。http的附加支持是有成本的，http的支持附加成本比rpc高 （rpc不需要附加成本，这些附加成本是指：LB的成本，申请域名、审批、创建http vip配置等的成本）


![[image-Dubbo-1 调用过程之调用者-20240420205050456.png|450]]

https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/architecture/code-architecture/

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

	消费者初始化refer引用中，会到注册中心创建调用者节点，并订阅提供者，再根据监听的生产者url开始封装invoker

#### Zookeeper

	![[image-Dubbo-1 调用过程之调用者-20240420225304815.png|450]]

-  **流程**
	1、服务提供者启动时:  向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址。
	2、服务消费者启动时:  订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址，并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址（用于监控）
	3、 监控中心启动时: 订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者 URL 地址


#### Redis





#### Nacos





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

-  **1、FailoverClusterInvoker**  默认执行策略，快速失败集群
-  **2、FailfastClusterInvoker** 
-  **3、BroadcastClusterInvoker** 
-  **4、FailbackClusterInvoker** 
-  **5、FailfastClusterInvoker** 
-  **6、FailfastClusterInvoker** 

## 2、生成代理
常用的动态代理大概有 ASM、CGLIB、JAVAASSIST等
###  javassist

### jdkProxy

只能代理接口，原因是生成的`$Proxy0`代理类要继承接口信息，做反射。其主要流程是
-  `ProxyGenerator` 自动生成 Proxy 的具体类 `$Proxy0`
-  由static初始化块初始化接口方法：2个IUserService接口中的方法，3个Object中的接口方法
-  由构造函数注入InvocationHandler
-  执行的时候，通过`ProxyGenerator`创建的 Proxy，调用 InvocationHandler 的invoke方法，执行我们自定义的invoke方法

```java
// 代理类
public class UserLogProxy {

    /**
     * proxy target
     */
    private IUserService target;

    /**
     * init.
     *
     * @param target target
     */
    public UserLogProxy(UserServiceImpl target) {
        super();
        this.target = target;
    }

    /**
     * get proxy.
     *
     * @return proxy target
     */
    public IUserService getLoggingProxy() {
        IUserService proxy;
        ClassLoader loader = target.getClass().getClassLoader();
        Class[] interfaces = new Class[]{IUserService.class};
        InvocationHandler h = new InvocationHandler() {
            /**
             * proxy: 代理对象。 一般不使用该对象 method: 正在被调用的方法 args: 调用方法传入的参数
             */
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                String methodName = method.getName();
                // log - before method
                System.out.println("[before] execute method: " + methodName);

                // call method
                Object result = null;
                try {
                    // 前置通知
                    result = method.invoke(target, args);
                    // 返回通知, 可以访问到方法的返回值
                } catch (NullPointerException e) {
                    e.printStackTrace();
                    // 异常通知, 可以访问到方法出现的异常
                }
                // 后置通知. 因为方法可以能会出异常, 所以访问不到方法的返回值

                // log - after method
                System.out.println("[after] execute method: " + methodName + ", return value: " + result);
                return result;
            }
        };
        /**
         * loader: 代理对象使用的类加载器.
         * interfaces: 指定代理对象的类型. 即代理代理对象中可以有哪些方法.
         * h: 当具体调用代理对象的方法时, 应该如何进行响应, 实际上就是调用 InvocationHandler 的 invoke 方法
         * proxy：这里是继承接口的代理类
         */
        proxy = (IUserService) Proxy.newProxyInstance(loader, interfaces, h);
        return proxy;
    }

}

// 所有类和方法都是final类型的
public final class $Proxy0 extends Proxy implements IUserService {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;
    private static Method m4;

    // 构造函数注入 InvocationHandler
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final List findUserList() throws  {
        try {
            return (List)super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void addUser() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            // 初始化 methods, 2个IUserService接口中的方法，3个Object中的接口
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("tech.pdai.springframework.service.IUserService").getMethod("findUserList");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m4 = Class.forName("tech.pdai.springframework.service.IUserService").getMethod("addUser");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```



### cglib

![[image-Dubbo-1 调用过程之调用者-20240423204123623.png|500]]

- 最底层是字节码
- asm 是操作字节码的工具
- cglib 基于ASM字节码工具操作字节码（即动态生成代理，对方法进行增强）
- springAOP 基于 cglib 进行封装，实现 cglib 方式的动态代理

![[image-Dubbo-1 调用过程之调用者-20240423204456809.png|650]]

```java
public class UserLogProxy implements MethodInterceptor {

    /**
     * 业务类对象，供代理方法中进行真正的业务方法调用
     */
    private Object target;

    public Object getUserLogProxy(Object target) {
        //给业务对象赋值
        this.target = target;
        //创建加强器，用来创建动态代理类
        Enhancer enhancer = new Enhancer();
        //为加强器指定要代理的业务类（即：为下面生成的代理类指定父类）
        enhancer.setSuperclass(this.target.getClass());
        //设置回调：对于代理类上所有方法的调用，都会调用CallBack，而Callback则需要实现intercept()方法进行拦
        enhancer.setCallback(this);
        // 创建动态代理类对象并返回
        return enhancer.create();
    }

    // 实现回调方法
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // log - before method
        System.out.println("[before] execute method: " + method.getName());

        // call method
        Object result = proxy.invokeSuper(obj, args);

        // log - after method
        System.out.println("[after] execute method: " + method.getName() + ", return value: " + result);
        return null;
    }
}


public class ProxyDemo {

    /**
     * main interface.
     *
     * @param args args
     */
    public static void main(String[] args) {
        // proxy
        UserServiceImpl userService = (UserServiceImpl) new UserLogProxy().getUserLogProxy(new UserServiceImpl());

        // call methods
        userService.findUserList();
        userService.addUser();
    }
}



```