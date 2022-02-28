# mysql 优化



## 一、优化 sql statements(statements 是数据逻辑上概念）





### 1、whre 优化

* 移除非必要的插入

   ((a AND b) AND c OR (((a AND b) AND (c AND d)))) -> (a AND b AND c) OR (a AND b AND c AND d)

* 常量folding

​      (a<b AND b=c) AND a=5 -> b>5 AND b=c AND a=5

* 常量条件移除

  (b>=5 AND b=5) OR (b=6 AND 5=5) OR (b=7 AND 5=6) -> b=5 OR b=6

* 被使用在indexes 上的常量表达式将只被计算一次。

* 单表上的  count* 在 myisam 和 memory 表中改写成直接通过表信息获取。 对于not null 表达式，并且只在一个表中实现的时候，它也会工作。

* 提前探测无效的常量表达式。mysql 快速探测一些不可能返回结果的表达式然后返回0条数据。

* having 将被归并在where，如果没有使用 group by 或者聚合函数。

* 对于每一个在join中的table，将尽可能的重新构造一个更简单的where条件 并且尽可能跳过更多的行。

* 所有的常量表将比其他的表更有先的查询。 常量表包含下面的情况：

  * 空表或者表里面只有一条数据。

  * 所有的索引部分使用的常量表达式 并且被定义成非空的表达式，并且也被定义成非空，同时， 查询这个表的where条件 使用了主键或唯一索引。

    All of the following tables are used as constant tables:

    ```sql
    SELECT * FROM t WHERE primary_key=1;
    SELECT * FROM t1,t2
      WHERE t1.primary_key=1 AND t2.primary_key=t1.id;
    ```

* 尽可能使用最好的连接方式来连接表。如果所有的在 order by 和group by的column都在同一个表，则这个表将在join中更可能的优先查询。
* 