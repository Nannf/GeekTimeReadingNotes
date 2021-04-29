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



