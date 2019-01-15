# SRT

基于QUIC协议的一个改进的实现

## 零散杂记（候选特性）

+ 可以使用protobuffer Varint编码方式压缩消息头体积
+ header和Body的加密不一样，可能有中继节点，body用server和client协商密钥，header使用单次传递消息两端协商的密钥
+ header里面需要携带server的ip port connectionID或者用两种类型消息包标记转发/直接连接 用于后期组网扩展
+ connectionID uint64随机产生 降低不正当使用时的冲突问题  最好通过 sitID+random方式保证唯一

## 层次结构

```ditaa{ args=["-E"] code_block=true cmd=false}
+-------------+
|  Session    |
+-------------+
|  Streams    |
+-------------+
| Connection  |
+-------------+
|io_interface |
+-------------+
```

io_interface : socket接口的封装，使用boost::asio::udp::socket。数据收发接口  
connection : 连接抽象，实现可靠连接。拥塞控制(GC)、流量控制(基于窗口)、Pacing发送、NACK、Retransmission  
Stream : 轻量级的读写流，相互之间不会阻塞 可以理解为一种弹性的“消息”
Session :app交互接口，可以在此实现多connection聚合

## Packet类型

### long header packet

```ditaa{ args=["-E"] code_block=true cmd=false}
+-+--------+------+------+--------------------------+--------------------------+-----------+
|1|Type(7b)|DL(4b)|SL(4b)|DstConnectionID(4~8 Bytes)|SrcConnectionID(4~8 Bytes)|Payload (*)|
+-+--------+------+------+--------------------------+--------------------------+-----------+
```

#### Hello

```ditaa{ args=["-E"] code_block=true cmd=false}
+----+------+------+--------------------------+--------------------------+-------------------------+
|0x80|DL(4b)|SL(4b)|DstConnectionID(4~8 Bytes)|SrcConnectionID(4~8 Bytes)|CRYPTO Frame(*) (PADDING)|
+----+------+------+--------------------------+--------------------------+-------------------------+
```

#### Retry

```ditaa{ args=["-E"] code_block=true cmd=false}
+----+------+------+--------------------------+--------------------------+-------------------------------------------+
|0x81|DL(4b)|SL(4b)|DstConnectionID(4~8 Bytes)|SrcConnectionID(4~8 Bytes)| TokenLen(Varint) Token(TokenLen) (PADDING)|
+----+------+------+--------------------------+--------------------------+-------------------------------------------+
```

#### HandShake

```ditaa{ args=["-E"] code_block=true cmd=false}
+----+------+------+--------------------------+--------------------------+--------------------------+
|0x82|DL(4b)|DL(4b)|DstConnectionID(4~8 Bytes)|SrcConnectionID(4~8 Bytes)| CRYPTO Frame(*) (PADDING)|
+----+------+------+--------------------------+--------------------------+--------------------------+
```

HandShake只能包含CRYPTO，PADDING，CONNECTION_CLOSE Frame，其他的Frame端点都视为Connection Error

### short header packet

```ditaa{ args=["-E"] code_block=true cmd=false}
+-+-----+------+--------------------------+---------------------------+-----------+
|0|R(3b)|DL(4b)|DstConnectionID(4~8 Bytes)|PacketNumber(Varint max=11)|Payload (*)|
+-+-----+------+--------------------------+---------------------------+-----------+
```

### 帧类型

|类型|名称|
|---|---|
|0x00|PADDING|
|0x01|PING|
|0x02~0x03|ACK|
|0x04|RESET|
|0x05|STOP_SENDING|
|0x06|CRYPTO|
|0x07|NEW_TOKEN|
|0x08~0x0F|PAYLOAD|
|0x10|MAX_DATA|
|0x14|DATA_BLOCKED|
|0x18|NEW_CONNECTION_ID|
|0x19|RETIRE_CONNECTION_ID|
|0x1A|PATH_CHALLENGE|
|0x1B|PATH_RESPONSE|
|0x1C~0x1D|CONNECTION_CLOSE|

## 通信过程

