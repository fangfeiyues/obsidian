
## 1、每天100w次登陆请求，8G内存该如何设置jvm参数

### 1、规划容量

-  **计算内存空间**

```text
计算系统每秒钟创建的对象会占用多大的内存，然后计算集群的每个系统每秒的内存占用用空间
（ 比如每台机器每秒会处理30个请求，一个请求对象20个字段500字节，加上网络IO缓存一般扩大20-50倍，即每秒 30 * 500 * 35倍 = 600kb ）
```

- **新生代机器配置**

```text
估算新生代的空间，比较不同新生代大小之下，多久触发一次MinorGC
（ 比如2C4G机器，分配2G堆内存，新生代就3/8 * 8/10 = 525m，那么平均800秒触发一次MinorGC ）
```

- **老年代机器配置**

```text
为了避免频繁GC，就可以重新估算需要多少机器配置，部署多少台机器，给JVM多大内存空间，新生代多大空间
```

推算出系统每秒创建多少对象，1s以后多少成为垃圾，运行多久新生代会触发一次GC，频率多高。


### 2、选择垃圾回收器

-  **吞吐量 vs 低延迟**

```text
  吞吐量 = CPU在用户应用程序运行的时间 / （CPU在用户应用程序运行的时间 + CPU垃圾回收的时间）
  响应时间 = 平均每次的GC的耗时
```

-  **垃圾回收器**

```
‐XX:+UseParNewGC - ParNew新生代垃圾回收器组合
‐XX:+UseConcMarkSweepGC - CMS多线程并行，加快MinorGC速度

‐XX:+UseG1GC - G1垃圾收集器
```

[[JVM-04 对象内存回收]]

总之，业务系统延迟敏感的推荐CMS、大内存要求吞吐高的采用G1

### 3、设置新生代分区比例 ☑️

JVM最重要最核心的参数是去评估内存和分配

-  **1、指定堆内存的大小**

```text
-Xms:0.5的OS  - 初始堆大小
-Xmx:0.5的OS  - 最大堆大小

后台Java服务中一般都指定为系统内存的1/2，过大会佔用服务器的系统资源，过小则无法发挥JVM的最佳性能
```

-  **2、指定新生代的大小**

```text
-Xmn: 3/4 - 新生代大小

这个参数非常关键，灵活度很大，虽然sun官方推荐为3/8大小，但是要根据业务场景来定，
  1、针对于无状态或者轻状态服务（现在最常见的业务系统如Web应用）来说，一般新生代甚至可以给到堆内存的3/4
  2、对于有状态服务（常见如IM服务、网关接入层等系统）新生代可以按照默认比例1/3来设置，服务有状态，则意味著会有更多的本地缓存和会话状态信息常驻内存，应为要给老年代设置更大的空间来存放这些对象
```

- **3、指定Eden:survivor比例**

```text
-XX:SurvivorRatio=8  - Eden: s1 : s2 = 8:1:1

survivor区只有1G的10%左右，就是几十到100M。如果每次minor GC垃圾回收过后进入survivor对象很多，并且survivor对象大小很快超过Survivor的50%，那么会触发动态年龄判定规则，让部分对象进入老年代.而一个GC过程中，可能部分WEB请求未处理完毕,  几十兆对象，进入survivor的概率，是非常大的，甚至是一定会发生的

如何解决这个问题呢？
为了让对象尽可能的在新生代的eden区和survivor区, 尽可能的让survivor区内存多一点,达到200兆左右
```

	![[image-JVM-05 实战经验-20240426110338688.png|600]]
	
	![[image-JVM-05 实战经验-20240426004232615.png|500]]


### 4、设置栈内存

```text
-Xss:1024kb

栈内存大小，设置单个线程栈大小，默认值和JDK版本、系统有关，一般默认512~1024kb。
一个后台服务如果常驻线程有几百个，那麽栈内存这边也会佔用了几百M的大小。
```


### 5、设置新老年代年龄

```text
-XX:MaxTenuringThreshold=5 - 经过5次minor gc会进入老年代

假设一次minor gc要间隔20-30s，并且，大多数对象一般在几秒内就会变为垃圾
如果对象这么长时间都没被回收，比如2分钟没有回收，可以认为这些对象是会存活的比较长的对象，从而移动到老年代，而不是继续一直占用survivor区空间。
所以，可以将默认的15岁改小一点，比如改为5，那么意味着对象要经过5次minor gc才会进入老年代，整个时间也有一两分钟了（5 * 30s= 150s），和几秒的时间相比，对象已经存活了足够长时间了

```


