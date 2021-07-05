Balking 警戒不前。

这个和等待唤醒模式都有一个相同点

在等待一个条件。

当等待的条件不满足的时候，Balking会直接退出，等待唤醒会一直等待。



Balking 模式使用synchronized关键字即可实现。



#### 课后思考

> 下面的示例代码中，init() 方法的本意是：仅需计算一次 count 的值，采用了 Balking 模式的 volatile 实现方式，你觉得这个实现是否有问题呢？

```java

class Test{
  volatile boolean inited = false;
  int count = 0;
  void init(){
    if(inited){
      return;
    }
    inited = true;
    //计算count的值
    count = calc();
  }
}  
```



```tcl
答： 多线程同时调用init方法时，会同时进入到计算逻辑。
因为判断和赋值不是原子操作。

```

