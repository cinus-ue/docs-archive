# Netty源码结构

## 包结构
```
netty-transport      核心的类(或接口): Bootstrap、Channel、ChannelHandler和EventLoop
netty-buffer         Netty的Buffer实现，API设计和性能更符合需求
netty-codec          编解码功能(常用的编解码类)
netty-handler        ChannelHandler的实现类，包含logging、ssl、trafic等
netty-resover        解析器，包含address、host、name、dns等数据的解析
netty-common         通用的工具类、并发包
netty-transport-native-epoll    Linux平台JNI实现
netty-transport-native-kqueue   MacOS/BSD平台JNI实现
```
Netty provides the following platform specific JNI transports:  

Linux (since 4.0.16)  
MacOS/BSD (since 4.1.11)  
These JNI transports add features specific to a particular platform, generate less garbage, and generally improve performance when compared to the NIO based transport.  

## Bootstrap
```
Bootstrap            客户端
ServerBootstrap      服务端

分类	           网络功能              EventLoopGroup数量    
Bootstrap	      连接到远程主机和端口    1
ServerBootstrap   绑定本地端口           2   
```

## Channel
```
EventLoopGroup   线程池，管理eventLoop的生命周期
ChannelPipeline  构造channelhandler的链表(ChannelPipeline就是ChannelHandler链的容器)
ChannelHandler   Bootstrapping阶段被添加ChannelPipeline中，被添加的顺序将决定处理数据的顺序
ChanneHandlerContext  包含着ChannelHandler中的上下文信息，用来管理它所关联的ChannelHandler和在同一个ChannelPipeline中ChannelHandler的交互
ChannelInboundHandlerAdapter    抽象基类
ChannelOutboundHandlerAdapter   抽象基类
SimpleChannelInboundHandler     抽象类，处理指定类型的消息，需实现channelRead0
NioSocketChannel           异步的客户端TCP Socket连接
NioServerSocketChannel     异步的服务器端TCP Socket连接
NioDatagramChannel      异步的UDP连接
NioSctpChannel          异步的客户端Sctp连接
NioSctpServerChannel      异步的Sctp服务器端连接，这些通道涵盖了UDP和TCP网络IO以及文件IO
ReflectiveChannelFactory      Channel工厂
```
Netty发送消息可以采用两种方式：直接写消息给Channel或者写入ChannelHandlerContext对象。这两者主要的区别是，前一种方法会导致消息从ChannelPipeline的尾部开始，而后者导致消息从ChannelPipeline下一个处理器开始。


## Codec
所有的编码器/解码器适配器类都实现自ChannelInboundHandler或ChannelOutboundHandler  
 ```
ByteToMessageDecoder   抽象基类
MessageToByteEncoder   抽象基类
 ```

## Buffer
Buffer API主要包括:  
```
ByteBuf  
ByteBufHolder
```  
Netty根据reference-counting(引用计数)来确定何时可以释放ByteBuf或ByteBufHolder和其他相关资源，从而可以利用池和其他技巧来提高性能和降低内存的消耗。  
ByteBuf API 的优点:  
* 允许用户自定义缓冲区类型扩展
* 通过内置的复合缓冲区类型实现透明的零拷贝
* 容量可按需增长
* 读写这两种模式之间不需要调用类似于JDK的ByteBuffer的flip()方法进行切换
* 读和写使用不同的索引
* 支持方法的链式调用
* 支持引用计数
* 支持池化


