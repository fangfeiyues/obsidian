## 锁类型

### 悲观锁

```sql
-- 0.开始事务
begin; 
-- 1.查询出商品库存信息, 这里用for update当前读防止读到MVCC快照下其他view之后的操作
select quantity from items where id=1 for update;
-- 2.修改商品库存为2
update items set quantity=2 where id = 1;
-- 3.提交事务
commit;
```

### 乐观锁

```sql
-- 查询出商品库存信息，quantity = 3
select quantity from items where id=1
-- 修改商品库存为2，同时避免ABA问题
update items set quantity=2 where id=1 and quantity = 3 and version = 2;;
```

怎么解决脏读、不可重复读、幻读这些问题呢？其实有两种可选的解决方案：
1. 读操作利用多版本并发控制（MVCC），写操作进行加锁
2. 读、写操作都采用加锁的方式。如银行转帐这些不允许读快照场景

### 一致性读

事务利用MVCC进行的读取操作称之为一致性读，或者一致性无锁读，有的地方也称之为快照读。所有普通的SELECT语句（plain SELECT）在 READ COMMITTED、REPEATABLE READ 隔离级别下都算是一致性读

比方说：SELECT * FROM t1 INNER JOIN t2 ON t1.col1 = t2.col2

### 一致性锁定度

显示地对数据库读操作进行加锁以保证数据逻辑的一致性，如：

SELECT ... FOR UPDATE

SELECT ... LOCK IN SHARE MODE

### 自增长与锁

系统实现自动给列修饰 AUTO_INCRMENT 的原因主要有两个

1. 采用AUTO_INC锁，为每个插入的行加上锁待语句执行结束后释放那么在加锁的过程中其他的插入都要阻塞
2. 采用轻量级的锁，在为插入语句生成AUTO_INCREMENT修饰的列的值时获取一下这个轻量级锁，然后生成本次插入语句需要用到的AUTO_INCREMENT列的值之后，就把该轻量级锁释放掉，并不需要等到整个插入语句执行完才释放锁

设计InnoDB的大叔提供了一个称之为innodb_autoinc_lock_mode的系统变量来控制到底使用上述两种方式中的哪种来为AUTO_INCREMENT修饰的列进行赋值，当innodb_autoinc_lock_mode值为0时，一律采用AUTO-INC锁；当innodb_autoinc_lock_mode值为2时，一律采用轻量级锁；当innodb_autoinc_lock_mode值为1时，两种方式混着来（也就是在插入记录数量确定时采用轻量级锁，不确定时使用AUTO-INC锁）。不过当innodb_autoinc_lock_mode值为2时，可能会造成不同事务中的插入语句为AUTO_INCREMENT修饰的列生成的值是交叉的，在有主从复制的场景中是不安全的

### 行锁

Record Lock：记录锁

Gap Lock：间隙锁会锁记录前的两个之间的间隙如update a = 8则在(3,8)之间加锁

Next-Key Lock：gap-lock + 行锁 即 (3,8]

**怎么减少行锁对性能的影响？**

见死锁。

## 锁解决一些隔离性问题

### 不可重复读

不可重复读和脏读的区别是：脏读是读到未提交的数据而不可重复读读到的却是已经提交的数据。

通过 `Next-Key Lock` 算法来避免不可重复读的问题

### 幻读

幻读指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行

-  **幻读带来的问题**

	![[MySQL-7 InnoDB 锁.png|500]]
	因为这三个查询都是加了 `for update`（当前读），当前读的规则，就是要能读到所有已经提交的记录的最新值。并且，session B 和 sessionC 的两条语句，执行后就会提交，所以 Q2 和 Q3 就是应该看到这两个事务的操作效果，而且也看到了，这跟事务的可见性规则并不矛盾。


-  **语义上**

	![[MySQL-7 InnoDB 锁-1.png|500]]
	如果 Session A 在T1时刻就声明了要把所有 d=5 的行锁住，但是 Session B 的更新还是能执行因为并没有把 id = 0 的锁住。这样就破坏了A的语义


-  **数据一致性上**

	![[MySQL-7 InnoDB 锁-2.png|500]]

	如果由于 session A 把所有的行都加了写锁，所以 session B 在执行第一个 update 语句的时候就被锁住了。需要等到 T6 时刻 session A 提交以后，session B 才能继续执行

```sql
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/

update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/
```

这样第三行还是会把新插入的(1,5,5) 修改成 (1,5,100) ，还是阻止不了新插入的行的更新

## 死锁

### 死锁原因

### 解决死锁

-  **超时等待**

	超时时间可以通过参数控制：_innodb_lock_wait_timeout_ 设置，默认是50s，如果时间太短就可能导致正常等待也被异常

-  **死锁检测**

	发现死锁，主动回滚死锁中的一个事务，让其他事务得以继续执行。参数：_innodb_deadline_detect = on_但这个检测是有额外负担的，要控制负担有两个思路
	1.  确保不会死锁可以将死锁检测关闭
	2.  控制并发。如限流、热点更新HotKey（核心：处理流程异步化如内存-写日志-日志消费）待确认：业务延迟怎么办？

## 加锁分析

### 1、普通的SELECT语句

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

