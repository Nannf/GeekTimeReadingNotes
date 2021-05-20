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

