**生命周期中各个节点的状态转换机制**



线程生命周期的各个状态在操作系统层面和jvm实现其实不是一一对应。就如四层tcp协议和OSI七层协议。

考虑到线程最初是os提出的概念，我们先介绍os的线程生命周期。

- 新建
- 等待执行
- 执行中
- 阻塞
- 结束

新建和等待执行的区别是，新建时还没有告诉cpu可以来调度，等待执行其实就是在等待cpu调度了。

执行中是线程已经获得了cpu的运行时间。（其实新建状态是编程语言特有的，os看来线程从被新建出来的那一刻开始就已经可以接受cpu的调度了，具体到编程语言，我们以java为例，线程创建完之后，必须调用start()方法才能变成一个os页面的新建）

阻塞是因为调用一些阻塞操作让线程没办法继续执行下去，而让出cpu执行时间，以及被cpu调度机会的一个状态。

如果让线程阻塞的条件不存在了，线程就会从阻塞态迁移到等待执行状态，等待cpu的重新调度。

当线程执行过程中有未经处理的异常，或者线程需要执行的工作全部执行完成，就会转到结束状态。

至此，线程的生命之旅就结束了。



#### Java中线程的生命周期

- NEW(新建)
- RUNNABLE（可执行）
- BLOCKED（阻塞）
- WAITING（无时限等待）
- TIMED_WAITING（有时限等待）
- TERMINATED（终止）

其中 

os中的等待执行和执行中，对应java的RUNNABLE

- 这一步而言，是因为java把线程的调度给了os，所以这个线程只要是有执行条件，在jvm而言都是一样的。

os中的阻塞对应java中的BLOCKED、WAITING、TIMED_WAITING

- 这种jvm做了细化
- 只要是这三种状态，线程都没有cpu的调度权
- 做细化的依据是都是阻塞，但是阻塞的原因并不相同，为了方便知道线程是基于哪种方式而阻塞，方便分析问题，jvm才有了这种划分。

new 和 terminated是一一对应的。



#### jvm中的阻塞原因

##### BLOCKED

- 当且仅当线程尝试获取synchronized修饰的方法的时候
- 调用一些阻塞方法不算阻塞，等待资源而陷入的阻塞，在os层面是休眠，但是在jvm看来是runnable



##### WAITING

- wait
- join
- LockSupport.park()



##### TIMED_WAITING

- Thread.sleep(time)
- wait(time)
- join(time)
- LockSupport.parkNanos(Object blocker, long deadline)
- LockSupport.parkUntil(long deadline)





#### NEW

- 继承Thread
- 实现Runnable
- 重写run()方法

调用start方法之后由NEW->RUNNABLE



#### TERMINATED

##### stop  & interrupt

- stop已经废弃

  - 废弃的原因是因为它会立马释放所有持有的锁，这就导致了被锁所保护的变量可能出于不一致的情况（因为临界区的操作本来就是要是原子的，如果被中止会导致一些无法预料的中间状态的产生）；
  - 放弃所有未执行的代码，包括catch和finally，包括打开的文件句柄和数据库句柄。

- interrupt会告诉线程，你可以终止了，被终止的线程会抛出异常InterruptedException，所以一般线程需要捕获这个异常，之前我不知道这个是干啥的，现在知道捕获这个异常之后需要对临界区的共享变量的处置需要保证在一个一致的状态。

  - 只有当线程出于WAITING 或者 TIMED_WAITING的时候，被调用interrupt方法才会抛那个异常
  - 如果线程再RUNNABLE状态，此时别的线程调用interrupt方法，线程鸟都不鸟你，除非线程自己主动检测

  







#### 问题：

synchronized修饰的方法和可重入锁，这些是一个概念吗？

- 我问这个问题时，如果我的线程再等待可重入锁，那么时BLOCKED状态吗
  - 我写了下面的代码进行了验证，使用了JConsole进行了分析，这两个线程的状态，一个时WAITING,一个是TIMED_WAITING,说明只有synchronized方法才会使线程进入BLOCKED状态，其他的锁都是使用LockSupport.park()来实现的，只会处于WAITING状态
- 这也给这个问题一个解答，两个是不一样的，synchronized是jvm提供的关键字。



```java
package thread;

import java.util.concurrent.locks.ReentrantLock;

/**
 * @author Nannf
 * @date 2021/6/23 15:31
 * @description 本代码的目标时检测是不是只有synchronized框起来的线程才是BLOCKED
 */
public class BlockedTest {

    private final static ReentrantLock lock = new ReentrantLock();


    public static void main(String[] args) {
        new Thread(() -> lockTest(),"nn").start();
        new Thread(()->lockTest2(),"ff").start();
    }



    public static void lockTest2() {
        try {
            lock.lock();
            Thread.sleep(100000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }
        finally {
            lock.unlock();
        }
    }

    public static void lockTest() {
        try {
            lock.lock();
            Thread.sleep(100000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }
            finally {
            lock.unlock();
        }
    }

}

```





#### 课后思考

下面代码的本意是当前线程被中断之后，退出while(true)，你觉得这段代码是否正确呢？

```java
Thread th = Thread.currentThread();
while(true) {
  if(th.isInterrupted()) {
    break;
  }
  // 省略业务代码无数
  try {
    Thread.sleep(100);
  }catch (InterruptedException e){
    e.printStackTrace();
  }
}
```



```tex
答： 实际跑下来发现取决于中断操作是在sleep之前还是之后，如果在线程sleep之前，是可以终止的，否则一旦线程进入了sleep方法，检测就无效了。
看目前的状态，当线程的中断异常被捕获之后，线程的中断标识同样的被抹除了，所以需要重新设定一下中断标识。

```

