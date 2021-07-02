#### 演进之路

和线程池一脉相承，之前的线程池在执行的时候，调用的时**`execute`**方法，这个方法没有返回值。

但是我们有时候需要获取线程的执行结果，之前的execute方法就有点捉襟见肘了，基于此，我们引出了另一种提交任务到线程池的方法**`submit`**方法，这个方法是可以接收线程的运行结果的，运行结果被封装在**FutureTask**中。



#### submit

```java
// 提交Runnable任务
Future<?> 
  submit(Runnable task);
// 提交Callable任务
<T> Future<T> 
  submit(Callable<T> task);
// 提交Runnable任务及结果引用  
<T> Future<T> 
  submit(Runnable task, T result);
```

##### Future

- `cancel()` 取消任务的方法
- `isCancelled()` 判断任务是否已经被取消的方法
- `isDone()` 判断任务是否已经结束的方法
- `get()` 获取任务的执行结果，会阻塞调用线程，直到任务执行结束
- `get(timeout, unit)`超时获取任务的执行结果，会阻塞调用线程，直到任务执行结束



##### Future<?>  submit(Runnable task);

- 参数是Runnable, 这个没有返回值，所以Future仅仅能判断线程是否运行结束，其他什么都做不了

```java
 public static void main(String[] args) throws Exception{
        Callable<String> callable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("hello callable");
                return "nannf";
            }
        };
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        Future<?> future = executorService.submit(() ->{
            try {
                Thread.sleep(3000);
                System.out.println("i am submit runnable!");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        while (!future.isDone()) {
            System.out.println("i am not done");
            Thread.sleep(1000);
        }
        System.out.println("i am done");
        System.out.println(future.get());

    }
```

执行结果

```tex
i am not done
i am not done
i am not done
i am submit runnable!
i am done
null
```







##### <T> Future<T> submit(Callable<T> task);

```java
public class SubmitTest {

    public static void main(String[] args) throws Exception{
        Callable<String> callable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("hello callable");
                return "nannf";
            }
        };
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        Future<String> future = executorService.submit(callable);
        System.out.println(future.get());
    }
}
```

- 线程执行结果即是T
- Future中的T也是线程的执行结果



##### <T> Future<T>  submit(Runnable task, T result);

```java
public static void main(String[] args) throws Exception{
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        Result result = new Result();
        result.setName("kkk");
        Future<Result> future = executorService.submit(new Task(result),result);
        System.out.println(future.get().getName());
    }

    static class Result {
        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }

    static class Task implements Runnable {
        private Result result;

        public Task(Result result) {
            this.result = result;
        }

        @Override
        public void run() {
            result.setName("nannf");
            System.out.println("start task！");
        }
    }
```



#### FutureTask

- 实现了 Runnable 和 Future
- 既可以当参数提交给线程池，也可以接收返回的结果



```java
FutureTask(Callable<V> callable);
FutureTask(Runnable runnable, V result);
```

```java
    public static void main(String[] args) throws Exception{
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        FutureTask<Integer> futureTask = new FutureTask<>(()->{
            return 1+1;
        });
        executorService.submit(futureTask);
        System.out.println(futureTask.get());
    }
```

- FutureTask中的泛型参数也是最后的返回结果

```java
public static void main(String[] args) throws Exception{
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        Result result = new Result();
        result.setName("kkk");
        FutureTask<Result> future = new FutureTask<>(new Task(result),result);
        executorService.submit(future);
        System.out.println(future.get().getName());
    }

    static class Result {
        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }

    static class Task implements Runnable {
        private Result result;

        public Task(Result result) {
            this.result = result;
        }

        @Override
        public void run() {
            result.setName("nannf");
            System.out.println("start task！");
        }
    }
```





##### FutureTask可以直接被Thread执行

```java
 FutureTask<Integer> futureTask = new FutureTask<>(()->{
            return 9527;
        });
        Thread thread = new Thread(futureTask);
        thread.start();
        System.out.println(futureTask.get());
```



#### 实战： 烧水泡茶

