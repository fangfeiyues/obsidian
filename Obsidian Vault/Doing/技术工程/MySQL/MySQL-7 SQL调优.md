
### 慢查SQL


要结合业务说明，某一次线上报警出现慢SQL，或者接口RT比较长，怎么处理的

-  **explain**

	首先explain执行的SQL，关键查看几个核心字段
	1.  type：是否走索引，理想情况下应该是`const`、`eq_ref`、`ref`
	2.  possible_keys：可能走的索引
	3.  keys：实际走的索引
	4.  extra：查询信息，理想情况不应包含`Using filesort` 或 `Using tempory`
	[[MySQL-5 InnoDB 索引#5.23 索引失效]]
	

- **没走索引**

	1.  索引没正确创建
	2.  索引区分度不高
	3.  表太小，不走索引
	4.  使用函数等


- **走索引**

	1.  深翻页 -> 延迟加载?
	2.  order by id 和 order by created


### 热点更新

Inventory Hint 技术

 当我们打上如 COMMIT_ON_SUCCESS 等标记的时候，MySQL内核层会自动识别带此类标记的更新操作，在一定的时间间隔内，将手机到的更新操作按照主键或者唯一键进行分组。
 
 这样更新相同的操作就会被分到同一组，就可以根据热点做了分组后，就可以进一步优化
 
	 1.  减少行级锁的申请等待
	 2.  减少B+树的索引便利
	 3.  减少事务提交次数


![[Pasted image 20250417200506.png|600]]