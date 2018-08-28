# SRT

## send receive
尝试发送:
1. 外部数据写入发送缓冲区 maybeSend
2. control frame加入 maybeSend
3. 接收包处理完成 maybeSend

设置延迟发送定时器：
1. 延迟ack 25ms
2. stream frame数据量小于阈值 延迟发送200ms

立即发送：
1. 延迟发送timer超时 sendPacket
2. 延迟ack timer超时 sendPacket
3. ack阈值（第二个full-size packet到达） sendPacket
4. 待发送数据到达阈值 sendPacket

## Ack

##SRTIOInterface
1. io接口抽象，提供 send 和 async_receive等方法

## receive/sent process
任务：
1. 处理读取事件，接收来自io的packet。
2. 维护connection和IOInterface是的映射关系，IOInterface可复用
3. 解出packet的connection id投递到对应的指定connection的消息队列
4. 将cinnection的packet通过IOInterface发出，后期加入路由选择和负载均衡处理
key point:
1. 只关心connection ID 不关心src ip/port等连接特性。连接迁移。