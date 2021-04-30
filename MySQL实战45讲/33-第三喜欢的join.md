#### 第三喜欢的join

感冒了，摸鱼

建表如下：

```mysql

CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)

```



#### Index Nested-Loop Join

```mysql
select * from t1 straight_join t2 on t1.a = t2.a;
```

- 如果不用straight 关键词，那么mysql的优化器会随机（感觉也是有评判标准的，为了便于我们的分析）选择一个表作为驱动表，驱动表的含义就是会全表遍历的表。

![img](https://static001.geekbang.org/resource/image/4b/90/4b9cb0e0b83618e01c9bfde44a0ea990.png)

执行计划如上图所示。

t1表走了全表扫描，t2表使用了a索引。

我们可以得出执行的步骤大致如下：

1. 从t1表中的主键索引上顺序取出一行
2. 获取其中的a字段，到t2中查询值等于a的主键id
3. 根据主键id获取到所有的返回值，并放到结果集中
4. 依次执行上面的1-3步，直到遍历结束。

我们回到这个奇怪的join名称

这个类似于一个嵌套的for循环，而且使用了被驱动表的索引，所以由此得名。



我们再来分析下一个扫描了几行

t1表扫描了100行，t2表也扫描了100行。一共扫描了两百次。

如果不用join，那么我们需要先查询select a from t1 ;然后循环得到的结果集，每次做一个查询，最好的扫描行数也是200行，但是需要和数据库有101次交互。

虽然我们使用数据库连接池后比较耗时的握手环节被取消了，但是还是比使用join的一次查询来的简洁明了。

**故，不是不能用join**



假设驱动表的行数是N，被驱动表的行数是M，且我们查询的时候可以使用被驱动表的索引。

那么时间复杂度是 N + N * logM

所以我们一般选择小表作为驱动表。



当我们的被驱动表无法使用索引的时候

#### Simple Nested-Loop Join

```mysql
select * from t1 straight_join t2 on t1.a = t2.b;
```

此时的时间复杂度就是N*M,此时选择谁做驱动表都是一样的。

当N和M都是10w级别的表的时候，这个复杂度已经突破天际。更为关键的是，这个是磁盘的随机io。



既然我们不能再算法上进行优化，那么我们就考虑同样的执行次数下，如何减少执行时间。

一个比较容易想到的方案就是通过把磁盘缓存ssd或者是使用内存。



下面就是基于内存进行优化的。

#### Block Nested-Loop Join

这个内存空间，是这个连接线程所能持有的，名为  **join_buffer**，其大小由参数**join_buffer_size**进行处理。

跟之前不同的是，会把t1表中需要被返回的和比对时需要用到的参数加到join_buffer中，然后逐行读取t2的记录和内存中的进行比较。

扫描行数是 ： N + M；

内存比较次数 ： N * M；



但是join_buffer的长度是有限的，如果放不下全表怎么办，这就出现了标题中的Block分段放。每次都取join_buffer大小的数据放到内存中然后逐个判断。

扫描行数 ：  N + K * M * N； 这个 K * N 中的K是一个常数，值为把N个记录放到buffer中需要放几次。

内存比较次数： N* M

 由以上我们可以得知，把join_buffer_size适当调大可以减少扫描的次数，也可以优化join的查询。





#### 总结：

##### 能不能用join

1. 当是 Index Nested-Loop Join时，就是可以使用被驱动表的索引时可以使用；

其他的不建议使用。



##### join如何选择驱动表，选择大表还是小表

1. Index Nested-Loop Join选择小表，因为驱动表会做全表扫描，这个小表的含义是数量小。

2. Block Nested-Loop Join

   1. 当join_buffer足够大时，一样的，随便选

   2. 当不够大时，需要选择小表，这里的小表的含义是，需要返回的，需要加到join_buffer中的数据尽量小。

      1. 可能t1有100行，t2有200行，但是t2只需要返回一个id，t1需要返回全表。

      更准确的说，需要让join的两个表各自计算自己满足条件的需要返回的字段的大小来决定。

      





 





