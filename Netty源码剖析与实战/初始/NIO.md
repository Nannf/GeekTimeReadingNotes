#### Buffer

- 内存
- 可以存放数据，可以写，可以读，读写的目标都是channel



```java
public class BufferTest {

    public static void main(String[] args) throws Exception{
        RandomAccessFile accessFile = new RandomAccessFile("E:\\github\\ZiMu\\关于未来\\恐惧与挣扎.md","rw");
        FileChannel fileChannel = accessFile.getChannel();
        // 
        ByteBuffer byteBuffer = ByteBuffer.allocate(460);
        int byteRead = fileChannel.read(byteBuffer);

        while (byteRead != -1) {
            byteBuffer.flip();
            while (byteBuffer.hasRemaining()) {
                System.out.println((char) byteBuffer.get());
            }
            byteBuffer.clear();
            byteRead = fileChannel.read(byteBuffer);
        }
        accessFile.close();
    }
}
```

##### position

- 写时表示下一个能写的位置
- 读时会初始化成0



##### limit

- 写时=capacity
- 读时表示最多能读多少数据=切换到读之前的写的时候的position





#### Selector

- 可以对接多个channel，channel注册在Selector上
- 可以直到channel中是否做好了读写准备
- 可以单线程管理多个注册在自己上的channel
- 可以单线程管理所有的通道，这样可以减少线程上下文切换的开销，但是多核系统现在走入寻常百姓家，这种等待io的用多核大势所趋



##### 如何开启

`Selector selector = Selector.open();`



##### 如何注册通道

```java
// 虽然不知道原因，但是channel和Selector不能是阻塞的，FileChannel就不能注册到Selector上
// 这是因为FileChannel实现了一个接口，不支持非阻塞模式，
// 接口的注释是说文件IO
channel.configureBlocking(false);
SelectionKey key = channel.register(selector,SelectionKey.OP_READ);
```

##### 关于SelectionKey

- 一共有四种类型
  - OP_READ  1 channel中有数据可读
  - OP_WRITE 4 channel可以写数据
  - OP_CONNECT 8  channel成功连接到一个服务器
  - OP_ACCEPT 16 服务端准备好接收连接
- 这个是说这个channel注册到这个selector上是告诉这个selector，我对什么类型的事件感兴趣



#### 总结

- Buffer - 所有的数据都经由它
- channel - 管道 有监控连接的ServerSocketChannel 有传送数据的SocketChannel,首先得开启ServerSocketChannel,然后监听连接请求，注册感兴趣的事件，此时需要用SocketChannel来传送数据
- Selector - 这个的出现是高并发的关键，准确的说是和非阻塞io同时出现，阻塞io的问题是读写都会阻塞，连接数高的时候，很多的连接都卡在了io上，所以非阻塞io应运而生。读写数据都要经由channel，当然FileChannel也有阻塞模式的，就网络连接而言，都是非阻塞的，此时管理多连接的io模型，Reactor应运而生。



