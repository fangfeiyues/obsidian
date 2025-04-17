## 1、锁类型

### 1.1 悲观锁

-  **案例**

```sql
-- 0.开始事务
begin; 
-- 1.查询出商品库存信息, 这里用for update当前读防止读到MVCC快照下其他view之后的操作
select quantity from items where id = 1 for update;
-- 2.修改商品库存为2
update items set quantity=2 where id = 1;
-- 3.提交事务
commit;
```

### 1.2 乐观锁

```sql
-- 查询出商品库存信息，quantity = 3
select quantity from items where id=1
-- 修改商品库存为2，同时避免ABA问题
update items set quantity=2 where id=1 and quantity = 3 and version = 2;;
```


### 1.3 一致性读

-  **原理**

	事务利用MVCC进行的读取操作称之为一致性读，或者一致性无锁读，有的地方也称之为快照读。
	普通的`SELECT`语句（plain SELECT）在`READ COMMITTED`、`REPEATABLE READ`隔离级别下都算是一致性读

-  **SQL**

	`SELECT * FROM t1 INNER JOIN t2 ON t1.col1 = t2.col2`

### 1.4 一致性锁定读

-  **SQL**

	显示地对数据库读操作进行加锁以保证数据逻辑的一致性，如：
	`SELECT ... FOR UPDATE`
	`SELECT ... LOCK IN SHARE MODE`

### 1.5 自增长与锁

-  **原理**

	 `AUTO_INCRMENT` 自增列锁的两种方案：
	1. AUTO_INC锁，插入的行加上锁，语句执行结束释放，那么加锁的过程中其他的插入都要阻塞
	2. 轻量级的锁，插入的行操作完成，就锁释放掉，并不需要等到整个插入语句执行完才释放锁

```
innodb_autoinc_lock_mode=0 : AUTO-INC锁
innodb_autoinc_lock_mode=1 : 混着，插入记录数量确定时采用轻量级锁，不确定时使用AUTO-INC锁
innodb_autoinc_lock_mode=2 : 
```


-  **SQL**

	insert 主键自增

### 1.6 行锁

Record Lock：记录锁

Gap Lock：间隙锁会锁记录前的两个之间的间隙如update a = 8则在(3,8)之间加锁

Next-Key Lock：gap-lock + 行锁 即 (3,8]

**怎么减少行锁对性能的影响？**

见死锁。

## 2、锁作用

### 2.1 不可重复读

通过 `Next-Key Lock` 算法来避免不可重复读的问题

### 2.2 幻读


## 3、死锁

### 3.1 死锁原因

### 3.2 解决死锁

-  **超时等待**

	超时时间可以通过参数控制：_innodb_lock_wait_timeout_ 设置，默认是50s，如果时间太短就可能导致正常等待也被异常

-  **死锁检测**

	发现死锁，主动回滚死锁中的一个事务，让其他事务得以继续执行。参数：_innodb_deadline_detect = on_但这个检测是有额外负担的，要控制负担有两个思路
	1.  确保不会死锁可以将死锁检测关闭
	2.  控制并发。如限流、热点更新HotKey（核心：处理流程异步化如内存-写日志-日志消费）待确认：业务延迟怎么办？


### 3.3 死锁case

-  **锁ID + 锁索引 -> 冲突**

	![[image-MySQL-8 InnoDB 锁-20240604193320558.png|600]]


## 4、加锁分析

### 1、SELECT

1. RU隔离级别下：不加锁直接读取记录的最新版本。可能发生脏读、不可重复读和幻读问题
2. RC隔离级别下：不加锁每次执行SELECT都会生成一个ReadView，这样解决了脏读问题但没解决不可重读读和幻读问题
3. RR隔离级别下：不加锁只在第一次执行SELECT语句时生成一个ReadView，这样解决了脏读、不可重复读和幻读问题

### 2、锁定读的语句

四种语句可以一起讨论：select … lock in share mode、select … for update、update … 、delete …

1. RU / RC 隔离级别
    1. 主键等值查询（select … where id = 1 for update）：主键的那条记录加上锁
    2. 主键范围查询（select … where id ≤ 10 for update）：这里理论上是锁（-%，10）但有点特殊的是server层判断到10以后的数据满足不满足条件的时候会先加锁，然后判断不满足再做释放
    3. 二级索引等值查询（select … where name = ‘a’ for update）：先锁二级索引再锁聚簇索引。但有一点如果这时候有更新语句同样更新这个name=’a’ 就可能发生更新锁定聚簇索引而查询锁定了二级索引而发生的死锁。（为什么更新要先锁聚簇索引再锁二级索引？？？）
    4. 二级索引范围查询（select … where name > ‘a’ for update ）：先锁完「二级索引 + 聚簇索引」再继续后推
2. RR 隔离级别（最主要的就是解决幻读问题）
    1. 主键等值查询：主键上加锁。但如果值不存在为了禁止幻读现象会在前后区间加锁
    2. 主键范围查询（select … where id > ）：

