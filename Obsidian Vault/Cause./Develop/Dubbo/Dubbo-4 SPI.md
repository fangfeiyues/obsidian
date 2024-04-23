 
 Dubbo框架中各层之间的引用注入都是依赖了 SPI 强大的能力，抛弃了 Spring IOC 的注入和 Spring AOP 的切面

# Java SPI

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

# Dubbo SPI

Dubbo SPI 在 Java SPI 基础上解决什么问题？
## 能力扩展

### 1、可按需加载扩展点

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

### 2、可支持AOP

简单来说就是如果扩展点A,B之间发生了互相依赖，Dubbo SPI 支持注入 如 ProtocolFilterWrapper、ProtocolListenerWrapper（ dubbo是通过识别 xxxWrapper 名称来识别是否是包装类的）就可以把这两个方法包装增强核心的 DubboProtoco l 类

```java
private T createExtension(String name){
             //这里省略了部分代码
             .......
            /**
             * 向扩展类注入其依赖的扩展点属性，这里是体现了扩展点自动装配的特性
             */
            injectExtension(instance);
            /**
             * 这里的cachedWrapperClasses对象 在执行getExtensionClasses方法时已经赋值
             * 扩展点自动包装特性，ExtensionLoader在加载扩展时，如果发现这个扩展类包含其他扩展点作为构造函数的参数，
             * 则这个扩展类会被认为是wrapper类，比如 ProtocolFilterWrapper,就是在构造函数中注入了 Protocol类型的扩展点
             * 那么这个wrapper类也会被实例化并且注入扩展点属性
             */
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    // *** wrapper对象的实例化（injectExtension():向扩展类注入其依赖的属性,如扩展类A又依赖了扩展类B，那么就向A中注入扩展类B）
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }

          //  .......

}
```


### 3、可异常处理

Java SPI 加载失败，可能会因为各种原因导致异常信息被“吞掉”，导致开发人员问题追踪比较困难。
Dubbo SPI 在扩展加载失败的时候会先抛出真实异常并打印日志，扩展点在被动加载的时候，即使有部分扩展加载失败也不会影响其他扩展点和整个框架的使用


## 源码

### 2.1 SPI可配置源码

核心是通过注解 `@SPI` 加载配置的实现类并初始化，然后对该类进行 注入属性 与 IOC包装

```java
/**
 * 1、加载定义文件中的各个子类，然后将目标name对应的子类返回后进行实例化
 * 2、通过目标子类的set方法为其注入其所依赖的bean，这里既可以通过SPI，也可以通过Spring的BeanFactory获取所依赖的bean，injectExtension(instance)。
 * 3、获取定义文件中定义的wrapper对象，然后使用该wrapper对象封装目标对象，并且还会调用其set方法为wrapper对象注入其所依赖的属性
 * @param name
 * @return
 */
@SuppressWarnings("unchecked")
private T createExtension(String name) {

    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        /**
         * 向扩展类注入其依赖的扩展点属性，这里是体现了扩展点自动装配的特性
         */
        injectExtension(instance);
        /**
         * 这里的cachedWrapperClasses对象 在执行getExtensionClasses方法时已经赋值
         * 扩展点自动包装特性，ExtensionLoader在加载扩展时，如果发现这个扩展类包含其他扩展点作为构造函数的参数，
         * 则这个扩展类会被认为是wrapper类，比如 ProtocolFilterWrapper,就是在构造函数中注入了 Protocol类型的扩展点
         * 那么这个wrapper类也会被实例化并且注入扩展点属性
         */
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                //wrapper对象的实例化（injectExtension():向扩展类注入其依赖的属性,如扩展类A又依赖了扩展类B，那么就向A中注入扩展类B）
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

1、getExtensionClasses 方法比较长，在此不一一列举，主逻辑就是获取到扩展点的所有实现类，中间会加入各种缓存提高性能，需要注意的是这里获取的只是扩展点的实现类，并没有实例化，这也印证了我们上面所说的按照需要获取扩展实现类，并且只是加载配置文件中的类， 并分成不同的种类缓存在内存中，而不会立即全部初始化，在性能上有更好的表现。

2、injectExtension 是扩展点自动扩展的特性具体实现，基本原理：

方法总体实现了类似Spring的IoC机制，其实现原理比较简单：首先通

过反射获取类的所有方法，然后遍历以字符串set开头的方法，得到set方法的参数类型，再通过ExtensionFactory寻找参数类型相同的扩展类实例，如果找到，就设值进去

### 2.2 Adaptive自适应源码

```
 ExtensionLoader 要注入依赖扩展点时，如何决定要注入依赖扩展点的哪个实现？这就需要提到扩展点的自适应注解@Adaptive。

 Adaptive就在spi所有扩展类中可选折自定义的类，其获取方式就是URL.getParam("xxx") 如果没有就取adaptive默认值。如下 ExtensionLoader 根据接口包生成的 Protocol$Adaptive 类
```

```java
String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
Protocol extension = (Protocol) ExtensionLoader
                     .getExtensionLoader(Protocol.class).getExtension(extName);
```

Dubbo Adaptive 的实现机制主要分为如下三个步骤：

- 加载标注有`@Adaptive`注解的接口，如果不存在，则不支持Adaptive机制；
- 为目标接口按照一定的模板生成子类代码，并且编译生成的代码，然后通过反射生成该类的对象；
- 结合 「生成的对象实例」 + 「传入的URL对象」，获取指定key的配置，然后加载该key对应的类对象，最终将调用委托给该类对象进行

为扩展点接口自动生成实现类字符串，实现类主要包含以下逻辑：为接口中每个有Adaptive注解的方法生成默认实现(没有注解的方法则生成空实现)，每个默认实现都会从URL中提取Adaptive参数值，并以此为依据动态加载扩展点。然后，框架会使用不同的编译器，把实现类字符串编译为自适应类并返回



https://www.jianshu.com/p/317ea9559ee2
[dubbo系列-扩展点机制-Dubbo SPI](https://www.jianshu.com/p/317ea9559ee2)