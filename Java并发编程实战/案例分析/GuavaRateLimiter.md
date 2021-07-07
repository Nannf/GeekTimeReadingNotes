高并发场景下的限流场景

- 因为我们的资源有限，并不能无限制的处置请求，比如我们一次性只能处理两个任务，否则我们的系统就会耗尽资源，导致程序无法响应

#### 初始RateLimiter

```java
package thread.test;

import com.revinate.guava.util.concurrent.RateLimiter;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author Nannf
 * @date 2021/7/6 10:17
 * @description
 */
public class RateLimiterTest {

    public static void main(String[] args) {
        RateLimiter rateLimiter = RateLimiter.create(2.0d);
        ExecutorService es = Executors.newFixedThreadPool(1);

        final long[] prevT = {System.nanoTime()};

        for (int i = 0; i < 20; i++) {
            rateLimiter.acquire();
            es.execute(() -> {
                long cur = System.nanoTime();
                System.out.println((cur- prevT[0])/1000_000);
                prevT[0] = cur;
            });
        }

    }
}

```



#### 令牌桶算法

##### 综述

- 令牌以固定的速度加到令牌桶中，比如我一秒可以处理4个数据，那么我的令牌就每1/4s放到令牌桶中
- 令牌桶是有容量的，假设为b，如果令牌桶已满，新生成的令牌会被丢弃
- 线程只有获取了令牌才能通过限流器



限流器为什么还要b呢？因为我们要有一个容器来存放这个令牌，但是我们并不能保证消费令牌的程序一定按照我们的生产速度来消费令牌，其实大概率是不一样的。

所以这个池子总得有个大小，不然就成一个无界的了，无界的影响就是会导致我们的限流失效。

所以这个池子就是最大能处理的突发流量。

其实我们的限流限制的是这个最大容量。

其实发放速率表示的是在同时处理b个任务的场景下，我单个任务的耗时是多少。

比如我同时处理10个任务，单个任务的耗时是0.25s，即1秒处理4个。

这个b我觉得和r是相关的。





##### 并发访问数量和访问速率

Semaphore限定的是并发访问数量，同时最多只能允许多少个线程在同时执行任务。这个根本不考虑时间，假设我们每个线程都需要执行半小时。



但是加上速率之后就不一样了，这个就对我们的处置性能给出了要求.

我们使用Semaphore只保证同时最多有多少线程在运行。

使用限流器我们给出了一个承诺，就是我们单位时间内可以完成请求的处理。

如果不能的话，还是不要使用限流器了。

比如我们承诺TPS是1000，那么1s就往令牌桶里放1000个令牌。





#### 我的设计

- 涉及到令牌的生产和获取，这个可以考虑使用生产者消费者模式

-  这个算法的目的是为了限流。
   - 我们如何定义令牌，需要一个专门的令牌对象吗，这个令牌对象需要有哪些属性，提供哪些方法呢
   - 如何生成令牌，令牌可以放在阻塞队列里
   - 如何限制令牌上限，当阻塞队列满了之后，我们变不在放
   - 如何控制速率，我们定时的往阻塞队列里加
   - 如何校验令牌，如何判断这个令牌对象是我生成的呢？

  

```java
package thread.test;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * @author Nannf
 * @date 2021/7/6 20:35
 * @description 先自己实现令牌桶算法。
 * 这个算法的目的是为了限流。
 * - 我们如何定义令牌，需要一个专门的令牌对象吗，这个令牌对象需要有哪些属性，提供哪些方法呢
 * - 如何生成令牌，令牌可以放在阻塞队列里
 * - 如何限制令牌上限，当阻塞队列满了之后，我们变不在放
 * - 如何控制速率，我们定时的往阻塞队列里加
 * - 如何校验令牌，如何判断这个令牌对象是我生成的呢？
 */
public class MyRateLimiter {
    // 最大容量
    private int burst;

    // 我们需要一个阻塞队列
    private ArrayBlockingQueue<Token> queue;

    ExecutorService executorService = Executors.newSingleThreadExecutor();

    private volatile boolean isTerminal;

    public MyRateLimiter(int burst) {
        this.burst = burst;
        queue = new ArrayBlockingQueue<>(burst);
    }


    // 获取token的方案
    public Token acquireToken() {
        // 如果获取不到，返回null
        return queue.poll();
    }

    // 生产令牌
    private void createToken() {
        executorService.execute(() -> {
            while (!isTerminal && !Thread.currentThread().isInterrupted()) {
                try {
                    queue.add(new Token());
                    // 定时往队列里加
                    TimeUnit.MILLISECONDS.sleep(1000L * burst / 60);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });

    }

    public void stop() {
        isTerminal = true;
    }


    // 令牌，但是我们发现这个令牌无法定义任何的属性和方法
    // 因为令牌只是一个逻辑上的概念，表示请求方需要获取到这个东西
    // 二是，如果令牌真的对应一个真实的对象，那么我们会产生很多朝生夕死的对象，这会增加系统的GC压力
    // 所以，关于令牌，新建对象可能不是一个好方法
    public static final class Token {

    }

}

```

