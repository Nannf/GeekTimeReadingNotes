#### 前言

狗日的order by 会影响加锁规则，我们单开一章来专门讨论其对加锁规则的影响。



我们讨论的前提有：

1. innodb
2. rr

我们建表如下：

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```





#### 场景一

```mysql
事务A：
begin;
select * from t where c>1 and c <=5 for update;

事务B：
update t set d = 9527 where c = 0
```

事务B可以修改成功。

非唯一索引的范围查找我们会怎么做？

优化器识别到c=5，先用等值查询，找到了第一条c=5的记录，此时加锁(0,5],然后继续遍历下一条，本来应该加(5,10]的锁，但是由于优化二，退化成间隙锁，所以此时的加锁范围是(0,5] （5，10）-->（0,10）;

然后使用范围查找 1<c<5，发现没找到记录，但是找到了一个间隙，就是(0,5)，锁住了这个间隙；

综上，我们最终的锁是 (0,10).

 

但是我们发现

下面两条语句

```mysql
update t set d = 9527 where c = 10;
insert into t values(8,8,8)
```

都会陷入锁等待状态，也就是我们的锁分析漏了c=10这一行

我不会了。





