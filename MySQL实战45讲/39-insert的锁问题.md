#### 概览

insert语句如果有自增主键的话，会获取下自增主键锁，这个获取锁的很快，如果没有自增主键的话，会获取一个插入意向锁，如果有那种表级锁再运行，会等待，但是其本身不会占用多少锁资源，所以作者说的这个其实是一些批量插入的场景下的insert的持有锁的情形。



#### insert... select 

书接上文，上文我们在说自增主键不是连续问题的时候，提到了这个场景，作者的行文构思真是巧妙，等我全部读完之后，试图理解一下作者的行文思路，这应该就是一条线。

为了叙述方便，我们建表如下：

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);

create table t2 like t
```



现有问题如下： 在RR级别下，binlog_format=statement 的情况下，执行

```mysql
insert into t2(c,d) select c,d from t;
```

为什么会对表t的所有行和间隙加锁？



分析一波：

binlog格式，说明跟主备同步有关，而且statement是没办法知道具体的插入字段的。问题可能出在这边。

我们来反推一波，假设没有锁，在什么样的场景下会有问题。

没有锁，意味着我的t可以修改，可以新增，可以删除。这也就意味着，如果是binlog的话，如果是先执行修改t的相关binlog，那么会出现主备不一致。

大致正确。



#### insert 循环写入

```mysql
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

这个语句的原意是找到c最大的那个，然后+1，插入一条新数据。

根据执行计划我们发现其中使用了临时表，之所以不是一边读一边插入，可能是为了避免在读取数据的过程中读取到我们刚刚插入的数据，这就导致了与原意不符合。



#### insert唯一键冲突

![img](https://static001.geekbang.org/resource/image/83/ca/83fb2d877932941b230d6b5be8cca6ca.png)

隔离级别：RR；

**这个就是mysql的机制，当因为唯一键冲突插入失败时，不仅打印错误的信息，还会在唯一键索引上，新增一个next-key lock.**

**(5,10] next-key 读锁。**

![img](https://static001.geekbang.org/resource/image/63/2d/63658eb26e7a03b49f123fceed94cd2d.png)

1. A插入了数据，在索引c上加入了next-key锁，但是c是唯一键，退化成行锁。
2. B和C都因为插入的时候需要获取写锁，但是因为A持有锁，获取不到，而进入等待状态，此时两个都是持有读锁
3. A回滚后，锁释放，B和C都需要等待对方释放读锁，而进入死锁状态。

不清楚的地方有：

1. A在insert的时候为什么会加锁。我们回顾了番外篇二，锁的相关内容，插入的时候首先要获取意向锁，判断有没有表级锁存在，本例中是没有的。此时A拿到了意向锁，所以后续的B和C会被阻塞，因为他们在插入的时候发现有事务持有了意向锁。意向锁会影响并发的，如果我们要插入的间隙不同，没必要锁我的。所以在RR级别下，有了插入意向锁，B和C来的时候，发现A已经占了5那个间隙的意向锁，导致B和C进入了等待状态。
2. A回滚之后，B和C都需要获取记录锁X，但是B和C都持有S锁，大家都不会释放，而造成了死锁。



作者的叙述我大致看懂了一点，插入时，获取的时next-key lock ，next_key lock 由gap lock + record lock,A持有了record lock, B和C持有了gap lock 再等待record lock.

当A释放锁资源之后，B和C都需要获取插入意向锁，但是这个插入意向锁被gap锁给阻塞了。



#### insert into … on duplicate key update

```mysql
insert into t values(11,10,10) on duplicate key update d=100; 
```

这个语句会插入（5，10]的next-key lock

如果我们插入的数据和多个索引起冲突了，我们会找到第一个起冲突的记录然后更新。

```mysql
## 表里已经有(1,1,1)(2,2,2)两行
insert into t values(2,1,10) on duplicate key update d=100; 
```

主键id先判断，其他的不确定，所以这里可以判定修改的是主键id=2冲突的那行。

![img](https://static001.geekbang.org/resource/image/5f/02/5f384d6671c87a60e1ec7e490447d702.png)

2 rows affected 并不表示修改了两行。

因为这个语句表示 insert 和 update 都认为是成功的。



#### 小结

- insert ... select 常用做表间数据复制，会对select 扫描到的记录加next-key lock
- insert 和 select 都来自一张表的话，可能会造成循环写入，需要引入临时表。
- insert 如何造成了唯一键冲突，会再冲突唯一值上加共享的next-key lock。
