- 性能优化 -> 串行变并行 -> 线程入场 -> 异步编程

为了方便 异步编程 CompletableFuture 入场。

有几个问题随即产生了：

- 异步编程之前是有什么问题吗？
- CF是如何解决这些问题的
- 还有别的解决方案吗？

我们使用 CF 改写了上节的泡茶程序如下：

```java
package thread;

import java.util.concurrent.CompletableFuture;

/**
 * @author Nannf
 * @date 2021/7/1 22:19
 * @description
 */
public class CompletableFutureTest {

    public void process() {
        CompletableFuture<Void> c1 = CompletableFuture.runAsync(() -> {
            System.out.println("我开始洗水壶，预计耗时1分钟");
            sleep(1000);
            System.out.println("我洗完水壶了，开始烧水，预计需要15分钟");
            sleep(15000);
            System.out.println("我水烧开了");
        });

        CompletableFuture<String> c2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("我开始洗茶壶，预计耗时1分钟");
            sleep(1000);
            System.out.println("我洗完茶壶了，开始洗茶杯，预计需要2分钟");
            sleep(2000);
            System.out.println("我洗完茶杯了，开始拿茶叶，预计耗时1分钟");
            sleep(1000);
            System.out.println("我拿完茶叶了");
            return "白毫银针";
        });

        CompletableFuture<String> c3 = c1.thenCombine(c2,(__,tf) -> {
            System.out.println("拿到茶叶"+tf);
            System.out.println("开始泡茶");
            return "上茶"+tf;
        });
        System.out.println(c3.join());
    }

    public static void main(String[] args) {
        new CompletableFutureTest().process();
    }

    public void sleep(long mills) {
        try {
          Thread.sleep(mills);
        }catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

直观的感受是我不用把 FutureTask 提交给线程池或者自己创建线程了。能更好的写业务代码。

thenCombine是什么骚东西，好像有点东西。

```java
 try {
            if (task1.get() == 9527 && task2.get() == 9527) {
                System.out.println("所有的工作全部准备完毕，可以泡茶了！");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
```

我在上节是使用get方法的阻塞特性完成了线程之间的互相等待。



#### 初识CompletableFuture

##### 如何创建

- `CompletableFuture.runAsync(Runnable runnable)` 没有返回结果
- `CompletableFuture.supplyAsync(Supplier supplier)` 有返回结果
- `CompletableFuture runAsync(Runnable runnable, Executor executor)` 指定线程池
- `CompletableFuture supplyAsync(Supplier supplier, Executor executor)` 指定线程池

> 前两个方法没有指定线程池，使用的是一个公用的线程池，ForkJoinPool，默认的线程数量是cpu的核数，可以在jvm的设置参数中修改。
>
> 公用的线程池的问题是，如果有一个慢操作，比如磁盘或者网络I/O，会导致很多线程一起在等待
>
> **每个任务使用独立的线程池会较好**



**关于异步任务，我们需要关注的是：**

- 异步任务什么时候运行结束
- 运行的结果我们如何获取
- 如何处置异步处理时遇到的异常

CF实现了Future接口，可以使用 `get()`方法阻塞直到运行结束，并获取到执行结果。

为了方便处置这两个问题，`CompletionStage` 应运而生，CompletableFuture实现了这个接口。这就是我们上面看到的`thenCombine`等的出处



#### CompletionStage

- 完成状态

我们从做一个任务而言，任务之间，有串行的关系，我先做完这个，才能做那个。

有并行的关系，我在做A的同时，可以同时做B。

有汇聚的关系，我做一件事需要A和B配合才行。

并发会涉及到 分工、同步、互斥。

我们写程序都是来完成一个任务，在没有并发之前，无论任务之间有没有关联，都需要串行执行。

串行执行不涉及分工问题，亦不涉及互斥。分工倒是需要的。其实串行说分工其实不太恰当。因为分工会涉及到多个人，串行的可能算是拆分任务更恰当些，拆分任务，定任务的执行计划。串行相当于一个人干完所有的活。

并发相当于我现在人手不限了，如果能更快。当我新增了人手之后，如何把任务分给不同的人做，其实最理想的状态是，分完之后，每个人做各自的，老死不相往来，这样就不涉及（沟通）同步和互斥，同步更像是同步消息，或者需要同一到相同的步调，然后往下接着做。

同步之前的理解是对访问临界区资源的一种说法，就并发分工而言，这个其实更多的表达的是多线程之间的通信、等待。统一步调之意。

互斥和分工和同步不是一个维度的概念，却和分工和同步息息相关。

我们区分分工好坏的一个标准就是多线程互斥要尽可能小。之所以这样说，是因为并发引入的目的就是为了提升速度，互斥必然涉及到至少一方的等待，这显然与我们的初衷背道而驰。



CompletionStage更关注的是同步，即多线程之间的时序关系。

`c3 = c1.thenCombine(c2,() -> {})`描述的就是一个汇聚关系，c3的执行需要等待c1和c2的执行完成，这种等待所有的全部完成的汇聚关系，又称AND关系，如果等待的任务中有任何一个执行完成都可以接着往下走，就称为OR关系。

##### CompletionStage接口的类型

##### 生产型

- 用上一阶段产生的结果做为指定函数的运行参数，并产生新的结果供后续阶段使用
- 名称中包含 Apply字样



##### 消费型

- 用上一个阶段的结果做为指定函数的输入，但是不对结果造成影响
- 名称中包含Accept字样



##### 不生产也不消费

- 不依赖上一阶段的运行结果，也不对结果造成影响
- 要求上一个阶段完成（正常完成）
- 名称中包含Run字样



##### 特殊的compose

- 不依赖结果，只依赖阶段本身
- 名称中带有compose



#### CompletionStage的阶段依赖类型

##### then

- 单阶段依赖

##### combine & both

- 依赖两个阶段全部完成

##### either

- 两阶段任何一个阶段完成即可



#### CompletionStage的执行方式类型

##### whenComplete

- 无论上一个阶段是i正常执行还是异常执行都会执行，这是消费型的处置，不对结果产生影响



##### handle前缀

- 是一个生产型接口，会对结果造成影响
- 可以针对上一个阶段的正常结果和异常结果处理



##### exceptionally

- 在上一个阶段是异常时处理



#### CompletionStage的异常相关

- 我们上述的whenComplete和handle前缀的可以不管上一个阶段的正常与否都能接着执行

- 其余所有的方法都无法处置异常相关

  - 一个阶段异常了，所有依赖于这个阶段完成的阶段全部`CompletionException`
  - 一个阶段依赖两个阶段，两个阶段全都执行异常，`CompletionException`会对应两个当中的任何一个
  - 一个阶段依赖两个阶段的任何一个，但是只有一个执行异常，因为异步，我们无法判断是否会抛异常
  - **whenComplete**本身导致的异常，异常会作为异常原因抛出。

  

  

  





##### 串行关系

- thenApply
- thenAccept
- thenRun
- thenCompose



###### thenApply

```java
    public void thenApplyTest() {
        CompletableFuture<String> c = CompletableFuture.supplyAsync(
                () -> "Hello CompletionStage").
                thenApply(s -> s + " i am nannf").
                thenApply(String::toUpperCase) ;
        System.out.println(c.join());
    }
```

- `supplyAsync()`是提供一个异步线程



#### AND汇聚关系

- thenCombine
- thenAcceptBoth
- runAfterBoth



#### OR汇聚关系

- applyToEither
- acceptEither
- runAfterEither



#### 异常处理

```java
CompletionStage exceptionally(fn);
CompletionStage<R> whenComplete(consumer);
CompletionStage<R> whenCompleteAsync(consumer);
CompletionStage<R> handle(fn);
CompletionStage<R> handleAsync(fn);
```



```java

CompletableFuture<Integer> 
  f0 = CompletableFuture.
    .supplyAsync(()->(7/0))
    .thenApply(r->r*10);
System.out.println(f0.join());
```

- exceptionally 类似 try-catch 中的catch
- whenComplete 和 handle 类似 finally，when不支持返回结果，handle支持

```java
    public void exceptionTest() {
        CompletableFuture<Integer>
                f0 = CompletableFuture.supplyAsync(
                        () -> (7 / 0))
                        .thenApply(r -> r * 10).handle((s,t) -> {
                            if (t != null) {
                                System.out.println(t);
                                return 9527;
                            } else {
                                return s;
                            }
                });
        System.out.println(f0.join());
    }
```

详见https://blog.csdn.net/weixin_30247781/article/details/101502182；



#### 课后思考

> 创建采购订单的时候，需要校验一些规则，例如最大金额是和采购员级别相关的。有同学利用 CompletableFuture 实现了这个校验的功能，逻辑很简单，首先是从数据库中把相关规则查出来，然后执行规则校验。你觉得他的实现是否有问题呢？

```java

//采购订单
PurchersOrder po;
CompletableFuture<Boolean> cf = 
  CompletableFuture.supplyAsync(()->{
    //在数据库中查询规则
    return findRuleByJdbc();
  }).thenApply(r -> {
    //规则校验
    return check(po, r);
});
Boolean isOk = cf.join();
```

```tcl
答： 实际运行了一下，发现好像就语法层面而言是没有问题的。
发现了一些其他的我没在意的地方
- 数据库查询这种耗时的操作应该放到单独的线程去处理，不应该使用默认的公共线程
- 没有做异常处理
```