![img](https://static001.geekbang.org/resource/image/86/ce/86193a2dba88dd15562118cce6d786ce.png)

```java
package thread;

import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;

/**
 * @author Nannf
 * @date 2021/7/1 14:06
 * @description 烧水泡茶
 * 这个其实是多线程之间的分工与同步之间的关系
 * 涉及到多个线程的步调一致，一组线程互相等待，考虑使用 CyclicBarrier，
 * 有两个工作之间还有先后关系。但是这两个工作与另外的几个工作之间是可以并行的。
 */
public class DrinkTea {
    ExecutorService callBack = Executors.newSingleThreadExecutor();

    ExecutorService executorService = Executors.newFixedThreadPool(4);

    private final CyclicBarrier cyclicBarrier = new CyclicBarrier(4, () -> {
        callBack.execute(() -> {
            System.out.println("所有的工作全部准备完毕，我要泡祁门红茶了！");
        });
    });

    public void process() {
        executorService.submit(() ->{
            try {
                System.out.println("我是洗茶壶程序，我要洗一秒钟");
                Thread.sleep(1000);
                System.out.println("我洗完茶壶了");
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() ->{
            try {
                System.out.println("我是洗茶茶杯程序，我要洗两秒钟");
                Thread.sleep(2000);
                System.out.println("我洗完茶杯了");
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() ->{
            try {
                System.out.println("我是拿茶叶程序，我要拿一秒钟");
                Thread.sleep(1000);
                System.out.println("我拿完茶叶了");
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() ->{
            try {
                FutureTask<Integer> futureTask = new FutureTask<>(()->{
                    System.out.println("我是洗水壶程序，我要洗一秒钟");
                    Thread.sleep(1000);
                    System.out.println("我洗完水壶了");
                    return 9527;
                });
                if (futureTask.isDone()) {
                    System.out.println("洗水壶程序完成了，给我返回了"+futureTask.get());
                    System.out.println("开始使用水壶烧开水，我要烧15秒钟");
                    Thread.sleep(15000);
                    System.out.println("水烧开了");
                }
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });
    }

    public static void main(String[] args) {
        new DrinkTea().process();
    }

}

```

- 初版的运行出来，发现水壶的那部分没执行。

修改如下

```java
package thread;

import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;

/**
 * @author Nannf
 * @date 2021/7/1 14:06
 * @description 烧水泡茶
 * 这个其实是多线程之间的分工与同步之间的关系
 * 涉及到多个线程的步调一致，一组线程互相等待，考虑使用 CyclicBarrier，
 * 有两个工作之间还有先后关系。但是这两个工作与另外的几个工作之间是可以并行的。
 */
public class DrinkTea {
    ExecutorService callBack = Executors.newSingleThreadExecutor();

    ExecutorService executorService = Executors.newFixedThreadPool(4);

    private final CyclicBarrier cyclicBarrier = new CyclicBarrier(4, () -> {
        callBack.execute(() -> {
            System.out.println("所有的工作全部准备完毕，我要泡祁门红茶了！");
        });
    });

    public void process() {
        executorService.submit(() ->{
            try {
                System.out.println("我是洗茶壶程序，我要洗一秒钟");
                Thread.sleep(1000);
                System.out.println("我洗完茶壶了");
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() ->{
            try {
                System.out.println("我是洗茶茶杯程序，我要洗两秒钟");
                Thread.sleep(2000);
                System.out.println("我洗完茶杯了");
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() ->{
            try {
                System.out.println("我是拿茶叶程序，我要拿一秒钟");
                Thread.sleep(1000);
                System.out.println("我拿完茶叶了");
                cyclicBarrier.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        });

        FutureTask<Integer> futureTask = new FutureTask<>(()->{
            System.out.println("我是洗水壶程序，我要洗一秒钟");
            Thread.sleep(1000);
            System.out.println("我洗完水壶了");
            return 9527;
        });
        executorService.submit(futureTask);
        try {
            if (futureTask.isDone()) {
                System.out.println("洗水壶程序完成了，给我返回了" + futureTask.get());
                System.out.println("开始使用水壶烧开水，我要烧15秒钟");
                Thread.sleep(15000);
                System.out.println("水烧开了");
                cyclicBarrier.await();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args) {
        new DrinkTea().process();
    }

}

```

发现卡死了，即futureTask.isDone() 一直没有返回。

改为下面这样就可以了

```java
package thread;

import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;

/**
 * @author Nannf
 * @date 2021/7/1 14:06
 * @description 烧水泡茶
 * 这个其实是多线程之间的分工与同步之间的关系
 * 涉及到多个线程的步调一致，一组线程互相等待，考虑使用 CyclicBarrier，
 * 有两个工作之间还有先后关系。但是这两个工作与另外的几个工作之间是可以并行的。
 */
public class DrinkTea {
    ExecutorService callBack = Executors.newSingleThreadExecutor();

    ExecutorService executorService = Executors.newFixedThreadPool(4);

    private final CyclicBarrier cyclicBarrier = new CyclicBarrier(4, () -> {
        callBack.execute(() -> {
            System.out.println("所有的工作全部准备完毕，我要泡祁门红茶了！");
        });
    });

    public void process() {
        executorService.submit(() -> {
            try {
                System.out.println("我是洗茶壶程序，我要洗一秒钟");
                Thread.sleep(1000);
                System.out.println("我洗完茶壶了");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() -> {
            try {
                System.out.println("我是洗茶茶杯程序，我要洗两秒钟");
                Thread.sleep(2000);
                System.out.println("我洗完茶杯了");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.submit(() -> {
            try {
                System.out.println("我是拿茶叶程序，我要拿一秒钟");
                Thread.sleep(1000);
                System.out.println("我拿完茶叶了");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        FutureTask<Integer> futureTask = new FutureTask<>(() -> {
            System.out.println("我是洗水壶程序，我要洗一秒钟");
            Thread.sleep(1000);
            System.out.println("我洗完水壶了");
            return 9527;
        });
        executorService.submit(futureTask);

        executorService.submit(() -> {
            try {
                System.out.println("洗水壶程序完成了，给我返回了" + futureTask.get());
                System.out.println("开始使用水壶烧开水，我要烧15秒钟");
                Thread.sleep(15000);
                System.out.println("水烧开了");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

    }

    public static void main(String[] args) {
        new DrinkTea().process();
    }

}

```

运行结果如下：

```tex
我是洗茶壶程序，我要洗一秒钟
我是洗茶茶杯程序，我要洗两秒钟
我是拿茶叶程序，我要拿一秒钟
我是洗水壶程序，我要洗一秒钟
我洗完茶壶了
我洗完水壶了
我拿完茶叶了
洗水壶程序完成了，给我返回了9527
开始使用水壶烧开水，我要烧5秒钟
我洗完茶杯了
水烧开了
所有的工作全部准备完毕，我要泡祁门红茶了！
```

但是我理解错了题意，这个只有一个人在做这些事情，我的考虑是有4个人同时在处理，也没说错吧，如果是一个人的话，这个人需要先洗水壶，然后烧开水，接着等待开水烧完毕，在等待的过程中，这个人要洗茶杯，洗茶壶，拿茶叶，然后等待水烧完毕之后，进行泡茶。

```java
package thread;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;

/**
 * @author Nannf
 * @date 2021/7/1 14:37
 * @description
 * 一个人做完的版本,
 * 这个人得先洗水壶，然后再用洗好的水壶烧开水，再等待水烧开的同时，我们依次洗茶杯，洗茶壶，拿茶叶，然后等水开，就可以泡茶了。
 * 这个是两个线程的处置过程，只有两条路都执行完成了，我们才能开始泡茶
 */
public class DrinkTeaOnePerson {

    ExecutorService executorService = Executors.newFixedThreadPool(2);

    public void process() {

        FutureTask<Integer> task1 = new FutureTask<>(() -> {
            System.out.println("我开始洗水壶，预计耗时1分钟");
            Thread.sleep(1000);
            System.out.println("我洗完水壶了，开始烧水，预计需要15分钟");
            Thread.sleep(15000);
            System.out.println("我水烧开了");
            return 9527;
        });
        executorService.submit(task1);

        FutureTask<Integer> task2 = new FutureTask<>(() -> {
            System.out.println("我开始洗茶壶，预计耗时1分钟");
            Thread.sleep(1000);
            System.out.println("我洗完茶壶了，开始洗茶杯，预计需要2分钟");
            Thread.sleep(2000);
            System.out.println("我洗完茶杯了，开始拿茶叶，预计耗时1分钟");
            Thread.sleep(1000);
            System.out.println("我拿完茶叶了");
            return 9527;
        });
        executorService.submit(task2);

        try {
            if (task1.get() == 9527 && task2.get() == 9527) {
                System.out.println("所有的工作全部准备完毕，可以泡茶了！");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args) {
        new DrinkTeaOnePerson().process();
    }
}

```

- 这个虽然可以了，但是还是没有模拟到一个人的流程性
  - 具体的体现在哪呢？就是我一个人在烧水的时候，就应该去洗茶壶茶杯，拿茶叶了，因为烧水的时间我们知道一定比拿茶叶这些耗时久，而且只有两个线程在等待，所以我的做法是更通用的。





#### 课后思考

> 不久前听说小明要做一个询价应用，这个应用需要从三个电商询价，然后保存在自己的数据库里。核心示例代码如下所示，由于是串行的，所以性能很慢，你来试着优化一下吧。

```java

// 向电商S1询价，并保存
r1 = getPriceByS1();
save(r1);
// 向电商S2询价，并保存
r2 = getPriceByS2();
save(r2);
// 向电商S3询价，并保存
r3 = getPriceByS3();
save(r3);
```

```tex
答： 三个之间没有依赖关系，是完全的并行，直接使用线程池，并行跑即可。
```

