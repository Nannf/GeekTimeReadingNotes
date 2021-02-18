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

假设我们执行 `select "10" > 9`;

如果返回的结果是1，表示是字符串转整形，否则是整形转字符串。

![img](https://static001.geekbang.org/resource/image/2b/14/2b67fc38f1651e2622fe21d49950b214.png)

我们此时再来看之前的查询语句，这就等于

```mysql
mysql> select * from tradelog where CAST(tradid AS signed int)=110717;
```

根据我们上面的分析，这个是执行了函数，而无论这个函数有没有改变索引树的顺序，这个都会走全表扫描。



#### 字符编码转换

这个的背景是mysql有两个字符编码，分别是utf8 和 utf8mb4，后者是前者的超集，如果连表时，两个表的字符分别是这两个，那么就需要转换，这个转换规则类似于java中的int和 double，当我们定义

```java
int i = 10;
double d = 9d;
if (i>d) {
    ...
}
```

当我们执行i>d这个比较的时候，会自动把i转换为double，类似的，当utf8和utf8mb4比较的时候，会自动把utf8转换为utf8mb4

`select * from t1 left join t2 on t1.id=t2.tID where t1.a = t2.b;`

我们上面的分析知道，倘若t1是utf8编码，t2是utf8mb4编码，那where语句就会变成 `CONVERT(t1.a USING utf8mb4)=t2.b`，倘若我们在t1.a上使用了索引，那这个索引是不会生效的。

解决方案呢？

1. 修改查询语句是 where t2.b = t1.a;
2. 修改t1的编码。
3. 修改查询语句是 where t1.a = convert(t2.b using utf8)





#### 综述

通过以上的分析我们可以看出，凡是引起索引失效的，都是在where语句的左侧使用了函数，无论是显式的，还是隐式的。这个我们只有explain 才能知道，有时候。

而造成索引失效的原因，是因为优化器在选择索引的时候，会有个是否使用树查询，还是全表搜索，会简单的根据是否使用函数来选择，一般而言，函数会导致索引树的有序性遭到破坏，而使索引失去了快速检索功能。mysql优化器的开发人员可能觉得要识别一个函数有没有造成有序性的破坏比较复杂，所以就简单的认定只要使用就默认认定破坏了有序性，而放弃使用索引提供的树检索功能。



