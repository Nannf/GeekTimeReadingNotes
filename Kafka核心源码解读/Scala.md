```scala
def sizeInBytes(segments: Iterable[LogSegment]): Long =
    segments.map(_.size.toLong).sum
```

- 这是一个方法
- 方法名是 `sizeInBytes`
- 方法的参数是一组 LogSegment对象
- 方法的返回值是一个Long
- 这似乎是一个函数式语言，用到了map方法，类似于java的stream

```java
List<String> list = new ArrayList<>();
        list.add("nannf");
        list.add("we");
        list.add("qer");

        long length = list.stream().mapToLong(String::length).sum();
        System.out.println(length);
```



```scala
val firstOffset: Option[Long] = ......

def numMessages: Long = {
    firstOffset match {
      case Some(firstOffsetVal) if (firstOffsetVal >= 0 && lastOffset >= 0) => (lastOffset - firstOffsetVal + 1)
      case _ => 0
    }
  }
```

- `firstOffset` 是一个 Option<Long>类型的变量
- `numMessages` 是一个long型的变量
- `numMessages`的值是用表达式算出来的
- match 和 case 类似java中的switch，我们可以把这段改成如下的java代码

```java
Option<Long> firstOffset = Option.of();
Long numMessages;
switch (firstOffset) {
    case :
        return;
}
```