### 6、设置新老年代大小

```text
-XX:PretenureSizeThreshold=1m - 大于1m的大对象直接在老年代生成

对于多大的对象直接进入老年代(参数)，一般可以结合自己系统看下有没有什么大对象生成，预估下大对象的大小，一般来说设置为1M就差不多了，很少有超过1M的大对象
```


### 7、设置老年代优化参数 ☑️

-  **CMS垃圾选择器**

```text
JDK8默认的垃圾回收器是  -XX:+UseParallelGC(年轻代) 和 -XX:+UseParallelOldGC(老年代)
  1.如果超过4G，内存较大的还是建议使用G1
  2.如果4G以内，又是主打“低延时” 的业务系统，可以使用组合 ParNew + CMS (-XX:+UseParNewGC -XX:+UseConcMarkSweepGC)

a) 新生代的采用ParNew回收器，工作流程就是经典复制算法，在3块区中进行流转回收，只不过采用多线程并行的方式加快了MinorGC速度。
b) 老生代的采用CMS，再去优化老年代参数：比如老年代默认在标记清除以后会做整理，还可以在CMS的增加GC频次还是增加GC时长上做些取舍，

```

-  **CMS GC时间**

```
-XX:CMSInitiatingOccupancyFraction=70
设定CMS在对内存占用率达到70%的时候开始GC，因为CMS会有浮动垃圾,所以一般都较早启动GC 
```

### 8、设置OOM时的内存dump文件和GC日志

```
-XX:+HeapDumpOnOutOfMemoryError - 在Out Of Memory，JVM快死掉的时候，输出Heap Dump到指定文件
-XX:HeapDumpPath=${LOGDIR}

-Xloggc:/dev/xxx/gc.log  - GC日志
-XX:+PrintGCDateStamps   
-XX:+PrintGCDetails
```


### 总结

-  **基于 ParNew + CMS 配置**

```text
‐Xms3072M - 最小最大内存3g，设置一致是防止内存抖动   
‐Xmx3072M 
‐Xmn2048M - 年轻代大小2g，eden与survivor的比例为8:1:1，也就是1.6g:0.2g:0.2g  
‐Xss1M - 线程栈1m  
‐XX:SurvivorRatio=8    
‐XX:MaxTenuringThreshold=5 - 年龄为5进入老年代
‐XX:PretenureSizeThreshold=1M - 于1m的大对象直接在老年代生成
‐XX:+UseParNewGC - 使用ParNew+cms垃圾回收器组合
‐XX:+UseConcMarkSweepGC - 采用多线程并行的方式加快了MinorGC速度
‐XX:CMSInitiatingOccupancyFraction=70 - 年代中对象达到这个比例后触发fullgc
‐XX:+UseCMSInitiatingOccupancyOnly - 老年代中对象达到这个比例后触发fullgc
‐XX:+AlwaysPreTouch - 强制操作系统把内存真正分配给IVM，而不是用时才分配
-XX:+HeapDumpOnOutOfMemoryError - OOM时候的内存dump文件
-XX:+PrintGCDateStamps - GC日志输出
-XX:+PrintGCDetails
```


- **基于 G1 配置**

```
-Xms8g  
-Xmx8g  
-Xss1m  
-XX:+UseG1GC  
-XX:MaxGCPauseMillis=150  
-XX:InitiatingHeapOccupancyPercent=40  
-XX:+HeapDumpOnOutOfMemoryError  
-verbose:gc  
-XX:+PrintGCDetails  
-XX:+PrintGCDateStamps  
-XX:+PrintGCTimeStamps  
-Xloggc:gc.log


G1收集器自身已经有一套预测和调整机制了，因此我们首先的选择是相信它，即调整 -XX:MaxGCPauseMillis=N 最大暂停时间参数，这也符合G1的目的——让GC调优尽量简单！同时也不要自己显式设置新生代的大小（用-Xmn或-XX:NewRatio参数），如果人为干预新生代的大小，会导致目标时间这个参数失效
```


## 2、排查一次线上OOM



