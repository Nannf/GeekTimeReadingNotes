本质； 网络应用程序框架

实现： 异步、事件驱动-> 高性能

特性： 高性能、可维护、快速快发

用途：服务器和客户端



![image-20210725101943599](https://github.com/Nannf/PicBed/main/blog_files/img/PicGo-Github-PicBed/image-20210725101943599.png)





#### 概述

已经学完了netty的源码，对netty做了什么有了一个大致的了解。

netty就是对jdknio包的一个封装，用来实现高性能的网络传输。

网络传输也是一种io，io不可避免的涉及到io模型。

有bio，nio，aio，bio是阻塞io，高并发下表现不好。aio经netty的实验在windows上表现较好，但是linux表现平平，现在都已废弃。

所以netty只实现了nio。

如果要传输，就需要ServerSocketChannel和SocketChannel以及Buffer和Selector。

netty解决了jdknio包的一些bug，然后做了一些易用化处理。

netty的一个主线流程就是

- 启动服务
- 创建连接
- 接收数据
- 业务处理
- 发送数据
- 断开连接
- 关闭服务

还有就是关于网络传输中的tcp的粘包半包问题的解决。

让我动了去学网络协议的念头。

然后还去并发编程网看了nio相关的介绍。关于nio的教程就是理清楚主线，这个值得我借鉴。比如我看到FileChannel不能开启非阻塞模式之后，就深度优先了，导致了迷失在细节中。

网络协议我决定也是这样一个方法，抓住主线，比如一次网络访问，到底经历了什么，慢慢的完善细节。

主要实际的环境，跨网络的情况太多，网络不应该成为我的黑盒。

