| Column | Meaning |
|--------|---------|
| id | The SELECT identifier |
| select_type | The SELECT type |
| table | The table for the output row |
| partitions | The matching partitions |
| type | The join type |
| possible_keys | The possible indexes to choose |
| key | The index actually chosen |
| key_len | The length of the chosen key |
| ref | The columns compared to the index |
| rows | Estimate of rows to be examined |
| filtered | Percentage of rows filtered by table condition |
| extra | Additional information |


## id

意为select查询的序列号，包含一组数字，表示查询中执行select字句或操作表的顺序

- id相同，顺序自然自上而下
- id全不同，如果是子查询，id的序号会递增，id数字越大越先执行
- id部分相同，先按数字大的先执行，数字相同的按自上而下顺序执行

## select_type

查询类型，用于区别普通查询、联合查询、子查询等复杂查询

- **simple**
    
    简单的select查询，查询中不包含子查询或union
    
- **primary**
    
    查询中包含子查询，最外层查询则标记为primary
    
- **subquery**
    
    在select或where中包含子查询
    
- **derived**
    
    在from列表中包含的子查询，也叫派生类，MySQL会递归执行这些子查询，把结果放在临时表里
    
- **union**
    
    若第二个select出现在union之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
    

## table

显示这一行的数据是关于哪张表，内容为表名或者别名，可能是临时表或者union合并结果集

## type

显示访问类型，表示以何种方式访问了数据，从最好到最差的排列为：

**system > const > eq_ref > ref > range > index > all**

**一般情况下，得保证查询至少达到range，最好能达到ref**

- **system** 
表里只有一行记录，是const的特例，一般不会出现
- **const**
表里至多有一个匹配行
- **eq_ref** 
主键索引（primary key）或者非空唯一索引（unique not null）等值扫描
- **ref** 
非主键非唯一索引等值扫描
- **range** 
利用索引查询的时候限定了范围，一般出现于between <（<=） > (>=) in 等查询
- **index** 
全索引扫描，比all效率高，index从索引中扫描，而all是从硬盘上扫描
- **all** 
全表扫描，一般出现此类型的sql并且数据量较大的话就要进行优化

## **possible_keys**

显示的是可能应用在这张表中的索引，一个或多个，查询涉及到的字段若存在索引，则该索引会被列出，但不一定被查询实际使用

## **key**

实际使用的索引，如果为null，则没有用到索引
若查询中用到了索引覆盖，则该索引和查询的select字段重叠，仅出现在key列表中

## **key_len**

表示索引中使用的字节数，可以通过key_len计算查询中使用的索引长度，在不损失精度的情况下长度越短越好

## **ref**

显示索引的哪一列被使用了，如果可能的话，是一个常数

## **rows**

 根据表的统计信息及索引选用情况，大致估算出找到所需记录需要读取的行数

## **extra**

包含的额外信息，无法在一个字段中概述但是依旧重要的额外信息

- **using filesort** 
文件排序意为MySQL无法利用索引进行排序，而使用了外部的排序算法，常见于order by 或 group by
- **using temporary** 
使用了临时表保存中间结果，查询完成后临时表删除
- **using index** 
索引覆盖意为直接从索引中读取数据，而不用访问数据表，如果同时出现 using where 表明索引被用来执行索引键值的查找，否则，索引被用来读取数据而不是执行查找操作
- **using where** 
使用where进行条件过滤
- **using join buffer** 
使用了连接缓存
- **impossible where**  
where语句结果是false
- **select tables optimized away** 
在没有group by字句情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT（*）操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化
- **distinct** 
优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作