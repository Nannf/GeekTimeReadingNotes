#### 管程

> 在利用管程实现进程同步时，当某进程通过管程请求获得临界资源而未能满足时，管程便调用wait原语使该进程等待，并将其排在等待队列上。仅当另一个进程访问完成并释放该资源后，管程才又调用signal原语，唤醒等待队列中的队首进程。
> 但是，考虑这样一种情况：当一个进程调用了管程后，在管程中时被阻塞或挂起，直到阻塞或挂起的原因解除；在此期间，如果该进程不释放管程，则其它进程就无法进入管程，被迫长时间等待。
> 为了解决这个问题，引入条件变量condition。通常，一个进程被被阻塞或挂起的条件（原因）可有多个，因此在管程中设置了多个条件变量，对这些条件变量的访问智能在管程中进行。

java中实现的Condition对应的就是管程中的condition

synchronized中有条件变量吗？有的但是有且仅有一个。

就是所有的线程无论因为什么原因阻塞了，我们都只能在一个队列里等待。



#### 管程关键字

##### synchronized

- wait
- notify
- notifyAll



##### lock&condition

- await
- signal
- signalAll

不能混用。



#### 阻塞队列的实现

是否有空间去添加元素；

是否有元素可以被获取；

当一个线程需要获取元素，但是此时队列是空的，需要等待一个条件，条件的内容就是队列不空，当我们向队列添加新的元素时，队列不空的条件就满足了，所有的等待队列不空这个条件的线程全都会被唤醒。

有这样一种情况，就是我有十个线程都在等待获取数据，但是只有一个线程在写。一个线程写完之后，唤醒了所有的等待队列中有数据的线程，这时候会出现空指针异常吗？

其实是不会的，因为被唤醒之后，因为这些线程在挂起之前就已经释放了临界区的锁，所以他们就算全部被唤醒了，还是会到等待的队列中，只有一个线程可以被调度成功。

#### 同步与异步

##### dubbo

```java
DemoService service = 初始化部分省略
String message = 
  service.sayHello("dubbo");
System.out.println(message);
```

```java

public class DubboInvoker{
  Result doInvoke(Invocation inv){
    // 下面这行就是源码中108行
    // 为了便于展示，做了修改
    return currentClient 
      .request(inv, timeout)
      .get();
  }
}
```

```java

// 创建锁与条件变量
private final Lock lock 
    = new ReentrantLock();
private final Condition done 
    = lock.newCondition();

// 调用方通过该方法等待结果
Object get(int timeout){
  long start = System.nanoTime();
  lock.lock();
  try {
  while (!isDone()) {
    done.await(timeout);
      long cur=System.nanoTime();
    if (isDone() || 
          cur-start > timeout){
      break;
    }
  }
  } finally {
  lock.unlock();
  }
  if (!isDone()) {
  throw new TimeoutException();
  }
  return returnFromResponse();
}
// RPC结果是否已经返回
boolean isDone() {
  return response != null;
}
// RPC结果返回时调用该方法   
private void doReceived(Response res) {
  lock.lock();
  try {
    response = res;
    if (done != null) {
      done.signal();
    }
  } finally {
    lock.unlock();
  }
}
```

本来远程调用是异步的，但是dubbo使用了lock+condition，一直在等待远程调用的返回。给调用方的感觉就是同步的。



#### 课后思考

> DefaultFuture 里面唤醒等待的线程，用的是 signal()，而不是 signalAll()，你来分析一下，这样做是否合理呢？



```tet
答: 这个方法是request的，request是每个请求一个的，所以需要唤醒的线程也只有一个。

看了下面的分析之后，我们发现这个已经被改成signalAll了，但是我仍然没明白
```





