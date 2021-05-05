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

NLJ本来其实就不慢，这个只是锦上添花，慢的是BNL，就是被驱动表不能走索引，只是进行全表扫描的算法。

下面来看看对BNL算法如何优化的问题。



#### BNL算法的性能问题

上一节我们说到，为了简单描述，我们假设join_buffer可以放下驱动表的所有数据，需要比对N*M次，且是在内存中进行比对。

但是实际上扫描的磁盘数= N + (N / K) * M 其中N和M表示驱动表和被驱动表的行数。K表示join_buffer的大小，由上面的公式我们可以看出，驱动表应该选择小表。

但是我们上面的讨论有个假设，就是N和M是行数，但是K是大小，这个只是一个简单的模型。

基于这个模型，我们知道被驱动表会被多次扫描，因为数据页是会加载在内存中的，所以如果被驱动表是一个大的冷数据表；

会导致不停的随机读取磁盘操作，这会带来比较多的IO消耗。除此之外还有什么影响吗？



我们知道内存中其实是有数据页的缓存的，名为Buffer_Pool，为了解决冷的大数据表的全表扫描而导致的缓存失效，把Buffer_Pool按照一定比例拆分，young层的占5/8 剩下的都是old层。在old层呆过超过1s之后会自动进到young层。



我们再来说下这个问题，当我们不停的对一个大的冷数据表进行扫描的时候，会把数据页一直刷到Pool中，这有什么问题呢？

问题就是会一直不停的刷，不停的替换，导致那些本来应该进化到young的数据页一直被替换，而young层本来应该被淘汰的一直没有被淘汰。

而且这个影响只能由本次扫描结束后，慢慢的查询，累积之渐才能慢慢替换。

这就是BNL算法的问题：

1. 会占用较多的io
2. 因为是在内存中比较，会占用较高的cpu
3. 会导致内存中的数据页的淘汰短期失效。具体表现是那些应该进入young层的存活不够1s，这进一步导致了那些young层中应该被替换的迟迟无法被替换。



#### BNL算法的优化

BNL算法慢的原因根本所在，是不能使用索引，倘若我们在那个字段上新增索引就没有这些问题了。

这个建议的代价如何呢？

这是一个DDL，虽然可以OnlineDDL，会影响正常运行的查询。

建立索引会占用磁盘空间。

现在假定我们评估之后，决定不需要在原表上新建索引。

此时我们引出了一个新的概念**临时表**。

为了方便叙述，我们假定我们的查询语句如下：

```mysql
select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;
```

我们先把t2中满足条件的行全部放到临时表中，并在b上建立索引，然后让t1和那个临时表join。

```mysql
create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
insert into temp_t select * from t2 where b>=1 and b<=2000;
select * from t1 join temp_t on (t1.b=temp_t.b);
```

我们在最开始的时候在t1插入了1000条数据，在t2插入了100w条数据。

如果没使用临时表之前，我们来看内存比较次数和磁盘读取次数分别是多少：

1. 内存比较次数:1000*1000000=1,000,000,000  10亿次比较。
2. 磁盘读取，t1表记录较少，可以一次性放入join_buffer中，一个读取了1000 +1000000次磁盘；



如果使用了临时表：

1. 磁盘读取，建立临时表的时候因为没办法使用索引，会进行全表扫描，100w， 假设临时表也是存放在磁盘上的，~~~那么还会加上满足条件的临时表 2000,再加上t1原本的数据1000~~~，这边叙述的有问题，因为临时表已经可以使用索引进行扫描了，所以这边只会进行1000次等值查询，1000+1000 = 2000；
2. 内存比较，1000次索引等值查询。



#### 扩展hash-join

我们在说BNL的时候，发现内存比较的时间复杂度是O(m*n),这是因为join_buffer中的数据结果是无序数组。如果可以改成hash表，时间复杂度就会降为O(Max(m,n));

但是mysql的执行器偏没有这样的hash，只有认为的进行hash-join了。

具体的操作是做两次查询，然后再代码中进行比较。











 

