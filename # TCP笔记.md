# TCP笔记

参考资料:
[TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)
[TCP 的那些事儿（下）](https://coolshell.cn/articles/11609.html)

## TCP头
|名称|长度|作用|
|---|---|---|
|SourcePort|16bit|源端口|
|DestinationPort|16bit|目的端口|
|SequenceNumber|32bit|包序列号，解决网络乱序问题|
|AcknowledgmentNumber|32bit|ACK，用于确认收到，解决丢包问题|
|offset|4bit|tcp头部大小，单位是四字节|
|Reserved|6bit|保留|
|Flag|6bit|TCP状态，依次是 URG: 紧急标示位，同urgent pointer一块使用，标示从sequence no 指示的位置偏移urgent pointer 位 为紧急内容，之后是正常内容  ACK: 表示是确认包  PSH: 表明不是用tcp缓存，尽快把包给应用层  RST: tcp 复位标识，用于异常终止连接，或者检测半打开  SYN:  Synchronize sequence numbers 同步序号，用于tcp建立  FIN:  No more data from sender 表示没有数据需要发送|
|window|16bit||
|CheckSum|16bit|校验和，tcp头和tcp数据段校验和|
|urgent pointer|16bit|紧急数据偏移量
|TCP Options|多字节|不常用，结合具体使用选项|
|Padding|若干|保证TCP头的四字节对齐，offset单位是四字节|

## TCP状态
### 连接 三次握手
三次握手主要是为了初始化双方sequence Number。因为是两端的seq并且要ACK所以是三次  
A --*SYN sqe=x*--> B  
A <--*ACK sqe=y,ACK=x+1*-- B  
A --*ACK=y+1*--> B  
* SYN超时
* SYN Flood
* ISN初始化
* 
### 断开 四次挥手
A --*FIN seq=x*--> B
A <--*ACK=x+1*-- B
A <--*FIN sqe=y*-- B
A --*ACK sqe=y+1*-->B

### 传输 重传机制
TCP的Seq和ACK是以字节数为单位的，ACK是确认最大连续收到的字节数。一旦收到ACK，就意味着在ACK之前的数据全部接收到了。ACK无法跳着确认，必须是连续数据中间不能有空缺。比如发出1、2、3、4、5，收到1、2、4、5；3丢失，那么收到的ACK是2
#### 超时重传
#### 快速重传
#### 


### 拥塞控制
慢启动 拥塞避免 拥塞发生 快速恢复