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