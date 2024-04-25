 
 Dubbo框架中各层之间的引用注入都是依赖了 SPI 强大的能力，抛弃了 Spring IOC 的注入和 Spring AOP 的切面

## Java SPI

-  **定义**
	SPI 全名为 Service Provider Interface，其思想是解耦，通过可插拔的方式把装配的控制权移到程序外

-  **约定**
	服务提供者提供接口实现 xxxImpl 后，在 jar 包的 `META-INF/services/` 目录创建一个以 服务接口命名 的文件，该文件内就是具体的实现类，然后通过`ServiceLoader`类对所有的实现进行加载

```java
/** Java SPI 的实现原理基于 类加载机制 和 反射机制，ServiceLoader.load(Class<T> service) 方法加载
		1、会检查 META-INF/services 目录下是否存在以接口全限定名命名的文件，
		2、通过 Class.forName() 方法加载对应的类 
**/
   public static void main(String[] args) {
        ServiceLoader<ProgrammingLanguageService> serviceLoader =  
					         ServiceLoader.load(ProgrammingLanguageService.class);
        Iterator<ProgrammingLanguageService> iterator = serviceLoader.iterator();
        while (iterator.hasNext()) {
            ProgrammingLanguageService service = iterator.next();
            service.study();
        }
    }
```



## Dubbo SPI

Dubbo SPI 在 Java SPI 基础上解决什么问题？
### vs Java SPI

#### 1、可按需加载扩展点

Java SPI 会通过 ServiceLoader 一次性加载所有扩展点，而 Dubbo SPI 通过注解 @SPI 实现了按需加载

```java
// PrintService接口的Dubbo SPI改造
// ① 在目录META-INF/dubbo/internal下建立配置文com.test.spi.Printservice,文件内容如下
// impl=com.test.spi.PrintServiceImpl 

// ② 为接口类添加SPI注解，设置默认实现为impl
@SPI("impl")   
public interface Printservice {
    void printlnfo();
)

// ③实现类不变
public class PrintServicelmpl implements Printservice ( 
Override
public void printlnfo() (
      System.out println("hello world");
} 
}

// ④调用Dubbo SPI
public static void main(String[] args) ( 
    //通过 ExtensionLoader 获取接口PrintService.class 的默认实现
    PrintService printservice = ExtensionLoader
    .getExtensionLoader(PrintService.class).getDefaultExtension();
    //此处会输出 PrintServicelmpl 打印的 hello world
    printService.printInfo();
}
```

#### 2、IOC 和 AOP机制

-  **扩展点的自动扩展特性**

	如A中注入了B，那么在实现A的时候也会扩展实现B，这个和spring的IOC原理类似，所以称它为dubbo spi中的 IOC，也称为扩展点的自动扩展特性


-  **扩展点的自动包装特效**

	如上，在扩展B时如何决定要注入依赖扩展点的哪个实现？ 在设计模式中有一个 装饰器模式，它通常用来在不改变原有对象的行为方法的同时，用来对原有对象进行方法增强。在dubbo中，实例化一个扩展点实现的同时，也会判断此对象有没有作为一个包装类对象中构造方法的对象，如果是，也会实例化该 `wrapper`包装
	
	在上面这个创建扩展点的方法中，扩展点自动装配以后，会继续对包装类对象进行注入。这个和aop原理一样，称为dubbo spi中的aop，也叫**扩展点的自动包装**

	
	```java
	private T createExtension(String name){
	             //这里省略了部分代码
	             .......
	            /**
	             * 向扩展类注入其依赖的扩展点属性，这里是体现了扩展点自动装配的特性
	             */
	            injectExtension(instance);
	            /**
	             * （ cachedWrapperClasses 对象在执行 getExtensionClasses 方法时已经赋值 ）
	             * 扩展点自动包装特性，ExtensionLoader在加载扩展时，如果发现这个扩展类包含其他扩展点作为构造函数的参数，则这个扩展类会被认为是wrapper类，那么这个wrapper类也会被实例化并且注入扩展点属性
	             */
	            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
	            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
	                for (Class<?> wrapperClass : wrapperClasses) {
	                    //  wrapper对象的实例化（injectExtension():向扩展类注入其依赖的属性,如扩展类A又依赖了扩展类B，那么就向A中注入扩展类B）
	                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
	                }
	            }
	          //  .......
	}
```

#### 3、异常处理

	Java SPI 加载失败，可能会因为各种原因导致异常信息被“吞掉”，导致开发人员问题追踪比较困难。
	Dubbo SPI 在扩展加载失败的时候会先抛出真实异常并打印日志，扩展点在被动加载的时候，即使有部分扩展加载失败也不会影响其他扩展点和整个框架的使用


