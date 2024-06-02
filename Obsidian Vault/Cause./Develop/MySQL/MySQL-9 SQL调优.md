
### 慢查SQL


要结合业务说明，某一次线上报警出现慢SQL，或者接口RT比较长，怎么处理的

-  **explain**

	首先explain执行的SQL，关键查看几个核心字段
	1.  type：是否走索引
	2.  possible_keys：可能走的索引
	3.  keys：实际走的索引
	4.  extra：查询信息（索引覆盖、join、sort..）
	[[MySQL-5 InnoDB 索引#5.23 索引失效]]
	

- **没走索引**

	1.  索引没正确创建
	2.  索引区分度不高
	3.  表太小，不走索引
	4.  使用函数等


- **走索引**

	1.  深翻页 -> 延迟加载?
	2.  order by id 和 order by created