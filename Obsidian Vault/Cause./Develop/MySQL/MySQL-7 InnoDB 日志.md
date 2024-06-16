
## 1、redo log（重做日志）

`WAL技术` 全称 Write-Ahead Logging，先写日志再写磁盘

-  **作用**

	MySQL用于奔溃恢复和数据持久性的一种机制，在事务进行时，MySQL会将事务做了什么改动到redo log，当系统崩溃或者发生异常时，MySQL会利用Redo log记录信息进行恢复
	
	防止奔溃的话，为什么不直接保存记录到磁盘，而要在redo log上走一次？-- redo log是保证的buffer可用性

### 日志结构

-  **Mini-Transaction**

	保证一条插入语句因页面分裂而造成多个redo日志保存时候的原子性，InnoDB允许一组同时提交。

- **Log Sequence Number** 逻辑日志序列号

	LSN单调递增，每次写入长度为 redo log 的 length。用于组提交的一组LSN

-  **checkpoint** 

	 `write pos` 记录当前位置，一边写一边后移
	 `check point` 擦除当前位置
	 write pos 和 check point 之间是空着部分还可以继续记录，如果write 追上 check 表示内存满了，则需要停下来清楚数据
	 
	![[MySQL-6 InnoDB日志-redo log.png|400]]

### 写入机制

-  1、配置写入 **innodb_flush_log_at_trx_commit** 
	1.  0，每次提交事务都把 redo log 留在 redo log buffer
	2.  1，每次提交事务都把 redo log 持久化到磁盘
	3.  2，每次提交事务都把 redo log 留在 page buffer

-  2、后台线程每隔1s，会把 redo log buffer 中的日志写到 page cache，然后调用 fsync 持久化

-  3、其他场景的持久化
  
	-  1、redo log buffer 占用空间到一半，主动写到page cache
	-  2、并行事务的时候会顺带提交
	  
	![[MySQL-6 InnoDB 日志-1.png|500]]

## 2、binlog（归档日志） 

### 格式

-  **statement**

	记录SQL原文，但在 `RC + statement` 条件下会导致不一致问题，用的比较少
	如 insert 和 delete 执行，但在`RR`级别下数据更新时候会增加 GAP锁 和 next-key锁，这样就session1加锁后卡住session2，然后先提交
	
	![[image-MySQL-7 InnoDB 日志-20240603234602666.png|600]]

-  **row**

	记录数据更改行的细节，意味着日志中会详细列出发生变更的内容。就不会导致主从不一致问题，但也会记录更多的内容，存储需要的空间更大

- **mixed**

	MySQL会根据SQL情况，自动在 row 和 statement 互相切换

### 结构

### 写入机制

两个操作
-  **write** ：把日志写入文件系统的page cache，并没有持久化到磁盘
-  **fsync**：持久到磁盘


两个操作时机都是由 `sync_binlog` 控制
1.  `sync_binlog = 0`，每次都只 `write` 不 `fsync`
2.  `sync_binlog = 1`，每次都会 `fsync`
3.  `sync_binlog = N`，每次都 `write` 等 N 次事务后`fsync`

![[MySQL-6 InnoDB 日志.png|600]]

-   **Redo Log vs BinLog**

	1、redo log 是 InnoDB 特有的，binlog 是MySQL 的 Server 层实现的，所有引擎都可以使用
	2、redo log 是 物理日志，记录是 “在某个数据页上做了什么修改”；
		binlog是逻辑日志，记录是这个语句的原始逻辑
	3、redo log 是循环写，binlog 是追加写


	![[MySQL-6 InnoDB 日志-2.png|400]]


Q&A 
-  binlog 为什么不能用于奔溃恢复？


## 3、undo log

为了实现事务的原子性，InnoDB存储引擎在实际进行增删改一条记录时，都需要先把对应的undo日志记录下来。一般每对一条记录做一次改动就对应着一条undo日志。
有两个作用：一是提供回滚；二是实现MVCC功能。