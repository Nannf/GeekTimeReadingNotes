- 读多写少的场景
- ReadWriteLock的优化版
- since 1.8



#### 它做了什么改动让性能更快

- ReadWriteLock在传统的锁的语义上引出了读锁和写锁的概念，并提出读锁是不存在线程安全问题的
- 读锁之间可以并行，但是读锁和写锁之间不能并行
- StampedLock就是针对这个地方做的优化，优化的前提是，不是每次的写操作都会影响到我读的部分
- 但是在StampedLock中，我们依然只允许最多只有一个线程在写。



#### 如何实现

- 正如这个锁的名字 stamp,标记的含义
- 这个stamp可以理解为程序关于共享变量维护的一个版本的概念，乐观读的时候，会记录关于我开启乐观读的瞬间的一个共享变量的快照，当我需要使用共享变量的时候，我需要判断目前共享变量的快照版本和我生成快照的那一刹那是否一致(此时就是当前读)，如果一致，表明从我生成快照到我要使用期间没有程序修改过。否则就需要使用当前读来重新获取。



#### 注意事项

- 不支持重入
- 悲观读和写都不支持条件变量
- 不要调用interrupt中断





#### 课后思考

> StampedLock 支持锁的降级（通过 tryConvertToReadLock() 方法实现）和升级（通过 tryConvertToWriteLock() 方法实现），但是建议你要慎重使用。下面的代码也源自 Java 的官方示例，我仅仅做了一点修改，隐藏了一个 Bug，你来看看 Bug 出在哪里吧。

```java

private double x, y;
final StampedLock sl = new StampedLock();
// 存在问题的方法
void moveIfAtOrigin(double newX, double newY){
 long stamp = sl.readLock();
 try {
  while(x == 0.0 && y == 0.0){
    long ws = sl.tryConvertToWriteLock(stamp);
    if (ws != 0L) {
      x = newX;
      y = newY;
      break;
    } else {
      sl.unlockRead(stamp);
      stamp = sl.writeLock();
    }
  }
 } finally {
  sl.unlock(stamp);
}
```



```tex
答： 实际运行下来代码会在最下面的finally报IllegalMonitorStateException。
对比了官网的示例，我们发现应该在if判断之后，把生成的ws参数赋值给stamp
```

