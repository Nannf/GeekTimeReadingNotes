```java
public void before() throws Exception{
        ExecutorService executorService = Executors.newFixedThreadPool(3);

        FutureTask<Integer> task1 = new FutureTask<>(
                () ->{
                    System.out.println("开始查询第一家厂商的数据，耗时2天");
                    sleep(2000);
                    System.out.println("查询第一家厂商的数据成功！");
                    return 234;
                }
        );

        FutureTask<Integer> task2 = new FutureTask<>(
                () ->{
                    System.out.println("开始查询第2家厂商的数据，耗时9天");
                    sleep(9000);
                    System.out.println("查询第2家厂商的数据成功！");
                    return 2534;
                }
        );

        FutureTask<Integer> task3 = new FutureTask<>(
                () ->{
                    System.out.println("开始查询第3家厂商的数据，耗时6天");
                    sleep(6000);
                    System.out.println("查询第3家厂商的数据成功！");
                    return 278;
                }
        );
        executorService.submit(task1);
        executorService.submit(task2);
        executorService.submit(task3);

        if (task1.get() != null) {
            System.out.println("第一");
        }
        if (task2.get() != null) {
            System.out.println("第2");
        }
        if (task3.get() != null) {
            System.out.println("第3");
        }

    }

```

- 没引入线程池和Future之前，串行操作
- 引入了之后，我们发现在入库的操作上还是串行的。

我们可以引入阻塞队列，用阻塞队列+线程池，把其改为异步执行。

```java

 public void before() throws Exception{
        ExecutorService executorService = Executors.newFixedThreadPool(6);
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(3);

        FutureTask<Integer> task1 = new FutureTask<>(
                () ->{
                    System.out.println("开始查询第一家厂商的数据，耗时2天");
                    sleep(2000);
                    System.out.println("查询第一家厂商的数据成功！");
                    return 234;
                }
        );

        FutureTask<Integer> task2 = new FutureTask<>(
                () ->{
                    System.out.println("开始查询第2家厂商的数据，耗时9天");
                    sleep(9000);
                    System.out.println("查询第2家厂商的数据成功！");
                    return 2534;
                }
        );

        FutureTask<Integer> task3 = new FutureTask<>(
                () ->{
                    System.out.println("开始查询第3家厂商的数据，耗时6天");
                    sleep(6000);
                    System.out.println("查询第3家厂商的数据成功！");
                    return 278;
                }
        );
        executorService.submit(task1);
        executorService.submit(task2);
        executorService.submit(task3);

        executorService.execute(() -> {
            try {
                queue.add(task1.get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        executorService.execute(() -> {
            try {
                queue.add(task2.get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.execute(() -> {
            try {
                queue.add(task3.get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        for (int i = 0; i< 3;i++) {
            System.out.println(queue.take().intValue());
        }
    }
```



#### CompletionService入场

```java
public void after() {
        ExecutorService executorService = Executors.newFixedThreadPool(6);
        CompletionService<Integer> completionService = new ExecutorCompletionService<>(executorService);

        completionService.submit(() -> {
            System.out.println("开始查询第3家厂商的数据，耗时6天");
            sleep(6000);
            System.out.println("查询第3家厂商的数据成功！");
            return 278;
        });
        completionService.submit(() -> {
            System.out.println("开始查询第2家厂商的数据，耗时9天");
            sleep(9000);
            System.out.println("查询第2家厂商的数据成功！");
            return 2534;
        });
        completionService.submit(() -> {
            System.out.println("开始查询第一家厂商的数据，耗时2天");
            sleep(2000);
            System.out.println("查询第一家厂商的数据成功！");
            return 234;
        });

        for (int i = 0; i< 3;i++) {
            try {
                System.out.println(completionService.take().get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

```

- CompletionService 内部也有一个阻塞队列
- 帮我们完成了异步操作。
- 对比before和after，我们发现引入后的代码可读性更好了。



其实我们发现最后的for循环那边，如果我们不用等待所有的全部执行完，而只需要等一个执行完就返回，在第一个值获取到的时候，之间break就可以了。

这就是Dubbo 的Forking Cluster的实现。



#### 课后思考

> 本章使用 CompletionService 实现了一个询价应用的核心功能，后来又有了新的需求，需要计算出最低报价并返回，下面的示例代码尝试实现这个需求，你看看是否存在问题呢

```java

// 创建线程池
ExecutorService executor = 
  Executors.newFixedThreadPool(3);
// 创建CompletionService
CompletionService<Integer> cs = new 
  ExecutorCompletionService<>(executor);
// 异步向电商S1询价
cs.submit(()->getPriceByS1());
// 异步向电商S2询价
cs.submit(()->getPriceByS2());
// 异步向电商S3询价
cs.submit(()->getPriceByS3());
// 将询价结果异步保存到数据库
// 并计算最低报价
AtomicReference<Integer> m =
  new AtomicReference<>(Integer.MAX_VALUE);
for (int i=0; i<3; i++) {
  executor.execute(()->{
    Integer r = null;
    try {
      r = cs.take().get();
    } catch (Exception e) {}
    save(r);
    m.set(Integer.min(m.get(), r));
  });
}
return m;
```



```tcl
答： 问就是有问题。
思考一： 入库的线程池和查询的线程池用一个合适吗？ 不用一个的原因是因为慢io会影响后续排队的任务执行，但是这个不会。不会的原因是，当我调用save的时候，上面的那个已经执行完了。
思考二： AtomicReference有必要吗？这个用了之后可以解决线程安全性吗？
这个似乎是不行了，因为这个不是原子操作，我们试想这样一个操作序列：
A 执行set的时候，此时m的值是 23， 自己的值是21.在执行min的时候，发生了线程切换。
B 开始执行，B的值是1，此时B把m的值设置成了1，此时再切换成A，m的值被设置成21，与实际的情况不符。
任何原子类的非原子操作不加锁，都是需要额外注意的地方。

另一个思考方向是： 因为是异步执行，所以会直接返回了。需要加一个CountDownLatch或者CyclicBarrier来保证三个异步线程全部执行完毕。


```

