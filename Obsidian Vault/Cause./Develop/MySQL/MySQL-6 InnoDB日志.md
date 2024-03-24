
## redo log（重做日志）

`WAL技术` 全称 Write-Ahead Logging，先写日志再写磁盘。具体来说更新一条记录的时候InnoDB会先写到redo log，并更新内存（buffer pool），这时候更新就算完成了。然后会在适当时候把更新到内存

### Mini-Transaction
为了保证一条插入语句因页面分裂而造成多个redo日志保存时候的原子性，InnoDB允许一组同时提交。

### Log Sequence Number


### checkpoint

![[MySQL-6 InnoDB日志-redo log.png]]

write pos是当前记录的位置，一边写一边后移
check point是当前擦除的位置
write pos 和 check point 之间是空着部分还可以继续记录，如果write 追上 check 表示内存满了，则需要停下来清楚数据

## binlog（归档日志） 

	redo log VS binlog
	-  redo log 是 InnoDB 特有的，binlog 是MySQL 的 Server 层实现的，所有引擎都可以使用
	-  redo log 是 物理日志，记录的是 “在某个数据页上做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑
	-  redo log 是循环写，binlog 是追加写


1.  执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行（在内存中直接返回，否则从磁盘读入内存再返回）
2.  执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3.  引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
4.  执行器生成这个操作的 binlog，并把 binlog 写入磁盘。（ 如果此时binlog写完后崩了？恢复后提交）
5.  执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。

### 两阶段提交
	让r edo log 和 binlog 这两个状态保持逻辑上的一致。

当需要恢复到指定的某一秒时，比如某天下午两点发现中午十二点有一次误删表，需要找回数据，那你可以这么做：
1.  首先，找到最近的一次全量备份，如果你运气好，可能就是昨天晚上的一个备份，从这个备份恢复到临时库；
2.  然后，从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。


## undo log
为了实现事务的原子性，InnoDB存储引擎在实际进行增删改一条记录时，都需要先把对应的undo日志记录下来。一般每对一条记录做一次改动就对应着一条undo日志。
有两个作用：一是提供回滚；二是实现MVCC功能。