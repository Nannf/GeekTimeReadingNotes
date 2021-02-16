#### 前言

sql语句的逻辑，以查询为例，就是我最终返回的都是一个结果集，但是有不同的查询语句，不同的sql语句之间的性能差异巨大。

我想本文作者应该是要去论述影响sql语句执行性能的一些因素，其中应该着重说明哪些操作是特别耗时的。



#### where条件使用函数

建表语句如下：

```mysql
mysql> CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

查询语句如下：

```mysql
mysql> select count(*) from tradelog where month(t_modified)=7;
```

这个我到略有耳闻，就是where条件不能使用条件查询，不然不走索引。

![img](https://static001.geekbang.org/resource/image/3e/86/3e30d9a5e67f711f5af2e2599e800286.png)

我们发现走索引快的原因是因为它是有序的，同层是有序的，我不会全部遍历，但是使用那些会导致索引有序性变化的函数时，就如本例的month函数。

因为我们只要主键和这个修改时间才有索引，我们走哪个呢？还是全表扫描。其实全表扫描是什么意思呢？它只表示了我扫描了表里的所有数据，我使用主键索引和修改时间索引都是可以完成这个功能的。所以全表扫描准确的说不能和这两个索引并列。

mysql优化器只是放弃使用树搜索功能，而不是放弃使用这个索引，其实还是会使用这个索引的，只是使用这个索引完成了全表扫描的功能。

当然本例中，我们这个函数改变了索引树的有序性，如果我不改变有序性呢，比如 where id + 1 = 1000; mysql 优化器是不管的，只要是函数，需要计算的，都放弃使用树搜索功能，而选择从索引中，评估出耗时较少的索引遍历扫描全表。



#### 参数类型转换

假设我们有如下的查询语句

```mysql
mysql> select * from tradelog where tradeid=110717;
```

表定义中tradeid是varchar，那么我们在查询的时候，是把后面的整形转换成了varchar，还是把tradeid强转成整形。

