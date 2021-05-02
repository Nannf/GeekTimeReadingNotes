#### 前言

上节我们提到了两种join

一种是可以使用被驱动表索引的Index Nested-Loop Join(NLJ)

一种是无法使用被驱动表索引的Block Nested-Loop Join(BNL)



这两种方式都有可以优化的空间。



我们建表如下

```mysql
create table t1(id int primary key, a int, b int, index(a));
create table t2 like t1;
drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t1 values(i, 1001-i, i);
    set i=i+1;
  end while;
  
  set i=1;
  while(i<=1000000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;

end;;
delimiter ;
call idata();
```



#### Multi-Range Read优化

#### 核心

优化的核心是使用了磁盘的顺序读。

磁盘顺序读的实现有几个前提和假设

- 前提一： 磁盘的读是按照页为单位，当我们读取一行数据时，实际读取的是这个数据行所在的数据页
- 前提二： 数据页是放在内存中，每次进行查询之前都要先判断数据页在不在内存中，如果不在就进行查盘操作，否则直接读取内存

- 假设一：当我们按照主键id递增顺序来进行磁盘查询的时候，一次的查询加载的数据页大概率会把后面的行也加载出来。

基于上面的前提和假设，引入了此优化。

根据叙述，我们可以得知MRR优化的适用场景

1. 需要去主键索引上进行回表
2. 回表时不止需要回一次

基于此场景，我们先使用一个缓存，read_rnd_buffer来存储需要回表的id，等收集所有的id之后，先进行排序，然后按照顺序逐个访问，这样就有机会少访问磁盘。

这里面有几个问题：

1. 如果buffer中放不下那么多的id，会先放慢，然后读取磁盘，读取完成后，把缓存清空，接着放。
2. 这个判断总感觉有点儿戏，把优化交给了命运。

这个我不清楚底层的磁盘存储是不是可以让这个实现。

mysql的优化器是默认把mrr优化关闭的。可以通过set optimizer_switch="mrr_cost_based=off"，让优化器不在评估mrr优化是否真的能带来优化，而强制走mrr优化。



![img](https://static001.geekbang.org/resource/image/d5/c7/d502fbaea7cac6f815c626b078da86c7.jpg)





#### Batched Key Access

![img](https://static001.geekbang.org/resource/image/10/3d/10e14e8b9691ac6337d457172b641a3d.jpg)

NLJ join方式，我们可以看到是取出驱动表t1的每一行字段，然后去和t2表中索引表去碰，这显然不符合我们刚刚说的MRR优化。

倘若我们让t1一次取很多满足条件的索引字段a出来，然后排好序之后，再拿排好序的字段，到t2表的索引树上逐行去碰，可能也能触发说的MRR优化。

但是多个a字段存储再哪呢？

join_buffer出来了。

同样的，BKA算法是依赖MRR的，所以需要再执行之前执行

set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';

打开MRR和BKA的开关。





