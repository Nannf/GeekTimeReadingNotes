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





