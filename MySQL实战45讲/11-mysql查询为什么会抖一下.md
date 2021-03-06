#### 前言

1.何谓“**抖**”

- 一条sql语句，正常执行的时候特别快，但是有时候会变得非常慢，这种慢是随机出现的，而且没办法复现。



#### 什么语句会变慢？

我们用常用的增删改查为例来逐个分析：

1. 增： 增加做了什么操作，写redolog,会写内存吗？我们来分析下，redolog是基于物理页的修改，如果插入一条数据时，是要计算出这个新增的数据应该插到那个数据页里，是否会具体插入呢？但是我们知道插入如果开启事务的话，事务是不会写磁盘的，是直接写redolog，加入插入100w条数据，每条都随机写磁盘，那基本没的完，我猜测，插入时并不写磁盘，而只是在内存中和redolog中记录这个插入的数据，等空闲时写入。

其他的三种类型我们等文章结束的时候再来分析。

> 分析完之后我们发现，无论是查询还是修改，都可能导致内存中进行刷脏页的操作，或者把redolog落盘的操作；以及服务后的自动的落盘操作，所以理论上所有的操作都会出现这种间歇性的性能下降。



我们以更新为例，当我们更新一条数据的时候，都涉及了哪些存储呢？

之前在分析一条更新语句是如何执行的时候，我们知道，数据是先写redolog，空闲时修改到磁盘上的，但是有个问题，就是假设我修改之后就读这条数据，那难道还要把这个数据页从磁盘上加载进内存，然后和redolog进行一次运算，才返回结果吗？

显然比较慢，这里还设计到changebuffer，其实这个会写到changebuffer中，当然如果当前的数据页在内存中，这个会直接修改数据页上的内容，并把修改同步到redolog中，但不会立即刷新到磁盘上。

这就导致了内存中的数据页上的数据和磁盘上的数据页的数据不一致的情况，这种不一致的页，我们称为**脏页**。

一直的页，我们称之为**干净页**

![img](https://static001.geekbang.org/resource/image/34/da/349cfab9e4f5d2a75e07b2132a301fda.jpeg)

因为我们一般而言，一个修改操作，最多只涉及一次磁盘文件的读取，一次顺序磁盘的写入，是不怎么耗时的。

但是比如redolog写满了，或者内存用完了，导致我们必须要把一部分数据写入磁盘的时候，这时候就会涉及到多个磁盘的随机访问，这个体现出来的就是那次的修改用的时间比平常慢的多。

我们来具体分析下这些场景：

1. redolog写满了
   - redolog是磁盘上一块固定大小的顺序空间，它是启动时由参数控制的，如果redolog的空间被用完了，那么此时mysql会暂停所有的都库表的修改，会把redolog记录的内容抽取一块出来同步到磁盘上。
   - redolog不是一直到用完才会去写，在服务器识别到不忙的时候，也是会同步的，这种场景常见于服务一直在响应请求，而且redolog的初始空间设定的不是很大。
   - 一旦出现这种情况，其实mysql服务整个的修改相关的需要写redolog的全部都会被暂停。
2. 内存被占满了
   - 内存一开始是空的，后来我们基于这样一个原则，如果这个数据页在现在被访问，那么他在未来也更有可能被再次访问，我们基于此，把一些此磁盘上的数据页放到内存中，如果查询的话，可以直接从内存中读取，而不用去读磁盘
   - 内存空间是有限的，能加载的数据页也是有限的，当内存用完的时候，我们此时需要读取新的数据页的时候，就需要按照一定的规则，比如最近最久未使用的算法，淘汰一些数据页，如果数据页和其从磁盘上加载的时候不一样的话，我们需要把数据页的修改写回磁盘上，如果你本次的操作要加载的数据页很多，多到了我们需要把很多内存中的数据页写回磁盘，这就会导致本次操作的变慢。
   - 这里顺便提一下，假如我们在写磁盘的过程中数据库服务意外宕机了，会有影响吗？简单的说，我们有redolog，那些已经修改的不会丢失，但是，如果我恢复到一半宕机了，这个就涉及到redolog恢复时的处理逻辑了，这个我们暂且认为它是不会造成丢失的。
3. 空闲的时候
   - 当服务器通过一些参数来识别出目前数据库处于空闲的情况，就会刷脏页，写redolog。
4. 数据库正常关闭的时候
   - 正常关闭的时候会刷脏页
   - 正常关闭的时候会同步redolog吗
   - 会同步的，而且每次启动的时候也会去判断有没有没有同步的redolog，这是为了处理那些意外的关闭



通过我们上面的分析我们知道，当出现如下两种情况的时候：

1. redolog写满了，更新会全部阻塞
2. 一次操作需要淘汰的脏页太多

都会导致明显的性能问题。

我们需要一些机制去规避这些问题。

无论这两个是哪个导致了，我们都涉及到随机磁盘i/o的写操作。InnoDB进行随机写的时候，是针对数据页的，而数据页是固定大小的，我们知道了mysql服务安装机器的i/o能力之后，才好去控制我同时最多能刷多少个数据页。

这边又会有个问题，就是我们知道了机器的最大磁盘i/o能力，但是我们的磁盘不能仅用来刷脏页，需要一个算法来控制，最终达到的效果就是我们根据需要刷新的磁盘的文件多少，来动态的规划同时使用多少的磁盘能力，当redolog或者内存中脏页比例已经接近百分之百了，这时候就应该全速刷了。

大致上的算法跟上述的差不多，就是内存种脏页有个比例，redolog有个使用的比例，通过这两个数据算出来我们最终需要的刷盘速度。

> 到这边我们就知道了，无论是我们执行的语句导致了同步磁盘动作的执行，还是mysql自己计算的同步磁盘都可能导致我们的执行时间变长。

8.0之前的版本中，如果刷脏页的时候，这个脏的数据页相邻的数据页也是脏页的话，会一直刷下去。

这个相邻我们如何定义呢？我猜测是内存上的相邻，内存上的存储位置的相邻。





问： 一个内存配置为 128GB、innodb_io_capacity 设置为 20000 的大规格实例，正常会建议你将 redo log 设置成 4 个 1GB 的文件。但如果你在配置的时候不慎将 redo log 设置成了 1 个 100M 的文件，会发生什么情况呢？又为什么会出现这样的情况呢？



答： 刷盘的速度是按照redolog的使用比例和脏页的比例来算的，因为redolog的空间设置的比较少，导致磁盘会一直全负荷的在刷redolog。

内存大相对而言，能加载的数据页多，当对数据页修改的时候，会记录redolog，redolog会很快满了，redolog一满，会导致所有的更新全部暂停，而且会把内存中的脏页页给刷了。

导致的是内存大没用，因为redolog很快就满了，磁盘一直在很高的使用。

**这边描述的不准确的是，因为redolog很快就满了而且数据量不大，导致了磁盘的压力不大，但是redolog满了之后，会导致所有的更新操作全部阻塞，mysql的性能间歇性下降。**

