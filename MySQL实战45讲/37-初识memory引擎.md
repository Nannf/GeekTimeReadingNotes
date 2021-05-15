#### 概览

- memory引擎和innodb引擎的索引存储方式的不同
- memory引擎存储的优劣
- 给出使用memory引擎使用的场景





#### 索引存储方式的探究

```mysql
create table t1(id int primary key, c int) engine=Memory;
create table t2(id int primary key, c int) engine=innodb;
insert into t1 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
insert into t2 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
```

![img](https://static001.geekbang.org/resource/image/3f/e6/3fb1100b6e3390357d4efff0ba4765e6.png)



innodb引擎的数据是存储在主键索引上的，主键索引默认是b+树，b+树是有序的

![img](https://static001.geekbang.org/resource/image/4e/8d/4e29e4f9db55ace6ab09161c68ad8c8d.jpg)



memory引擎的索引和数据是分开的，数据是存放在数组中。

![img](https://static001.geekbang.org/resource/image/dd/84/dde03e92074cecba4154d30cd16a9684.jpg)

所以查询数据时，是按照插入顺序来返回的。且索引是hash索引，所以不是有序的。



我们把innodb这种，数据保存在主键索引树上，其他所以保存主键id的存储方式称为**索引组织表**；

把memory这种。数据单独存放，索引存储的是数据存放位置的，称为**堆组织表**；



- innodb是顺序存放的，memory是插入顺序存放的
- innodb的数据插入必须在固定的数据页上，memory是从前到后遍历数组，只要有空间，就可以插入
- 数据位置发生变化时，innodb只需修改主键索引，内存表需要修改所有的索引。
  - 数据位置发生变化，什么时候会发生变化？
  - 这个我后来明白了，因为innodb的数据只和主键索引有关，其他所有的索引都是通过索引id和数据间接关联，所以当数据的物理存储发生变动时，只有主键索引会受影响
- 还是上面的那个原因，当我们在innodb时使用普通索引进行查找数据时，需要先查找id，再根据id查找数据，需要多进行一次
- memory是使用数组存储的，所以是不允许text和blob这种动态大小的字段类型的，varchar也会被默认的转换成char，一条记录在表结构创建好的时候，就已经确定了。



```mysql
delete from t1 where id=5;
insert into t1 values(10,10);
select * from t1;
```

这个对应上面的第二点，就是见缝插针，（10，10）会插在（4，4）的后面



```mysql
select * from t1 where id<5;
```

这个没啥用，因为memory的主键索引是hash，hash在进行范围查找的时候用不上。





