# SRT

## 零散杂记（候选特性）

+ 可以使用protobuffer Varint编码方式压缩消息头体积
+ header和Body的加密不一样，可能有中继节点，body用server和client协商密钥，header使用单次传递消息两端协商的密钥
+ header里面需要携带server的ip port connectionID或者用两种类型消息包标记转发/直接连接 用于后期组网扩展
+ connectionID uint64随机产生 降低不正当使用时的冲突问题  最好通过 sitID+random方式保证唯一

## 层次结构

```ditaa{ args=["-E"] code_block=true cmd=false}
+-------------+
|  Stream     |
+-------------+
|  Session    |
+-------------+
| Connection  |
+-------------+
|io_interface |
+-------------+
```

io_interface : sockket接口的封装，使用boost::asio::udp::socket。数据收发接口  
connection : 连接抽象，实现可靠连接。拥塞控制(GC)、流量控制(基于窗口)、Pacing发送、NACK、Retransmission  
Session : 回话管理，管理connection的多路复用（一个connection多个stream/多个connection）
Stream : 应用程序读写接口

## 通信过程

```sequence {theme="simple"}
Note Over Server: listen
Client->Server: Init Packet(hello)
Note Over Server:Accept Connection
Server->Client:HandShake Packet,MAX_DATA
Note Right Of Client:connected

Client->Server:STREAMS_BLOCKED
Note Over Server:Accept  stream
Server->Client:MAX_STREAMS,MAX_STREAM_DATA
Note Right Of Client:new stream  S1 created
Client->Server:STREAM
Server-->Client:STREAM
Note Right Of Client:send some data ...
Note Left Of Server: receive some data
Client->Server:DATA_BLOCKED/STREAM_DATA_BLOCKED
Note Right Of Client:connection/stream blocked
Server->Client:MAX_DATA/MAX_STREAM_DATA
Note Right Of Client:stream continue
Client->Server:STREAM

Note Left Of Server:maybe path validation
Server->Client:PATH_CHALLENGE
Client->Server:PATH_RESPONSE
Note Right Of Client: path response

Client->Server:STREAM(FIN)[1]
Server-->Client:STREAM(FIN)[2]
Server->Client:ACK([1])
Note Right Of Client:close stream S1
Client->Server:ACK([2])
Note Left Of Server:close stream  S1

Server-->Client:CONNECTION_CLOSE
Client-->Server:CONNECTION_CLOSE

```

### 关闭stream逻辑

发送端stream数据（除最后一个STREAM Frame）被确认接收完毕，发送最后一个STREAM frame（如果数据全部确认接收STREAM为空）标记为FIN状态，接收端接收到FIN状态的STREAM，则数据传输完毕，关闭本端stream，当发送端收到携带FIN STREAM frame被确认的ACK后关闭发送端stream。

### 关闭connection逻辑

发送端数据全部发送完毕，发送CONNECTION_CLOSE主动关闭当前连接，发送后等待Ack

## Send

**数据发送过程：**

1. application向stream wBuffer 写入数据
2. stream向Session投递trySend事件  
3. session通知connection发送事件  有多个connection时需要决定向哪个connection投递。（一般只有一个connection）
4. connection根据状态决定发送时机：立即发送、延迟发送（小数据量delay）、状态改变后发送（blocking）；满足发送条件的数据通过pacing的方式发出

**尝试发送事件:**

1. 外部数据写入发送缓冲区 maybeSend
2. control frame加入 maybeSend
3. blocking状态改变

**启动定时器：**

1. only ack 延迟 25ms
2. stream数据量从零到小于阈值 延迟发送200ms

发送事件：

1. 延迟发送timer超时 sendPacket
2. 延迟ack timer超时 sendPacket
3. ack阈值（第二个full-size packet到达） sendPacket
4. 待发送数据到达阈值 sendPacket

## Receive

**数据接收过程：**

1. 数据包到达socket
2. io_interface异步接收事件执行，数据填入connection接收队列尾部，通知connection数据到达
3. connection从接收队列头部解包处理frame，将stream frame的数据填入stream接收缓冲区

## 缓冲区设计

1. stream接收缓冲区，内存片队列，单片8k，带有数据区间记录，只能读取头部连续区间数据。
2. stream发送缓存区，内存片队列，单片8k，带有数据区间记录，只有头部连续区间可擦除。
3. connection接收缓存区，内存片队列，单片maxPackSize，每次接收一个数据包独占一片，接收、解包不拷贝数据，数据包解析完毕，stream数据写入stream接收缓存后擦除。
4. connection发送采用发送时打包的方式，单片maxPackSize

connection缓存大小设计，按照最大包分片，存在一定的内存利用浪费，connection缓冲区容量根据 带宽/数据包处理时间max 来设置

## 线程安全

设计进行横向的线程划分：

+ 所有io_interface数据包接收操作作为一个集合，投递到同一个io_service上这样可以通过一个独立的线程或线程池来将数据尽快地从socket缓冲区读取到connection中
+ 所有connection的解包、timer等操作被投递到一个io_service上

stream缓冲区单片通过mutex互斥

connection缓冲区单片通过C++ automic进行状态标识

## Ack


## receive/sent process

任务：

1. 处理读取事件，接收来自io的packet。
2. 维护connection和IOInterface是的映射关系，IOInterface可复用
3. 解出packet的connection id投递到对应的指定connection的消息队列
4. 将cinnection的packet通过IOInterface发出，后期加入路由选择和负载均衡处理

key point:

1. 只关心connection ID 不关心src ip/port等连接特性。连接迁移。