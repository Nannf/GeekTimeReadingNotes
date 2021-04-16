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

这个我再读20章的时候，发现这个虽然是先使用等值查询先找的5，然后往后找的时候找到10，但是这里没办法应用优化二因为这个不是等值查询，所以没退化。

(0,10]

#### 场景二

```mysql
事务A：
begin;
select * from t where c>1 and c <=5 order by c desc for update;

事务B：
update t set d = 9527 where c = 0；

事务C
insert into t values(-1,-1,-1)
```

事务B不出意外的被锁住了；

事务C也被锁住了。

这个正如我们所料，我们执行where语句锁住了(0,10],执行order by desc 遍历到了到了0，加了n锁，此时锁的范围变成了

(-无穷，10].



##### 场景三

```mysql
事务A：
begin;
select * from t where c in(5,10) order by c desc for update;

事务B：
update t set d = 9527 where c = 0；

事务C
insert into t values(-1,-1,-1);

事务D
insert into t values(4,4,4)
```

我们发现事务B和C可以执行，但是事务D会被锁住。

他娘的，这个锁的范围和不加order by是一样的。

我们来大胆的猜测，这个in和上面的范围有区别吗？in相当于多个or的等值查询，范围查询不是。

in是一种特殊的等值查询；



##### 场景四

```mysql
事务A：
begin;
select * from t where c in(5,10)  for update;

事务B：
update t set d = 9527 where c = 0；

事务C
insert into t values(-1,-1,-1);

事务D
insert into t values(4,4,4)
```

效果和上面的场景三是一样的。

这个order by 似乎没啥用。

我们知道mysql 中 in 和多次or的等值查询的区别就是 in会进行排序。

这个结果集天然就是有序的，加不加order by其实对语句执行没有影响。



为什么范围查找的order by就要去查找不满足条件的下一条记录，而等值的不会。

这个我觉得还是扫描到了，但是就在这个加锁优化二原则的理解

我们引用优化二的原文

> 索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。

其实我们发现这个默认是往右遍历的，当我们发现下一个值不满足的时候，我们是不会干扰到下一个值以及以后的值的。

如果这样理解的话我们就能轻松的理解order by 为什么不锁0了，所以它不会锁0，也不会锁0之前的。





