InnoDB 用缓冲池 buffer pool 管理内存

```SQL
[server]
innodb_buffer_pool_size = 268435456 （ 265M，默认128M ）
```

## Buffer Pool

![[MySQL-4 Buffer Pool.png|600]]

### Free链表

![[MySQL-4 Buffer Pool-1.png|600]]

LRU管理，线型管理缓存的数据。

-  **优化**
	1.  `预读失效`：预读会数据加载到buffer中，如果这些数据用不到可能导致缓存命中率下降
	2.  `Buffer Pool污染`：大量数据加载如全表扫描，会换血一批数据导致缓存命中率受影响
	  通过配置 `innodb_old_blocks_time` 来按一定比例Free链表分成两截，分别代表频率使用高的 **热数据（young区域）**  和  频率一般的 **冷数据（old区域）**，这样就能保证数据来先到 old区不影响正常的 young区域

- **进一步优化**
	 防止 `young` 区域的热点数据被频繁移动到头部，只有被访问的缓存页位于`young`区域的`1/4`的后边，才会被移动到`LRU链表`头部


### Flush链表

### 刷盘

#### 刷盘时机

1.  redo log 满了的情况下，会主动触发脏页刷新到磁盘（这种情况整个系统就不能再接受更新）
2.  Buffer Pool 空间不足需要将一部分数据页淘汰掉，如果淘汰的是脏页，需要先将脏页同步到磁盘
	`为什么不直接淘汰内存？下次查询的时候从磁盘读取再结合redo log
3.  MySQL 认为空闲时后台线程回定期将适量的脏页刷入到磁盘；即使非空闲时也会见缝插针地刷盘；
4.  MySQL 正常关闭之前，会把所有的脏页刷入到磁盘；

#### 刷盘策略

-  **刷盘影响**
	1.  一个查询要淘汰的脏页个数太多，会导致查询时间变长
	2.  日志写满，则会全部堵住，写性能为0

- **解决方案**
	1.  设置redo log写磁盘能力 `innodb_io_capacity`  看SSD硬盘
	2.  控制脏页比例 `innodb_max_dirty_pages_pct` 默认75%

## Change Pool

-  **更新步骤**
	1、如果数据在内存，则直接更新
	2、否则会缓存操作到 change buffer，来避免读磁盘数据到内存
	3、再等下次访问数据页时，会把数据读到内存再触发 merge，或 后台定期merge

-  **使用条件**
	1、唯一索引不能，由于数据唯一性限制
	2、change buffer 用的是 buffer pool内存，不能无限大，`innodb_change_buffer_max_size` 控制

-  **使用场景**
	1、 写多读少的业务如账单、日志等。写多可以一次性merge；读少可以减少merge