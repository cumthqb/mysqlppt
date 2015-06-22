title: Mysql语句调优
speaker: 韩庆宾
transition: slide2
theme: colors
[slide]
# Mysql调优
[slide]
# Sql执行过程（关键步骤）
---
* 打开游标
* 语句解析，语法、语义检查
* <span id="css-red">**绑定变量**</span>
* <span id="css-red">**查询转换，根据内部规则对语句进行优化**</span>
* <span id="css-red">**根据统计信息（索引、采样数据等信息），生成执行计划**</span>
* 执行语句，返回结果
* 关闭游标

> mysql没有shared pool概念，不会对硬解析结果进行缓存。每次语句执行都是硬解析。

>> 在同一个游标中，同一个语句不会被重新解析（批处理）。


<style>
#css-red{
    color: red;
}
</style>

[slide]
# 绑定变量
---
* <span id="css-red">**业务系统中，能使用绑定变量的语句一定使用绑定变量**</span>
* <span id="css-red">**防止SQL注入**</span>
* mysql目前没有SQL共享池概念，如果有可以减少硬解析次数，缓存执行计划，提升性能。
[slide]
# 索引的使用
---
* 选择性高，频繁使用的查询条件，需要建立索引，尽量使用索引排序。
	* 订单号属于选择性高的列
	* 订单状态属于选择性低的列
* mysql查询一次只使用一个索引，根据语句的使用频繁情况，建立组合索引。
* 组合索引，遵循『最左前缀』原则，建立组合索引后，查询条件一定要带上最左边列作为where条件。
* where+order by 可以建立组合索引提高效率
	<pre><code class="sql">select col1,col2 from table where col1=:val1 order by col2</code></pre>
	建立组合索引(col1,col2)提升效率
[slide]
* 小表不需要建立索引（数据量低于500）。
* 变更频繁的字段不要建立索引。
* 索引不是越多越好，一个表上索引不要超过3个。尤其是需要频繁写入的表。
* 不要迷信索引，实际执行状况需要根据执行计划来确认。
[slide]
# 执行计划
---
* explain是常用的sql性能分析方法。
* 优化器会对SQL进行优化，通过查看执行计划可以确认索引是否使用合理。
* 可以查看到SQL的执行轨迹。
* 可以查询SQL优化后的结果。
* 查询执行计划的方法：
	* <pre><code class="sql">mysql> explain select * from table;</code></pre>
	* 添加 extended 参数，结合show warnings查看SQL优化后的结果
		<pre><code class="sql">mysql> explain extended select * from table;
mysql> show warnings;</code></pre>
	* 其他第三方工具
[slide]
<style>
#css-blue{
    color: blue;
}
</style>
<a href="https://github.com/freedomopencoder/mysqlppt/blob/master/mysqlexplain.ppt" target="_blank"><span id="css-blue">执行计划详解（点击下载）</span></a>

<pre><code class="sql">mysql> explain select * from demo_order;
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | demo_order | ALL  | NULL          | NULL | NULL    | NULL |    1 | NULL  |
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
1 row in set (0.00 sec)

mysql> explain extended select * from demo_order;
+----+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | demo_order | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
mysql> show warnings;
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                         |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select `demo`.`demo_order`.`id` AS `id`,`demo`.`demo_order`.`created_time` AS `created_time`,`demo`.`demo_order`.`user_id` AS `user_id` from `demo`.`demo_order` |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)</code></pre>
[slide]
# SQL语句编写原则
[slide]
* 索引字段设置NOT NULL 赋默认值，禁止对索引字段使用 is null或is not null
	* 低效：<pre><code class="sql">select col1,col2 from recordA a where col1 is null</code></pre>
	* 高效：<pre><code class="sql">select col1,col2 from recordA a where col1=0</code></pre>
* 子查询用于判断是否存在时，使用"exists" "not exists" 避免 "in" "not in"
	* 低效：<pre><code class="sql">select col1,col2 from recordA a where col1 in (select col1 from recordB b)</code></pre>
	* 高效：<pre><code class="sql">select col1,col2 from recordA a where exists (select 1 from recordB b where a.col1=b.col1)</code></pre>
[slide]	
* in 在语句中尽量避免，常量数据除外。能用表连接就用表连接，不能使用时，可以用exists代替
	* 低效：<pre><code class="sql">select col1,col2 from recordA a where col1 in (select col1 from recordB b)</code></pre>
	* 高效：<pre><code class="sql">select col1,col2 from recordA a,recordB b where a.col1=b.col1</code></pre>
* 对默认字段模糊搜索时 使用 like 'abc%'，禁止使用like '%abc%'
	* 低效：<pre><code class="sql">select col1,col2 from recordA a where col1 like '%abc%'</code></pre>
	* 高效：<pre><code class="sql">select col1,col2 from recordA a where col1 like 'abc%</code></pre>
[slide]
* 禁止对索引字段条件进行计算 如：where col1/2=10 必须使用 where col1=10*2
* 除非建立了函数索引，否则禁止对索引字段进行函数处理 如：where LENGTH(col1)=5
* 表关联查询时，关联的字段类型必须一致，否则出现强制转换可能会无法使用索引。关联字段尽量建立索引。
* 避免不必要的类型转换（例如：col1是数字类型）
	* 低效：<pre><code class="sql">select col1,col2 from recordA a where col1='1'</code></pre>
	* 高效：<pre><code class="sql">select col1,col2 from recordA a where col1=1</code></pre>
[slide]
* 增加范围查询限制，避免全范围搜索。
	* 低效：<pre><code class="sql">select col1,col2 from recordA a where created_time>'2015/01/01'</code></pre>
	* 高效：<pre><code class="sql">select col1,col2 from recordA a where created_time between '2015/01/01' and '2015/01/31'</code></pre>
* 一条语句中禁止出现3张表以上的关联，优先考虑对表做字段冗余，以空间换取时间。如果必须要三表以上关联需通过dba审核。
* select * 不要出现在代码中，mysql优化器会虽然优化为具体字段，但是如果添加或者减少字段，程序可能会出现异常，必须指定具体读取的字段。
[slide]
* 避免使用 "!=" "<>"，如果是枚举值，可以改用in 如state有0,1,2,3  where state!=0 可替换为 state in (1,2,3)或者state>0
* blob text等大容量存储字段，单独建表存放。
* or 慎用，必要时可以用union all替换，不确定时根据执行计划来确认，union all有时会产生物理读问题。
* 表之间没有关联或者说where条件中没有关联字段时，禁止出现在一条语句中，如：select * from A a,B b,C c where a.id=b.id  其中C就是多余的。
* 禁止无意义的排序，比如在子查询中进行order by 操作。
* 使用范围分区表时，禁止跨分区搜索。首先要确定分区字段，保持搜索条件在一个范围分区内。
[slide]
# 总体原则
* 能单表查询就单表查询，该冗余就冗余，空间不值钱，时间是最值钱的！
* 读写分离，非关键查询业务，一定不能影响到关键业务。
* sql调优仅限于正常情况，sql优化器内部做了很多事情，在出现响应慢时，优先查看explain
[slide]
# 应用内优化
* 使用数据库连接池，减少申请物理连接的开销，必须清楚数据库连接池几个参数的意义，合理设置。
* 避免大事务操作，将流程拆分为小事物执行。
* 非关键数据，在应用层进行缓存，减少数据库访问压力。
* 减少数据库访问次数。目前数据库是瓶颈，逻辑计算尽量放到应用层处理。应用层可以通过集群缓解计算压力。
* 日常统计、查询操作在读库操作，禁止在写库操作。








