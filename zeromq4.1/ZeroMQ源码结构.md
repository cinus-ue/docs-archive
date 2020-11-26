# ZeroMQ源码结构

## 全局状态
```
ctx.cpp        管理全局状态
```

## 并发模型(actor模式)
线程间的通信是通过线程间传递异步消息（内部命令）来实现。

## 线程模型
ZMQ内部使用两种不同类型的线程（拥有邮箱的对象）：I/O线程(io_thread_t)和socket(socket_base_t)。
```
io_thread.cpp       I/O线程
socket_base.cpp     特殊的socket线程（根据类型创建），异步收发消息
```

## ZMQ SOCKET实现
```
pair.cpp        独立对模式(用来精确的连接另一个对端)
push.cpp        推送（单向）
pull.cpp        拉取（单向）
router.cpp      请求/响应模式的扩展（兼容的对端套接字ZMQ_DEALER, ZMQ_REQ, ZMQ_ROUTER）
dealer.cpp      请求/响应模式的扩展
req.cpp         请求（继承router_t）
rep.cpp         响应（继承router_t）
stream.cpp
xpub.cpp
xsub.cpp
pub.cpp         发布（继承xpub_t）
sub.cpp         订阅（继承xsub_t）
```


## 命令
```
object.cpp               命令发送与处理（所有继承object_t的子类都具备该类的功能）
command.hpp              内部命令
io_thread.cpp            监听句柄的读、写、异常状态（继承自object_t）
mailbox.cpp              邮箱
signaler.cpp             信号通知(mailbox中使用)
mutex.hpp                同步锁(mailbox中使用)
```

## 消息
```
stream_engine.cpp        鉴权，数据打包、解包
session_base.cpp         管理zmq_socket的连接和通信，与engine进行消息交换
msg.cpp                  消息
i_decoder.hpp            接口定义
i_encoder.hpp            接口定义
decoder.hpp              解码器基类
encoder.hpp              编码器基类
v1_decoder.cpp           V1 解码器
v1_encoder.cpp           V1 编码器
v2_decoder.cpp           V2 解码器
v2_encoder.cpp           V2 编码器
v2_protocol.hpp          ZMTP/2.0 传输协议
```

## 连接器与监听器
```
tcp_connecter.cpp        zmq_socket的连接器（tcp协议使用）
tcp_listener.cpp         zmq_socket的监听器（tcp协议使用）
ipc_listener.hpp         ipc协议使用
ipc_connecter.cpp        ipc协议使用
tipc_connecter.cpp       tipc协议使用
tipc_listener.cpp        tipc协议使用
```

## 安全销毁机制
对象树模型产生的主要目的是为了实现一致性的销毁机制（对象销毁）。  
向其所有子对象发送销毁请求命令，当收到所有孩子对销毁请求的确认时对象才会真正被销毁。  
与销毁有关的有三种命令：父对象要求子对象销毁的term命令，子对象向父对象的销毁确认term_ack命令，子对象想销毁自身时向父对象发送的term_req 命令。
```
own.cpp                   对象树
```


## 回收线程
```
reaper.cpp                负责回收socket
```