### 注解

#### 2.1 控制与包装：SPI

- **例子**

	```java
	/** PrintService接口的Dubbo SPI改造
	① 在目录META-INF/dubbo/internal下建立配置文com.test.spi.Printservice,文件内容如下
	impl=com.test.spi.PrintServiceImpl 
	
	② 为接口类添加SPI注解，设置默认实现为impl
	**/
	@SPI("impl")   
	public interface Printservice {
	    void printlnfo();
	)
	
	// ③ 实现类不变
	public class PrintServicelmpl implements Printservice ( 
	Override
	public void printlnfo() (
	      System.out println("hello world");
	} 
	}
	
	// ④ 调用Dubbo SPI
	public static void main(String[] args) ( 
	    //通过 ExtensionLoader 获取接口PrintService.class 的默认实现
	    PrintService printservice = ExtensionLoader
	    .getExtensionLoader(PrintService.class).getDefaultExtension();
	    //此处会输出 PrintServicelmpl 打印的 hello world
	    printService.printInfo();
	}
	```


- **原理**

	Dubbo SPI 解决的是一个怎么加载自定义类的问题


#### 2.2 自适应扩展： Adaptive

-  **例子**
	
	```java
	@SPI("javassist")  
	public interface ProxyFactory {  
	  
	    /**  
	     * create proxy.     *     * @param invoker  
	     * @return proxy  
	     */    @Adaptive({Constants.PROXY_KEY})  
	    <T> T getProxy(Invoker<T> invoker) throws RpcException;  
	  
	    /**  
	     * create proxy.     *     * @param invoker  
	     * @return proxy  
	     */    @Adaptive({Constants.PROXY_KEY})  
	    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;  
	  
	    /**  
	     * create invoker.     *     * @param <T>  
	     * @param proxy  
	     * @param type  
	     * @param url  
	     * @return invoker  
	     */    @Adaptive({Constants.PROXY_KEY})  
	    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;  
	  
	}
	
	public class ProxyFactory$Adaptive implements org.apache.dubbo.rpc.ProxyFactory {  
	
	    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0, boolean arg1) {  
	       // 取URL中 proxy = 'xxx' 参数，如果不存在用默认的 ‘javassist’
	        String extName = url.getParameter("proxy", "javassist");  
	        // extName -> ProxyFactory 如 JavassistProxyFactory
	        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory)ExtensionLoader  .getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);  
	        return extension.getProxy(arg0, arg1);  
	    }  
	  
	    public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1,  org.apache.dubbo.common.URL arg2)   {  
	        String extName = url.getParameter("proxy", "javassist");  
	        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory)ExtensionLoader  .getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);  
	        return extension.getInvoker(arg0, arg1, arg2);  
	    }  
	}
	
	```
	

-  **使用**

	Adaptive 解决的是在类似装饰器模式场景下，接口注入中有其他接口的时候，怎么识别其实现类的问题


-  **实现**

	动态生成的 xxx$Adaptive 类可以得知，每个默认实现都会从`URL`中提取`Adaptive参数值`，并以此为依据动态加载扩展点，调用 `getExtension(extName)`。  
	1.  优先通过 `©Adaptive`注解传入的值去查找扩展实现类
	2.  如果没找，到则通过`@SPI`注解中的值去查找
	3.  如果`@SPI`注解中没有默认值，则把类名转化为key，再去查找




## 总结

结合上面SPI注解 IOC和AOP 以及 Adaptive 的方法路由，可以推演扩展一种代理方式的实现大概如下
-  **初始化配置**
	1.  实现接口 `ProxyFactory（SPI = "javassist") ，Adaptive = "proxy" ）`自定义实现类 AsmProxyFactory
	2.  文件 `resources/META-INF/dubbo/org.apache.dubbo.rpc.ProxyFactory` 内容  `asm = org.apache.dubbo.samples.extensibility.proxy.AsmProxyFactory`
	3.  配置 `<dubbo:protocol proxy="asm" />`
- **请求过程**
	1.  Dubbo框架触发扩展点加载。-- 通过`ExtensionLoader`加载`ProxyFactory`的实现
	2.  Dubbo调用`ProxyFactory`的`getProxy`方法来创建代理对象。-- 通过 SPI 机制
	3.  代理对象在 URL 中选择参数动态选择一个实现类。-- 通过Adaptive参数实现


所以总结来说，Dubbo SPI机制核心就是为了给各个接口提供动态的自定义扩展能力，其实现方式是通过 SPI注解 和 Adaptive注解 生成代理对象然后根据URL传参路由到具体的实现类。

[dubbo系列-扩展点机制-Dubbo SPI](https://www.jianshu.com/p/317ea9559ee2)