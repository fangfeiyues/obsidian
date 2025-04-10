## 1、MySQL

-  **数据库为什么会忽然抖动**

		1. 慢查询：索引、数据量太大
			1. 刷脏页：把内存脏页刷回磁盘的过程，innodb_max_dirty_pages_pct 可控制比例页
			2. redo log满了：系统会停止更新，危矣！
			3. 内存不足了：
			4. 空闲了
		2. 硬件和网络导致的
		3. MySQL后台任务导致资源不足

-  **MySQL vs HABSE vs Redis 存储差异**

-  **怎么做分库分表**


## 2、分布式

-  **04.10｜怎么理解ACP**

-  **04.10｜MQ异常重试 & 超时Ack **

-  **04.10｜分布式事务解决方案**

	[[Distributed-3 分布式事务]]
	
		1. 2PC：准备 -> 提交/回滚
			1. 事务协调者挂机，
			2. 单节点超时
			3. 事务协调者 & 节点 状态不一致
		2. 3PC：准备 ->  提交/回滚

-  **04.10｜分布式锁实现**

	[[Distributed-2 分布式锁]]
	
		1. MySQL vs Redis vs ZK
			MySQL <key,value> 存在单点问题、自动失效、不能阻塞、非可重入
			Redis 锁不能续期、未设置有效期而死锁问题、非可重入、单点
			ZK ZAB协议强一致性、到期自动删除节点，但要新增&删除节点
		
		2. Redis vs Redission
			Redission 看门狗续期、支持Fair
			
		3. RedLock -> RedissionRedLock -> 被废弃
			解决Master单点故障后被重复加锁的问题，引入多个节点，
			但也存在脑裂（分区导致）而被多次加锁、网络分区和节点故障、操作复杂更高
			
		4. watchdog 解决业务执行续期问题，延长有效期1/3时间，会一直续期到手动结束，用的时间轮算法（槽+轮）
		    
		5. Bond
			1. Etcd -> Redis，也就是强一致性到最终一致性，即允许两机器不一致性换取性能
			2. 阻塞性：锁
			3. 可重入性
			4. 异常场景：
				1. 加锁异常：不会立即失败，而是回查结果防止因超时导致
				2. 业务执行 > 锁过期时间：看门狗方案
				3. A unlock B：解锁时间 > T + leaseTime -150ms 则不执行解锁操作

	 ![[Pasted image 20250410145647.png|400]]
	 
	 
	![[Pasted image 20250410115514.png|400]]

## 3、JVM
 
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
