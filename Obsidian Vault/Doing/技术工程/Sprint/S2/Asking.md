
### 1、MySQL

**数据库为什么会忽然抖动？**

	刷盘期间

**MySQL vs HABSE vs Redis 存储差异？**


### 2、分布式


### 3、JVM
 
  **CPU飙高**

	1. cpu利用率 = cpu非空闲时间/cpu空闲时间 * 100%｜vmstat 2 2 
              每两秒收集一次信息（包括memory,swap,io,system,cpu..）
	2. top -> top -Hp 1893 -- 查看进程到线程的资源占用情况（包括cpu占用，memory占用）
	3. jstack -l 12345 -- 查看线程堆栈信息 nid 16进制，如 RUNNABLE 具体代码行、deadlock（死锁
	4.  arthas: thread -n 3  -- 查看最忙的3个线程


**OOM频繁**

	1. 自动dump文件：-xx:HeapDumpOnOutOfMemoryError 
	2. 手动dump文件：jmap -dump:live,format=b,file=xxxheap.bin <pid>
	3. MAT(Memory Analysic Tool) 内存分析
         1. 死锁
         2. 大对象


**FullGC频繁**

    1. dump & 分析内存情况
    2. 大对象是否存在？
        1. LIST<object>是否频繁添加对象没移除
        2. 分页查询没做好分页，导致一次查询数量过多，如传参没带id等参数导致数据全部搂出来
