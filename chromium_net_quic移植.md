# chromium quic 源码分析

## quic_connect

connect实现

## quic_stream

stream实现

## quic_stream_send_buffer

stream发送缓冲区，队列，按照发送的包切片。ack之后从队头移除，发送的放到队尾

## quic_stream_sequence_buffer

stream接受缓冲区，环形流式缓冲区，按照stream offset写入数据，可以从头部读取没有gap的接收数据

## quic_session

将stream和connect绑定

```C++
  using StaticStreamMap = QuicSmallMap<QuicStreamId, QuicStream*, 2>;

  using DynamicStreamMap =
      QuicSmallMap<QuicStreamId, std::unique_ptr<QuicStream>, 10>;

  using ClosedStreams = std::vector<std::unique_ptr<QuicStream>>;

  using ZombieStreamMap =
      QuicSmallMap<QuicStreamId, std::unique_ptr<QuicStream>, 10>;
```

三个map保存了streamId和对应的Stream指针  

public继承QuicConnectionVisitorInterface通过connection_->set_visitor(this)注入QuicConnect中  