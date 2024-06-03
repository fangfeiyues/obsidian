
MySQL查找要回答的几个问题（ 持续更新ing... ）
1.  查询路子怎么样的？怎么支持更多更快、以及范围查找模糊匹配...等能力
2.  怎么解决高流量
3.  怎么保证高可用（宕机后怎么自动恢复）

## 查询计划
### 访问方式
1. const：常数级，代表查询的代价可以忽略不计的如主键查或唯一索引
2. ref：二级索引执行回表查
3. ref_or_null：二级索引字段 + NULL（在记录最前面）
4. range：范围查询
5. index：不用回表的二级索引查询
6. all：扫描聚簇索引

### 查询原理

`SELECT、FROM、JOIN、WHERE、GROUP BY、HAVING、ORDER BY、LIMIT` 执行顺序
1. FROM：第一步识别查询涉及的表
2. JOIN：优化表查询效率
3.  WHERE：对 JOIN 操作的结果过滤
4.  GROUP BY：条件过滤后分组数据
5.  HAVING：分组后继续筛选
6.  SELECT：选择特定的列
7.  ORDER BY：拿到列后进行排序
8.  LIMIT：限制返回的行数


##### Intersection & Union合并

##### Join
