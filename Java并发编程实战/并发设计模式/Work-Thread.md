书接上回的分工TPM。

其实就是线程池的实现。



#### 线程池的设置

##### 别用无界队列

使用有界队列+拒绝策略。





##### 一个任务一个线程池

不要多个任务公用一个线程池，会饿死。

```java

//L1、L2阶段共用的线程池
ExecutorService es = Executors.
  newFixedThreadPool(2);
//L1阶段的闭锁    
CountDownLatch l1=new CountDownLatch(2);
for (int i=0; i<2; i++){
  System.out.println("L1");
  //执行L1阶段任务
  es.execute(()->{
    //L2阶段的闭锁 
    CountDownLatch l2=new CountDownLatch(2);
    //执行L2阶段子任务
    for (int j=0; j<2; j++){
      es.execute(()->{
        System.out.println("L2");
        l2.countDown();
      });
    }
    //等待L2阶段任务执行完
    l2.await();
    l1.countDown();
  });
}
//等着L1阶段任务执行完
l1.await();
System.out.println("end");
```

在第一个for循环的时候，线程池里的线程全部在等待，根本没有办法执行下面的l2。



#### 课后思考

> 小灰同学写了如下的代码，本义是异步地打印字符串“QQ”，请问他的实现是否有问题呢？

```java

ExecutorService pool = Executors
  .newSingleThreadExecutor();
pool.submit(() -> {
  try {
    String qq=pool.submit(()->"QQ").get();
    System.out.println(qq);
  } catch (Exception e) {
  }
});
```



```tcl
答： 把线程池数量改为2就好了。
这个乍看之下没有问题，但是这个线程池只有一个线程，当提交第一个submit的时候，执行Runnable任务的时候，已经把线程池的线程用完了，但是执行这个Runnable任务的时候，我们又需要一个新的线程去执行新的Callable任务，但是线程池中已经没有线程了。
```

