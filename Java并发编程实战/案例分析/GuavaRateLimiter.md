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

  