```java
class MyRateLimiterTest {
    static ExecutorService executorService = Executors.newFixedThreadPool(20);

    public static void main(String[] args) {
        MyRateLimiter myRateLimiter = new MyRateLimiter(10);

        myRateLimiter.createToken();

        executorService.execute(() ->{
            while (true) {
                if (myRateLimiter.acquireToken() != null) {
                    System.out.println("i get token");
                } else {
                    System.out.println("i am not ");
                }
                try {
                    // 定时往队列里加
                    TimeUnit.MILLISECONDS.sleep(100);
                }catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }

}
```

- 我使用一个空对象Token表示令牌
- 线程通过判断获取的Token是否为null来判断是否获取到了令牌
- 使用sleep方法来完成定时的任务
- 我们不校验令牌的签名



问题就出在定时的实现上，定时的实现依赖线程的sleep方法，但是当系统很繁忙的时候，我们的线程睡醒之后，不一定能立马得到cpu的调度，这很大依赖于运气，这让我们的速率完全依赖于不确定的cpu的调度。

而且我们创建了一个空对象来表示令牌，这似乎不是一个明智的选择，因为这个在高并发的场景下，会产生数不清的对象。





#### Guava的设计

- 放弃了定时器
- 记录并动态计算下一个令牌的发放时间
- 这个可以精确到1ns一个令牌，在这个基础上理论的tps是1亿。



我在分析的时候总是想一下解决这个问题，发现毫无思路，其实这个可以由一个小问题的答案慢慢演进而来。

我们考虑这样一个场景，就是一秒一个令牌，令牌桶的大小就是一。在这个场景下我们如何实现。

```java
package thread.test;

import java.util.concurrent.TimeUnit;

/**
 * @author Nannf
 * @date 2021/7/7 14:06
 * @description
 * 简单限流器，简单的含义是
 * - 令牌的产生时间固定，为一秒一个
 * - 令牌桶大小固定，为一个
 * - 用时间表示令牌的产生时间
 * - 当线程获取令牌的时间在下一个令牌的产生之前，会让线程等待两者之间的差值，并更新令牌的产生时间
 * - 当线程获取令牌的时间在令牌的时间之后，线程会立马执行，下一个令牌的时间是获取时间+一秒
 *
 *
 */
public class SimpleRateLimiter {
    // 下一个令牌的产生时间
    private long next = System.nanoTime();
    // 令牌的生成周期，单位ns
    private long interval = 1_000_000_000;


    // 线程请求令牌的时候会传请求的时间过来
    // 方法返回的是当前线程获取到令牌需要等待的时间
    synchronized long reserve(long now) {
        // 如果下一个令牌的产生时间在申请时间之后
        if (next - now >= 0) {
            // 表示线程需要等待这两个的差值，
            // 因为这个令牌已经被当前线程获取，所以我们需要更新下一个令牌的产生时间
            long wait = next - now;
            // 下一个令牌的产生时间为当前的值加上产生周期
            next = next + interval;
            return wait;
        } else {
            // 如果令牌在第4s产生，线程到第七秒才来获取，那么线程应该无需等待，并更新令牌产生时间
            next = next + interval;
            return 0L;
        }
    }


    public void acquire() {
        long wait = reserve(System.nanoTime());

        if (wait > 0) {
            try {
                TimeUnit.NANOSECONDS.sleep(wait);
                // 业务代码
            }catch (InterruptedException e) {
                e.printStackTrace();
                Thread.currentThread().interrupt();
            }
        } else {
            // 业务代码。
        }
    }

}

```

