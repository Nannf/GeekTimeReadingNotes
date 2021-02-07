#### 前言

到了我第二喜欢的order by 环节

看之前有个问题，如果表里有一亿条数据，难道排序也是把所有的记录全处理吗？写完这个问题，我感觉我像个脑残，不然呢。



#### 排序的地方

##### 是内存还是磁盘？

如果内存够用的话，就在内存中排，不够的话就用磁盘归并排序。

##### 够不够用是相对于什么来说的？

这取决于排序需要用到的字段。

##### 排序用到哪些字段呢？

取决于我们最后返回的字段是否大于一个参数`max_length_for_sort_data`，这个会把要返回的字段(个人感觉，这边也要加上排序的字段)的定义长度加起来判断是否大于这个，如果大于这个参数，只会把排序字段和主键id放到排序内存中用来排序，否则会把要返回的字段和排序的所有字段全部加载进内存排序。

##### 排序字段要不是索引怎么办

那就是全表扫描，建议所有的排序字段最后都是索引。

##### 排序的步骤是什么样子的

我们假定排序的字段已经建了索引，且我们是单条件查询，我们首先会找到那个索引树，为了方便叙述，我们使用如下的例子来说明。

```mysql

CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```

```mysql

select city,name,age from t where city='杭州' order by name limit 1000  ;
```

![img](https://static001.geekbang.org/resource/image/53/3e/5334cca9118be14bde95ec94b02f0a3e.png)

1. 初始化sort_buffer 确定放入name、city、age三个字段。sort_buffer是mysql未每个线程分配的用来排序的内存。
2. 第一步查询city对应的索引树，找到杭州出现的第一个位置，记录下它的id
3. 到主键索引上去除id对应的name，city，age三个字段，放到sort_buffer中
4. 到city索引上接着去下一条记录判断是否满足条件，如果满足重复2、3步骤，知道遇到第一个city不是杭州的为止。
5. 对sort_buffer中的字段按照name排序
6. 返回给服务器端

其中第五步，排序的时候，因为sort_buffer的大小是确定的，是通过参数控制的，但是我们实际排序需要的大小是未知的，如果超过了给的的sort_buffer的大小，我们就会把本次排序的内容写到磁盘的文件上，进行外部归并排序。

我们都知道，使用磁盘进行外部排序的速度有点慢，

