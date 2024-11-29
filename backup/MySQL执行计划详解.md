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

## **filtered**
filtered是结果集行数和扫描行数的比值的百分数，这个值是越大越好。
举几个例子：
1、如果我们使用主键等值查询，那么结果集是1行，由于可以通过聚簇索引直接找到这行记录，所以总共只需要扫描1行，filtered就会是100；
2、例如一个查询的结果集是10行，为了查到这10行数据，mysql一共扫描了100行数据，那么filtered值就是10。

## **extra**

包含的额外信息，无法在一个字段中概述但是依旧重要的额外信息

### **Using index**
代表使用了**覆盖索引**。
当前命中的索引中包含了sql语句查询的所有字段时，就不需要再回表了，直接使用索引中的字段即可。

> [!TIP]
> 需要注意的是**Using index**并不代表一定使用了这个索引做查询，例如：`test_table`表有索引`idx_a_b`是`a,b`两个字段, 此时执行`select a, b from test_table`，会命中`idx_a_b`，执行计划`extra`字段包含`Using index`，但是此时做的是全索引扫描。
在实操中没必要特意去追求覆盖索引，一般情况下，回表都不会是瓶颈所在。

### **Using where** 
说明where条件没有全都包含在索引内，在使用索引筛选完数据后，还需要再次回表筛选其他的where条件。

这种情况是经常会发生的，需不需要优化要看具体情况，如果索引筛选完的结果集很小，那么回表的成本也会比较小；如果结果集很大，那么回表的成本也会高，就要考虑优化了。

### **Using index condition** 


### **Using filesort** 
文件排序意为MySQL无法利用索引进行排序，而使用了外部的排序算法，常见于order by 或 group by

### **Using join buffer** 
使用了连接缓存

### **impossible where**  
where语句结果是false

### **select tables optimized away** 
在没有group by字句情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT（*）操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化

### **distinct** 
优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作

### **Backward index scan**
使用索引进行倒序的扫描。

### **Using temporary** 
使用了临时表保存中间结果，查询完成后临时表删除