LOCK-互斥

CONDITION-同步

#### 何为管程

《计算机操作系统》

> 在利用管程实现进程同步时，当某进程通过管程请求获得临界资源而未能满足时，管程便调用wait原语使该进程等待，并将其排在等待队列上。仅当另一个进程访问完成并释放该资源后，管程才又调用signal原语，唤醒等待队列中的队首进程。
> 但是，考虑这样一种情况：当一个进程调用了管程后，在管程中时被阻塞或挂起，直到阻塞或挂起的原因解除；在此期间，如果该进程不释放管程，则其它进程就无法进入管程，被迫长时间等待。
> 为了解决这个问题，引入条件变量condition。通常，一个进程被被阻塞或挂起的条件（原因）可有多个，因此在管程中设置了多个条件变量，对这些条件变量的访问智能在管程中进行。

我们是否可以简单的理解为，管程是一种对共享资源的访问机制，让其线程安全。

#### synchronized和lock的爱恨情仇

- synchronized是如何实现管程的？
  - synchronized通过在编译之后的字节码中加两个monitorenter和monitorexit保证了对临界区的访问最多只有一个线程。
- 其实在jdk1.5之前java只提供了synchronized关键字作为并发编程的手段，说明了这个关键词已经满足了并发编程的需要。
- 后续出现lock等一系列并发包的原因在哪呢？

##### synchronized性能问题？

后面通过优化，synchronized的性能已经和lock等相当了，这个可能不是主要的因素。

当有线程在等待synchronized锁的时候，线程会陷入阻塞，会死等，自然也不会释放自己持有的资源。

我们希望我们的线程有这样的功能

- 能够响应中断： 我持有资源A，想要申请资源B，但是申请不到，我得等，在我等待的过程中，我可以响应外界发给我的中断信号，这样我就可以知道，我是获取不到资源B了，我可以做一些清理工作，然后释放出我持有的资源A；
- 支持超时： 上一个是有人来告知我去放弃，我其实也不必一直再等，如果我等一段时间，她还是没来，我就直接做些清理工作然后放弃了。
- 非阻塞的获取锁： 我如果获取不到直接就放弃了，等都不等。

死等固然痴情，但是很大程度上会误人误己。

更灵活的等待是以lock为代表的一系列并发工具包的产生的内因。



##### lock等方法如何保证可见性

synchronized的可见性是有happens-before原则保证的。

Lock呢？

```java
class X {
  private final Lock rtl =
  new ReentrantLock();
  int value;
  public void addOne() {
    // 获取锁
    rtl.lock();  
    try {
      value+=1;
    } finally {
      // 保证锁能释放
      rtl.unlock();
    }
  }
}
```

线程一执行完addOne方法后，我们如何保证后续的线程一定看的到value值是最新的。

volatile语义。

我们知道volatile的写happens-before volatile变量的读。

ReentrantLock中有一个volatile修饰的变量state。

在lock的时候会进行state的读写。

在unlock的时候也会进行一波读写。

顺序性原则： state的第一次读写先于value的读写，第二次的state读写晚于value的读写

volatile的原则： 当我们的线程二开始尝试加锁的时候，会对state进行一次读写

传递性原则： value 的读写早于第二次的state读写，第二次的state读写早于线程二的第一次state读写。value的读写早于线程二的读写。



#### 可重入锁

同一个锁，一个已经获取的线程可以再次获取。我还没见过不可重入的锁。

```java

class X {
  private final Lock rtl =
  new ReentrantLock();
  int value;
  public int get() {
    // 获取锁
    rtl.lock();         ②
    try {
      return value;
    } finally {
      // 保证锁能释放
      rtl.unlock();
    }
  }
  public void addOne() {
    // 获取锁
    rtl.lock();  
    try {
      value = 1 + get(); ①
    } finally {
      // 保证锁能释放
      rtl.unlock();
    }
  }
}
```



#### 用锁的最佳实践

Doug Lea

> 永远只在更新对象的成员变量时加锁
>
> 永远只在访问可变的成员变量时加锁
>
> 永远不在调用其他对象的方法时加锁

第一个加锁原则是保证同时只有一个线程能修改共享变量；

第二个加锁原则是保证修改对后续的线程是可见的；、

第三个的加锁原则我极少用到，因为其他对象的方法对我们是黑盒，

- 不知道其他对象有没有内部锁而导致的死锁
- 不知道其他对象的方法有没有耗时操作而导致我的这个线程长时间占用锁



#### 课后思考

你已经知道 tryLock() 支持非阻塞方式获取锁，下面这段关于转账的程序就使用到了 tryLock()，你来看看，它是否存在死锁问题呢？	

```java

class Account {
  private int balance;
  private final Lock lock
          = new ReentrantLock();
  // 转账
  void transfer(Account tar, int amt){
    while (true) {
      if(this.lock.tryLock()) {
        try {
          if (tar.lock.tryLock()) {
            try {
              this.balance -= amt;
              tar.balance += amt;
            } finally {
              tar.lock.unlock();
            }
          }//if
        } finally {
          this.lock.unlock();
        }
      }//if
    }//while
  }//transfer
}
```



```text
答： 不会死锁，但是会死循环。问题就是不会出于等待状态，会直接返回false，导致两个账户在互相转账时出现。
```

