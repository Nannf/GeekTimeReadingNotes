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



#### memory引擎与b+树索引

memory引擎默认会有一个hash主键索引。但是我们可以通过显示的创建，来构造一个b+树的索引。

```mysql
alter table t1 add index a_btree_index using btree (id);
```

![img](https://static001.geekbang.org/resource/image/17/e3/1788deca56cb83c114d8353c92e3bde3.jpg)

![img](https://static001.geekbang.org/resource/image/a8/8a/a85808fcccab24911d257d720550328a.png)

id < 5的时候，memory的自带的主键hash索引失效，优化器选择了我们刚刚创建的b+树索引。这时，返回的数据就已经有序了，后面当我们强制优化器不分析直接选取默认的哈希索引，返回的又恢复了插入的顺序。



memory为什么不常用呢？数据都存放在内存中，速度不应该飞快吗?

这要看你如何定义快，因为内存表的锁粒度当时在实现的时候就只有表级锁，并发性能不高。而且内存存储数据的特性会让它在持久化方面表现的不那么令人满意，具体的我们在下文叙述。



##### 内存表的锁

如上文所描述的那样，内存表的锁粒度只有表级锁，当一个seesion更新的时候，其他的seesion只能等待。

这个尚且好说，只要不是那种特意写的语句，一般而言，我们的更新速度不涉及磁盘io，都是在内存中，所以虽然对并发度的支持不友好，但是平常场景下的速度也不会很夸张。



##### 数据持久性的问题

内存表的数据是存储在内存中的，当数据库重启的时候，内存表的数据会被清空。

其实这个不算是问题，因为选择一张表当内存表，就已经预料到了这种情况。



![img](https://static001.geekbang.org/resource/image/5b/e9/5b910e4c0f1afa219aeecd1f291c95e9.jpg)

让 我们把视线转移到主备高可用上，如果slave重启，内存表数据被清空，假设该表名为t1,主表可以正常操作。

备库重启后同步主库发来的update t1的语句，会因为找不到具体的行而报错，导致主备同步终止。如果此时发生主备切换，客户端会感知到t1的数据被清空了。

mysql为了避免我们说的，因为重启而导致的主备数据不一致的问题，当数据库服务重启的时候，会在binlog中写一条delete 语句，把所有的内存表的数据全部清空。

只要有一个实例重启，所有实例上的内存表的数据都要划为0.



1. 如果数据量大，内存表的表级锁不一定表现的有innodb的行级锁来的更快
2. 如果数据量不大，那么innodb有缓存，基本都可以命中。



是不是内存表就一无是处了呢？

当然不是，我们之前说的join使用临时表优化是可以的。

1. 内存临时表不存在并发问题
2. 临时表本来就需要被清空，不存在持久化问题
3. 一个实例上创建不会影响别的实例。









