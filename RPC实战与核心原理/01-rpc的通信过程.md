#### 概览

**RPC**：Remote Procedure Call 远程过程调用；

1. 远程即需要跨机器，需要网络传输



RPC和http请求的区别：

1. RPC是基于TCP协议，Http请求时基于http协议，http也是基于tcp协议
2. RPC的开发要求比http高，比如客户端和服务端的技术栈必须一样，要求更严苛；http什么都可以，java可以调用python。
3. RPC比http更快，具体的原因，大致说来是http为了通用性牺牲了性能，RPC是特殊场景下的性能优化。





#### 一次RPC通讯的过程

1. 网络，跨机器，需要网络传输，网络传输涉及到可靠性使用TCP，UDP一般用于内网局域网，网络环境比较稳定，可以容许丢失的场景，比如视频流，公司内部的oa这些，但是大部分系统像金融或者其他的场景，对数据的可靠性传输的需求更为迫切，所以使用的是TCP协议。
2. TCP传输的是二进制流，但是我们调用的时候传输的是对象，所以需要把对象转换成二进制流，这一步我们称为序列化。
3. TCP传输层，下一层是IP，网络层，如果我们要进行网络通讯，通讯的格式，即协议需要确定好，不然接收方，不知道如何处置接受到的二进制流，所以需要一个协议，这个我理解就是类似http协议类似的东西。
4. 根据定好的协议，接收方就可以把接收到的二进制流进行解析，知道了发送方的需求，这个解析的过程，我们称之为反序列化。
5. 接收方处置完毕之后，会返回结果给发送方，发送的内容当然也是按照协议规定的来，也需要经过序列化，经由TCP协议发送给发送方，然后发送方进行反序列化，解析。



#### 总结

- http协议通讯的性能强化版

  
