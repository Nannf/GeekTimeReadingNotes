#### 前言

- grant 是赋用户权限的关键字，grant修改之后的权限会同步更新表和内存。
- 权限存在于内存中和表中，表中的权限是最新的，内存中的不一定是，二者差异的根本原因是使用了DML语句修改了表，而没有同步更新内存。
- flush privileges 是使用表中的权限更新内存中的权限





#### 如何创建用户

```mysql
create user 'ua'@'%' identified by 'pa';
```

- 这里的'ua'@'%'表示创建了名为ua的所有ip地址的对象
- 用户名加地址才能唯一表示一个用户
- 这个操作做了两件事情
  - 往磁盘上的**mysql.user**表中插入一行，因为没有指定权限，所以这个用户的所有权限都是N
  - 往内存中的**acl_users**数组中插入一个acl_user对象，对象的access字段值为0



#### 权限的范围

- 全局权限
- 数据库权限
- 表权限
- 列权限

范围依次降低



##### 全局权限

这个权限作用于整个mysql实例，故名全局权限。

```mysql
grant all privileges on *.* to 'ua'@'%' with grant option;
```

这个语句表示给用户'ua'@'%'赋予所有的操作权限。

体现在具体的操作上，包括：

1. mysql.user表的所有权限全部变为Y
2. acl_users中对应的用户对象，将access(权限位)全部置为1

需要说明的两点是：

- 我们在第一节的时候知道，这个赋权限的操作不会对一个已经存在的连接生效
- 如果执行完语句之后，有新的ua用户登录，那么权限就已经生效，此时无需flush操作



当然这个操作我们不应该执行，因为一个用户的权限太大了，会带来不必要的风险。

```mysql
revoke all privileges on *.* from 'ua'@'%';
```

我们可以使用上述的语句回收，做的事情跟grant相反、



##### 数据库权限

mysql实例可以有多个库，我们可以让一个用户拥有一个库的全部权限。

```mysql
grant all privileges on db1.* to 'ua'@'%' with grant option;
```

这个体现在实际的操作上：

- 在**mysql.db**表中插入一行记录，权限值全为Y
- 在内存中，找到**acl_dbs**数组，插入一个对象，权限位全为1

![img](https://static001.geekbang.org/resource/image/ae/c7/aea26807c8895961b666a5d96b081ac7.png)

- 全局权限修改之后，已经存在的连接不会感知。从sessionB的T4可以看出
- 数据库权限之后，已经存在的连接会感知，我们从sessionB的T6可以看出
- 特别的，如果一个连接是在移除权限前就在当前的db里，那么它的权限不会被grant语句影响，这个从sessionC的T6可以看出。

我们可以看出，数据库权限对已有的连接的影响要区分在移除权限前，当前的连接是否已经在db中，这个和全局不同，这个其实好理解，因为全局是最大的，所有的连接都在全局中，所以所有的存在的连接不会受影响。





##### 表权限和列权限

和全局权限和数据库权限类似的，

mysql.tables.priv、mysql.columns_priv分别存放着表权限和列权限

column_priv_hash存放着二者的内存中的权限。

```mysql

create table db1.t1(id int, a int);

grant all privileges on db1.t1 to 'ua'@'%' with grant option;
GRANT SELECT(id), INSERT (id,a) ON mydb.mytbl TO 'ua'@'%' with grant option;
```

这两个赋权限的操作也是同时修改了表和内存。是即时生效的。

因为表和列已经是最小的了，所以这个修改之后，所有的连接的权限全部被修改了。





#### flush Privileges使用场景

所有的grant语句都是修改内存和表的，只有dml语句认为的修改表的时候，造成内存和磁盘文件的不一致的时候，我们才需要使用。

![img](https://static001.geekbang.org/resource/image/90/ec/9031814361be42b7bc084ad2ab2aa3ec.png)

这个很简单，A在T5之前，B是使用内存来进行权限判断的，T5之后，内存中的权限被修改了。



![img](https://static001.geekbang.org/resource/image/dd/f1/dd625b6b4eb2dcbdaac73648a1af50f1.png)



T4失败是因为在表中找不到用户。

T5失败是因为内存中已经存在了用户。