```sequence {theme="simple"}
Note Over Server: listen
Client->Server: Hello
Server-->Client: Retry (token)
Client-->Server: Hello (token)
Note Over Server:Accepting Connection
Server->Client:HandShake
Note Right Of Client:connected

Client->Server:C0:STREAM(data)
Note Left Of Server:Connected
Note Over Server:App notify
Server->Client:S0:STREAM(data) ACK-C0
Client->Server:C1:ACK-S0
Note Right Of Client:End with ack only
Note Left Of Server: End with ack only
Note Left Of Server: If C1 Loss, Tail Loss Probe
Server-->Client:S1:STREAM(data) ACK-C0 [TLP]
Client-->Server:C2:ACK-S1-S0
Note Left Of Server: C2 Loss
Server-->Client:S2:STREAM(data) ACK-C0 [TLP]
Client-->Server:C3:ACK-S2-S0
Note Left Of Server:C3 Loss, twice TLP fail,\n RTO Send dual packets
Server-->Client:S3:STREAM(data) ACK-C0 [RTO]
Server-->Client:S4:STREAM(data) ACK-C0 [RTO]
Client-->Server:C4:ACK-S4-S0
Note Left Of Server: If C4 Loss, RTO Double


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

### 握手流程

+ Client向Server发送Hello Packet，源connection ID为client connection ID 目标connection ID 为0；携带D-H密钥交换参数。
+ Server Acceptor接收到Init Packet进行源connection ID匹配，如果匹配上则表明这是一个重传的Init Packet 派送给对应的connection处理。没有匹配上则表明是一个新建连接Init Packet，新建一个连接派送处理。
+ 新建connection处理第一个Init Packet，发出handshake响应包，连接建立。

### Ack

即使在接收到的packet之前存在间隙，端点绝不能发送ACK Frame来确认只包含ACK Frame和PADDING Frame的数据包。  
必须在确认其他分组时确认只包含ACK Frame和PADDING Frame的packet。  
为了保证接收到需要确认的packet时每个RTT内至少发送一次确认，接收方的延迟确认定时器不应大于min(RTT,kMaxAckDelay) kMaxAckDelay默认取25ms。  
一旦ACK Frame被对端确认（ACK Tracked）那么它所确认的packet不应再次ACK。  
为了限制接收器状态或ACK Frame的大小，接收器可以限制发送的ACK block数量（这可能导致不必要的重传）。  
接收方应该优先重复确认最新接收的数据包而不是之前接收的数据包，尽快触发对端标记packet丢失。

ACKOnly的数据包不受拥塞控制限制且不计入 bytes_in_flight

ACK Frame可以包含最近几次（默认3次）发出但未被Tracked的ACK信息

### 重传

#### 基于ACK的重传

+ PacketNumber阈值
跟踪到Send packet被对端确认，在 最大被确认packet_number - 3 之前发送的packet被标记为丢失

+ 时间阈值
跟踪到Send packet被对端确认，在 最大被确认packet_number 之前发送的packet，如果超过时间阈值被标记为丢失。  
时间阈值 = kTimeThreshold * max(SRTT,last_RTT)  
系数kTimeThreshold默认取9/8（可调整）减小会增大虚假重传的概率 增大会增加丢失检测延迟

#### 基于超时的重传

+ 尾丢失探测（Tail Loss Probe）
  探测超时（Probe Timeout，PTO） PTO = min(max(1.5*SRTT+MaxAckDelay, kMinTLPTimeout),RTO)  
  TLP至少为1.5倍的SRTT是（TCP TLP定义），为了保证ACK过期。  
  为了减少延迟，在RTO定时器之前允许TLP定时器触发两次，当TLP定时器第一次到期，发送TLP分组并重新开始TLP定时器计时，当TLP定时器第二次到期，发送第二次TLP分组并开启RTO定时器计时。  
  TLP数据包应尽可能发送新的数据，如果没有新数据可用或流控限制则可以重传未确认的数据，从而尽可能缩短恢复时间。  
  TLP定时器超时发送探测数据包是在确认数据包丢失之前，因此不应将未确认的数据包标记为丢失状态。
  TLP重传不受拥塞控制限制，但需要计入bytes_in_flight

+ 重传超时时间（Retransmission Timeout，RTO）
  当发送最后一个TLP分组时，RTO定时器启动，如果RTO定时器超时，发送方发送两个packet，以唤醒对端进行ACK，并重新启动RTO定时器  
  最后一个TLP发送时 RTO = max(SRTT + 4*RTTVAR + MaxAckDelay, kMinRTOTimeout)  
  每当RTO定时器超时，则RTO翻倍  
  RTO重传不受拥塞控制限制，而是通过RTO惩罚性翻倍来保证收敛

### 关闭connection流程

发送端数据全部发送完毕，发送CONNECTION_CLOSE主动关闭当前连接，发送后等待Ack

## 安全性
相关参数定义：

+ initial_salt  20bytes随机常量


### Packet加密保护
long header数据包：

```ditaa{ args=["-E"] code_block=true cmd=false}
+-+--------+------+------+--------------------------+--------------------------+-----------+
|1|Type(7b)|DL(4b)|SL(4b)|DstConnectionID(4~8 Bytes)|SrcConnectionID(4~8 Bytes)|Payload (*)|
+-+--------+------+------+--------------------------+--------------------------+-----------+
```

short header数据包：

```ditaa{ args=["-E"] code_block=true cmd=false}
+-+-----+------+--------------------------+---------------------------+-----------+
|0|R(3b)|DL(4b)|DstConnectionID(4~8 Bytes)|PacketNumber(Varint max=11)|Payload (*)|
+-+-----+------+--------------------------+---------------------------+-----------+
```

#### Header保护

对long header packet的第一二字节 和 short header packet的第一字节使用 initial_salt[0]与之异或进行隐藏
对long header packet的 DstID，SrcID字段，short header packet 的 DstID字段 使用采样+init_salt 作为输入生成的密钥 进行AEAD_SHA_128_GCM 加密

### 放大攻击

避免服务端被放大攻击利用，严格限制对未验证的客户端的响应不会超过客户端请求数据大小。要求连接建立期间，客户端发起的建立连接请求数据包，必须填充到默认数据包最大长度。否则服务器可以不响应。

### 中间人攻击

采用DH密钥协商机制容易收到中间人攻击  
中间人攻击是指：A和B进行密钥协商，消息被中间人C拦截 A向B发起的密钥协商信息被C拦截，并响应使得AC连接，同时C向B发起协商信息连接到B，这样A发送给B的信息会发送给C，C可以拦截并篡改后再发送给B。  
要解决中间人攻击问题需要引入证书认证，Server和Client之间进行认证，认证信息需要结合域名、IP等信息进行。暂不考虑在传输层进行认证，由应用(或中间层)按照需要进行认证过程。

## 客户端验证

避免伪装客户端大量非法建立连接请求，服务端可基于token对客户端进行验证，客户端发送Init Packet要求建立连接，服务端返回Retry Packet并携带一个token，要求客户端基于一定的规则，对token进行处理后，再次请求连接并携带token（QUIC中是要求token原样返回，这里要求token进行一定规则的处理，如携带token的Hash256签名）

## 地址欺骗

侦测到连接地址迁移（接收到来自其他源地址的高PN编号的packet），必须对迁移地址进行验证，发送PATH_CHALLENGE帧，在接收到正确的PATH_RESPONSE不会向该地址发送新的数据（防止放大攻击），验证失败继续使用原来的源地址。在此期间来自其他地址的更高PN编号的packet将覆盖当前验证地址。


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

1. 数据包到达socket
2. io_interface异步接收事件执行，数据填入connection接收队列尾部，通知connection数据到达
3. connection从接收队列头部解包处理frame，将stream frame的数据填入stream接收缓冲区

## 缓冲区设计

1. stream接收缓冲区，内存片队列，单片8k，带有数据区间记录，只能读取头部连续区间数据。
2. stream发送缓存区，内存片队列，单片8k，带有数据区间记录，只有头部连续区间可擦除。

## 线程安全

采用流水线设计来分解处理各阶段任务；每一级流水线对应一个boost::asio::io_services实例，每个io_service绑定到一个线程或线程池处理。

1. io_interface 
   所有的io_interface投递到同一个io_service中，主要处理两个任务：
   + 异步接收Packet，解出dstConnectionID，投递到对应注册的connection上
   + 异步发送connection打包的packet
2. packet预处理（暂未使用）
   预处理阶段设计用来对packet进行加解密
3. connection
   处理Frame生成、打包、解包工作，每一个connenction通过一个boost::asio::io_service::strand保证处理过程的串行化。



## receive/sent process

任务：

1. 处理读取事件，接收来自io的packet。
2. 维护connection和IOInterface是的映射关系，IOInterface可复用
3. 解出packet的connection id投递到对应的指定connection的消息队列
4. 将cinnection的packet通过IOInterface发出，后期加入路由选择和负载均衡处理

key point:

1. 只关心connection ID 不关心src ip/port等连接特性。连接迁移。

## 链路信息统计

+ RTT
+ Lost
+ Latency
+ 最大延迟波动 ==> 最大包乱序